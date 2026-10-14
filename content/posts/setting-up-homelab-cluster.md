+++
title = 'Bootstrapping Canonical Kubernetes on a Raspberry Pi Cluster'
date = 2026-10-14T09:00:00+03:00
draft = false
tags = ['kubernetes', 'homelab', 'canonical-kubernetes', 'raspberry-pi']
+++

I run a 3-node Raspberry Pi cluster at home — `sentinel`, `watchtower`, `cyclone`. It's been through a few lives: it started on k3s, went through a stint of Kubernetes the Hard Way to actually understand what k3s was hiding, got fully reflashed to Ubuntu Server 24.04, and most recently moved onto Canonical's own production distribution — the `k8s` snap — installed and operated directly, no Juju in this pass.

Here's how that bootstrap actually went, including the two decisions that get locked in permanently and the mistake that changed my topology.

## Two decisions before touching anything

**Networking: stay on the LAN for now.** The long-term goal is to eventually add a cloud VM as a node, which argues for binding the control plane to something reachable from outside the house — Tailscale was the obvious candidate. I decided against it for this pass: binding to Tailscale before there's an actual cloud node to justify it just adds a moving part, and risks learning "Kubernetes via Tailscale" instead of Kubernetes itself. Bootstrapped on the plain home LAN. Cloudflare Tunnel already handles external exposure for the one service I run (more on that in the next post), so there's no urgency here. I'll revisit this — likely as a rebuild — once there's a real cloud node to add.

**Datastore: etcd, not k8s-dqlite.** k8s-dqlite is deprecated and gets removed in Canonical Kubernetes 1.36 with no upgrade path. etcd is already the default when you don't specify `datastore-type`, but I set it explicitly anyway, so the cluster's config doesn't quietly depend on a default that could change later.

## Why skip Juju?

Juju is an optional orchestration layer that can install and manage this same `k8s` snap for you — `juju deploy k8s` drives the identical CLI commands below, just with Juju as the operator instead of me typing them. I wanted to get hands-on with Kubernetes itself first. Juju/charm operation is a separate exercise on the same underlying distribution, for later.

## Install and bootstrap

```bash
sudo snap install k8s --classic
```

Canonical Kubernetes ships no CNI, DNS, ingress, or storage by default — unlike k3s, which bundles Flannel/Traefik/ServiceLB and turns them on automatically. Everything here is opt-in.

`sentinel` has three network interfaces, so I pinned the bind address explicitly rather than relying on autodetection:

```bash
cat > /tmp/bootstrap-config.yaml <<EOF
datastore-type: etcd
EOF

sudo k8s bootstrap --address 192.168.100.18 --file /tmp/bootstrap-config.yaml --timeout 3m
```

Right after bootstrap the node is `NotReady` — that's expected, since there's no CNI yet. I enabled exactly what I needed:

```bash
sudo k8s enable network         # Cilium — the one supported CNI for this distro
sudo k8s enable dns             # CoreDNS
sudo k8s enable local-storage   # rawfile CSI
```

Cilium took about 3 minutes to come up on the Pi (cold image pulls on a Pi's SD/NVMe are not fast), and `sentinel` flipped to `Ready` right after.

I left `ingress`, `load-balancer`, and `gateway` disabled. Canonical's own docs are blunt about it: if you need an Ingress controller, that's on you to bring via a normal Helm chart or manifest. My old k3s cluster shipped Traefik by default and I never actually used it — my one service goes out through a Cloudflare Tunnel straight to its Service, bypassing Ingress entirely. I may not need one here either.

## The mistake that decided my topology

I generated join tokens for the other two boards like this:

```bash
sudo k8s get-join-token watchtower
sudo k8s get-join-token cyclone
```

What I missed: `k8s get-join-token --help` has a `--worker` flag. Without it, the default is a **control-plane** token, not a worker one. Both boards joined as control-plane voters instead of the workers I'd planned.

I caught it when `k8s status` showed three voters instead of one. Rather than tear everything down and rejoin correctly, I compared the two topologies honestly:

| | 1 control-plane + 2 workers | 3-node HA (what I ended up with) |
|---|---|---|
| Survives losing any 1 node | Only if it's a worker — lose the control-plane node and the API server, scheduler, and etcd all go down | Yes — etcd keeps quorum with 2 of 3, no matter which node drops |
| Resource cost | Workers run no control-plane components | All three run etcd + API server + scheduler + controller-manager — negligible on this hardware |
| Adding a cloud node later | Adding it as a 2nd voter is **worse** than 1 (needs both alive, zero fault tolerance) | Add it as a worker — fits cleanly. A 4th voter would actually be a step backward (etcd wants odd numbers) |

3-node HA won on every axis that mattered to me. I kept it deliberately. If you're doing this on purpose: `k8s get-join-token <name> --worker` is the flag you want.

## Joining the rest of the cluster

```bash
# on watchtower / cyclone
sudo snap install k8s --classic
sudo k8s join-cluster <token>
```

The join token is single-use with a 24-hour expiry, generated fresh from `sentinel` for each board.

## Verifying it's actually a cluster, not just three Ready nodes

`kubectl get nodes` showing `Ready` three times isn't proof the thing works — it's proof three kubelets can report in. I wanted to see cross-node Service routing actually happen:

```bash
kubectl create deployment nginx-smoketest --image=nginx:latest
kubectl rollout status deployment/nginx-smoketest --timeout=90s
kubectl expose deployment nginx-smoketest --port 80 --type NodePort

NODE_PORT=$(kubectl get svc nginx-smoketest -o jsonpath="{.spec.ports[0].nodePort}")
for ip in 192.168.100.18 192.168.100.12 192.168.100.14; do
  curl -s -o /dev/null -w "HTTP %{http_code}\n" --max-time 5 http://$ip:$NODE_PORT/
done
```

The pod landed on `watchtower` only — but all three node IPs returned `HTTP 200`. That's kube-proxy/Cilium doing its job: routing to the pod regardless of which node you hit. (My first attempt returned `HTTP 000` on all three, because I ran curl before the image finished pulling — not a cluster problem, just an impatient `curl`.)

## Where it landed

| Host | Role | Datastore | CNI |
|---|---|---|---|
| sentinel | control-plane (voter) + worker | etcd | Cilium |
| watchtower | control-plane (voter) + worker | etcd | Cilium |
| cyclone | control-plane (voter) + worker | etcd | Cilium |

## Pointing kubectl at it from my Mac — the wrong port, twice

`k8s config` generates an admin kubeconfig. First attempt:

```bash
ssh sentinel@192.168.100.18 'sudo k8s config --server 192.168.100.18:6400' > ~/k8s-admin.conf
```

Installed it and ran `kubectl get nodes`:

```
Unable to connect to the server: tls: failed to verify certificate: x509: certificate is valid for 127.0.0.1, ::1, not 192.168.100.18
```

Port 6400 is `k8sd` — Canonical's own cluster-management daemon, the thing behind `bootstrap`/`join-cluster`/`status`. It is not the Kubernetes API server, and its certificate is only valid for loopback. I confirmed what's actually listening where:

```bash
sudo ss -tlnp | grep -E "6400|6443"
```
```
LISTEN  192.168.100.18:6400   k8sd
LISTEN  *:6443                kube-apiserver
```

The real API server is on the standard 6443, and its cert already carries the node's LAN IP as a SAN from bootstrap. Fixed:

```bash
ssh sentinel@192.168.100.18 'sudo k8s config --server 192.168.100.18:6443' > ~/k8s-admin.conf
mkdir -p ~/.kube && mv ~/k8s-admin.conf ~/.kube/homelab && chmod 600 ~/.kube/homelab
export KUBECONFIG=~/.kube/homelab
kubectl get nodes -o wide
```

One gotcha worth flagging: `--server` replaces the *entire* `host:port`, not just the host. Leave the port off and it's silently dropped rather than defaulting to something sane.

A note on HA and this kubeconfig: a kubeconfig's `server:` field is a single address — `kubectl` won't fail over to another control-plane node on its own. The cluster tolerates losing any one node, but if the node *named in your kubeconfig* goes down, that specific kubeconfig stops working even though the cluster itself is fine. I'm starting with a single-IP kubeconfig rather than standing up a VIP (`kube-vip` or similar) — if `sentinel` is ever down when I need `kubectl`, I'll just regenerate against `.12` or `.14` instead. A floating VIP is a reasonable later addition, not something I needed on day one.

## Re-bootstrapping GitOps

My GitOps repo's `task bootstrap` only does generic `kubectl apply`/Kustomize operations — nothing k3s-specific — so I expected it to work unchanged against the new cluster. Two surprises first:

**`task` itself was never installed on this Mac.** `brew install go-task` fixed that in ten seconds.

**A YAML bug that had been sitting in `Taskfile.yaml` since its very first commit, silently.** Four `desc:` lines had a second, unquoted `: ` inside parentheses — something like `desc: Sync a specific application (usage: task sync:app APP=app-name)`. YAML plain scalars can't contain a bare `: ` outside the key's own colon, so this broke parsing the moment `task` actually tried to read the file. It had simply never run in this repo before, so nothing had ever triggered it. Quoting all four values fixed it.

After that, `task bootstrap` ran clean: ArgoCD installed, root Application applied, everything else handed off to GitOps from there.

## Cert-manager: did I even need it?

Before chasing down why `dev-cert-manager` was failing to sync, I checked whether it was wired to anything at all:

```bash
grep -rl "ClusterIssuer\|kind: Issuer\|kind: Certificate\|cert-manager.io/" --include="*.yaml" .
grep -rl "kind: Ingress" --include="*.yaml" .
```

Both empty. No certificates, no Ingress — my one service goes out through a Cloudflare Tunnel, which terminates TLS at Cloudflare's edge. The cluster has never needed its own cert. `dev-cert-manager` was installed but connected to nothing, and had quietly been broken since before the teardown — nothing ever noticed because nothing depended on it.

I decided to fix it rather than rip it out. It's a Helm-values-only fix, no cluster-wide RBAC widening, and it keeps the option open for an in-cluster Ingress controller later at zero ongoing cost.

**Bug one — leader election landing in a namespace my ArgoCD project doesn't allow.** cert-manager's Helm chart creates leader-election `Role`/`RoleBinding` objects in `kube-system` by default. My `dev-core` AppProject deliberately restricts its destination to the `cert-manager` namespace only, so ArgoCD refused to create them, retried three times, and gave up — the whole app came back `Missing`, not just out of sync.

```yaml
# charts/cert-manager/dev-values.yaml
global:
  leaderElection:
    namespace: cert-manager
```

One value, under `global`, covers both the controller and cainjector. No functional change — with `replicaCount: 1` everywhere, there's only one leader-election candidate anyway; it trivially wins and holds the lease forever. Only the lease's location moved.

**Bug two — Cilium poisoning ArgoCD's resource tracking.** Even after the RBAC fix, the app stayed stuck `OutOfSync`, blamed on three `CiliumIdentity` objects that aren't part of the cert-manager chart at all. Cilium auto-generates a `CiliumIdentity` per unique pod label set for its own network-policy matching — and it copies the *entire* label set onto that object, including ArgoCD's own `app.kubernetes.io/instance` tracking label. ArgoCD's default tracking method is exactly that label, so it mistook Cilium's object for something it owned.

This isn't a cert-manager-specific bug — any app's pods on a Cilium-managed cluster will spawn a same-labeled `CiliumIdentity`. The fix belongs in ArgoCD's own config (`argocd-cm`), not any single Application's `ignoreDifferences`, because the application controller builds one shared resource cache *before* attributing anything to an app by label — `resource.exclusions` skips the object at the caching stage; `ignoreDifferences` only quiets diffs on objects already correctly matched.

```yaml
# bootstrap/argocd-install/argocd-cm-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  resource.exclusions: |
    - apiGroups:
      - "cilium.io"
      kinds:
      - CiliumIdentity
      clusters:
      - "*"
```

One Kustomize gotcha here: my first attempt included `namespace: argocd` in the patch's metadata, and it failed to match. The upstream `argocd-cm` resource has no `namespace` field at all until Kustomize's top-level `namespace:` transformer injects one — and that happens *after* patch matching, not before. Dropping `namespace:` from the patch fixed it.

After applying and restarting the ArgoCD application controller (it's a `StatefulSet`, so `kubectl rollout restart statefulset/argocd-application-controller`, not a Deployment restart) to drop its stale cache, everything came back `Synced` and `Healthy`.

## What's next

The cluster's solid now: 3-node HA, Cilium, GitOps fully wired back up. The one service I actually run on it — a self-hosted bookmark manager — had its own restore story, including a Cloudflare Tunnel issue that took some real digging to root-cause. That's the next post.
