---
title: ArkCase-CE Installation
description: Using Talos and Proxmox to install ArkCase-CE
published: true
date: 2026-08-25T21:20:08.736Z
tags: 
editor: markdown
dateCreated: 2026-08-24T09:06:14.149Z
---

# Important note
The guide below was prepared by Claude using the Sonnet 5 model, with a little bit of manual tweaking.  It has been hand-verified to ensure that it works, but no guarantee is made as to accuracy of any other statements.

---

# ArkCase-CE Standalone Install: Talos on Proxmox, No ArgoCD

This is a from-scratch installation guide for ArkCase Community Edition on a single Talos Linux node running as a Proxmox VM, deployed with plain Helm rather than ArgoCD/GitOps. It exists because a production ArkCase-CE deployment managed by ArgoCD ran into a cluster of interacting bugs -- credential rotation on every sync, a chart that assumes real Helm release history ArgoCD never provides, `ignoreDifferences` silently failing to protect live patches -- serious enough that this guide deliberately steers around ArgoCD entirely. **Appendix A** has the full story; read it before you decide whether to reintroduce GitOps here later.

## Who this is for

You're comfortable with basic Linux administration and have a Proxmox host available, but you haven't necessarily used Kubernetes or Helm before. This guide explains both as they come up, and includes installing every tool you'll need. You do need your own domain name and the ability to create DNS records for Phase 2.

## The two-phase approach

Phase 1 gets ArkCase running with the Helm chart's own defaults: it answers to the hostname `server.dev.arkcase.com`, uses a certificate the chart generates itself (which your browser won't trust automatically -- you'll click through a warning), and you'll edit your computer's hosts file so that hostname resolves anywhere at all. This phase has one job: prove the install itself works, with the smallest number of moving parts possible.

Phase 2 replaces all of that with your own domain and a real, browser-trusted certificate. Nothing about Phase 1 is wasted -- you're changing configuration on the same running install, not starting over.

## Choosing an ingress controller: Traefik or HAProxy

Part 4b has you pick one of two fully supported paths -- **Traefik** or **HAProxy** (specifically the [jcmoraisjr/haproxy-ingress](https://github.com/jcmoraisjr/haproxy-ingress) project) -- and every later step that differs between them is clearly labeled by name, so you only need to read the subsection matching your choice. Pick Traefik if you want the same controller this guide's reference production cluster runs, or no particular preference either way. Pick HAProxy if you'd rather lean on ArkCase's own chart defaults directly -- its bundled `Ingress` resource already carries annotations written for this exact project, so that path needs noticeably fewer hand-authored manifests.

## A few Kubernetes concepts, briefly

You'll see these terms throughout; here's what they mean, in the order you'll meet them:

- **Cluster / node**: a Kubernetes cluster is one or more machines ("nodes") working together. This guide uses exactly one node -- your Talos VM plays every role at once.
- **Pod**: the basic unit Kubernetes runs -- one or more containers sharing a network address. ArkCase-CE is made of about a dozen pods (its Tomcat-based core app, a database, an LDAP directory, and so on), all running on your one node.
- **Namespace**: a folder-like grouping for related resources. You'll put everything ArkCase-related in a namespace called `arkcase-ce`, keeping it visually and administratively separate from cluster-level plumbing.
- **Service**: a stable network name/address in front of a pod (or group of pods), so other things in the cluster can reach it without caring which exact pod is currently running.
- **PersistentVolumeClaim / StorageClass**: a pod's request for durable disk space, and the thing that actually provisions it. Without a `StorageClass` configured, Kubernetes has no way to satisfy that request.
- **Secret**: a Kubernetes object for holding sensitive values -- passwords, certificates. You'll retrieve one directly to get ArkCase's auto-generated admin password.
- **Ingress / Ingress controller**: `Ingress` is the standard, provider-agnostic Kubernetes resource for mapping an external hostname/path to an internal Service; an *Ingress controller* is what actually enforces those rules (a rule alone does nothing without something to enforce it). Traefik additionally has its own, more capable `IngressRoute` resource, used in this guide's Traefik path for things standard `Ingress` can't express.
- **Helm / chart / release**: Helm is Kubernetes' package manager; a "chart" is a package (ArkCase-CE ships one); a "release" is one installed instance of a chart, tracked by Helm so that later upgrades know what's already there. That release-tracking is the specific thing ArgoCD skips, and why Appendix A matters.

---

## 0. Sizing

ArkCase's own published minimum (2 CPU / 16 GB RAM / 50 GB disk) is for an older, non-containerized install path and is **not** sufficient here. The Helm chart deploys ArkCase as roughly a dozen separate pods sharing one node: the Tomcat-based core application, OpenLDAP, PostgreSQL, Solr, ZooKeeper, ActiveMQ (messaging), two MinIO instances (content storage), a Pentaho-based reports service plus its cron sidecar, an HAProxy front for the core app, and an internal certificate-authority service. Five of those are JVMs, each wanting its own heap on top of baseline container overhead.

The chart itself sets no default resource requests or limits -- nothing stops any one container from consuming whatever the node has, which is fine on a dedicated single-node VM but means there's no chart-provided floor to size against. Based on the shape of that component list rather than any official number:

| Resource | Recommendation | Why |
|---|---|---|
| vCPU | 8 minimum, 16 preferred | Five concurrent JVMs, plus Postgres/Solr/MinIO doing real I/O under load |
| RAM | 32 GB minimum, 48-64 GB preferred | JVM heaps add up fast; give yourself headroom before you're debugging out-of-memory kills instead of installing software |
| Disk | 200 GB minimum, on SSD/NVMe-backed storage | Postgres, Solr indices, and MinIO object storage all do real random I/O; spinning disk will make this miserable |

Treat this as an informed starting point, not gospel -- watch for out-of-memory kills and slow Solr indexing after install and scale the VM up if you see them.

---

## Part 1: Proxmox VM Creation

Talos's own documentation for this exact scenario is at **[docs.siderolabs.com -- Proxmox](https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/virtualized-platforms/proxmox)** -- worth reading alongside this section, since Talos's recommendations occasionally change between versions.

1. **Download the Talos ISO.** Get the current `metal-amd64.iso` from [Talos's release page](https://github.com/siderolabs/talos/releases) (or use [Image Factory](https://factory.talos.dev/) if you want to bake in extra kernel modules -- not needed for this guide). Upload it to your Proxmox host: **Datacenter -> your node -> local -> ISO Images -> Upload**.

2. **Create the VM** ("Create VM" in the Proxmox web UI):
   - **General**: a name (e.g. `arkcase-ce`).  You'll likely also want to check "Start at boot."
   - **OS**: select the Talos ISO. Guest OS type: Linux, version 7.x - 2.6 Kernel.
   - **System**: Machine type `q35`, BIOS `OVMF (UEFI)` -- leave the box checked for "Add EFI Disk," and select an appropriate location for "EFI Storage."  Uncheck "Pre-Enroll Keys". Leave the Qemu Agent checkbox **unchecked** -- it only does anything useful if your ISO was built with the corresponding extension, and otherwise just generates log noise.  Set the SCSI Controller to **VirtIO SCSI** (not "VirtIO SCSI single" -- Talos's own docs call out that variant as a cause of bootstrap hangs).
   - **Disks**: one disk sized per the table above. Format Raw for best performance, or QCOW2 if you want Proxmox-level snapshots. Enable "Discard" if your storage pool is thin-provisioned SSD/NVMe.
   - **CPU**: per the sizing table, type `host` (passes through real CPU features; the tradeoff is you lose the ability to live-migrate this VM to a different physical CPU model -- irrelevant for a single-node, single-host setup).
   - **Memory**: per the sizing table. Leave **ballooning disabled** -- Talos doesn't support memory hotplug, and needs to see a fixed amount of RAM.
   - **Network**: model VirtIO, bridged to your LAN.
   - Finish the wizard **without starting the VM**.

3. **Boot the VM.** Talos boots into a maintenance mode with no configuration applied -- it sits listening on an API port for configuration to be pushed to it. Find its IP from the Proxmox console (or your DHCP server's lease list) -- you'll need it in a moment.

---

## Part 2: Installing the Tools You'll Need

Run everything in this section on your own workstation (your laptop/desktop), not on the VM -- Talos deliberately has no shell to log into; every interaction with it goes through its API.

**macOS** (using [Homebrew](https://brew.sh)):
```bash
brew install kubectl helm siderolabs/tap/talosctl
```

**Linux**:
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# talosctl
curl -sL https://talos.dev/install | sh
```

Confirm all three landed:
```bash
kubectl version --client
helm version
talosctl version --client
```
Now that these utilities are installed, all the other shell commands for the remainder of this guide will be run on this same workstation.

---

## Part 3: Talos Installation & Kubernetes Bootstrap

(Talos's own Proxmox-specific walkthrough, linked in Part 1, covers this same ground -- this section is the same process narrated for this guide's specific single-node setup.)

Set the VM's IP once, and reuse it for every command below:
```bash
export VM_IP=192.168.1.50
```
(Find it in the VM's own console in Proxmox -- Talos's maintenance-mode boot screen prints the address it picked up, and it's also visible in your DHCP server's lease list, per Part 1 step 3. This IP becomes the Kubernetes API's address, so ideally pick something stable -- a DHCP reservation, or the static IP set in step 3 below.)

1. **Generate a machine config.** This produces `controlplane.yaml` and `worker.yaml`; a single-node cluster only needs `controlplane.yaml`.
   ```bash
   talosctl gen config arkcase-ce https://$VM_IP:6443 --output-dir _talos-config
   ```

2. **Patch the config to allow ArkCase's pods to run on this node.** Kubernetes normally refuses to schedule ordinary workloads on a control-plane node, on the assumption you have separate worker nodes. You don't, so tell it otherwise. Create `_talos-config/patch.yaml`:
   ```yaml
   cluster:
     allowSchedulingOnControlPlanes: true
   ```
   Apply it into the generated config:
   ```bash
   talosctl machineconfig patch _talos-config/controlplane.yaml --patch @_talos-config/patch.yaml -o _talos-config/controlplane.yaml
   ```

3. **(Optional) Set a static IP**, if you'd rather not rely on a DHCP reservation. Add to the same patch file, under a `machine.network` key -- adjust to your actual subnet:
   ```yaml
   machine:
     network:
       interfaces:
         - interface: eth0
           addresses:
             - 192.168.1.50/24
           routes:
             - network: 0.0.0.0/0
               gateway: 192.168.1.1
       nameservers:
         - 192.168.1.1
   ```
   Re-run the patch command from step 2 if you add this after already patching once.

4. **Apply the config to the VM.** This is what actually installs Talos to the VM's disk and reboots it into a real, running system:
   ```bash
   talosctl apply-config --insecure --nodes $VM_IP --file _talos-config/controlplane.yaml
   ```
   `--insecure` is correct here -- the node has no certificates yet on this first contact. The VM will reboot on its own; give it a minute.

   *If this hangs or the node never comes back*: Talos occasionally installs to the wrong disk device name inside a VM (`/dev/sda` vs `/dev/vda`, depending on exact virtual hardware) -- using VirtIO SCSI as instructed in Part 1 avoids the version of this that trips people up, but if you do hit it, `talosctl get disks --insecure --nodes $VM_IP` shows what Talos actually sees, and you can add an explicit `install.disk` to the machine config pointing at the right one.

5. **Point `talosctl` at your node by default**, so you don't need to repeat `--nodes`/`--endpoints` on every command:
   ```bash
   export TALOSCONFIG=$(pwd)/_talos-config/talosconfig
   talosctl config endpoint $VM_IP
   talosctl config node $VM_IP
   ```

6. **Bootstrap the cluster.** This initializes Kubernetes' internal database (etcd) -- a one-time operation, run exactly once ever for this cluster, even if you later reboot or reconfigure the node:
   ```bash
   talosctl bootstrap
   talosctl health
   ```
   The health check can take a couple of minutes the first time.

7. **Retrieve the kubeconfig** -- the credentials file `kubectl` needs to talk to this cluster:
   ```bash
   talosctl kubeconfig .
   export KUBECONFIG=$(pwd)/kubeconfig
   kubectl get nodes
   ```
   You should see your one node, already `Ready` -- **no separate CNI install step needed.** Talos ships Flannel as its own default CNI and deploys it automatically as part of `talosctl bootstrap`, unless you explicitly opted out with a `cluster.network.cni.name: none` machine config patch (this guide never did, so yours is running it). It's easy to assume otherwise and manually apply the upstream `kube-flannel.yml` manifest on top "just in case" -- don't: that creates a second, independent Flannel installation (its own `kube-flannel` namespace and DaemonSet, alongside Talos's own in `kube-system`) running side by side with the one already there. Both will report `Running`, and networking may appear to work for a while, but two CNI daemons independently managing the same node's VXLAN interfaces and IP allocation is not a state to leave in place -- confirmed firsthand: it took a `kubectl delete -f` of that same manifest to get back to a single, consistent CNI.

   If you ever *do* want to run your own CNI choice instead of Talos's default, disable the built-in one before bootstrapping (`cluster.network.cni.name: none` in the patch from step 2), and the node will correctly show `NotReady` until you install one yourself.

---

## Part 4: Cluster Prerequisites

Two more things need to exist in the cluster before ArkCase itself goes in: somewhere for it to store data, and something to let it be reached from a browser.

**Two distinct PodSecurity behaviors show up repeatedly from here through the rest of this guide -- worth understanding once, up front, rather than re-diagnosing at every step:**
- A `kubectl apply`/`helm install` **warning** that a resource "would violate PodSecurity `restricted:latest`" is harmless noise. Storage provisioners, ingress controllers, and CNI plugins all need `hostPath` access or elevated privileges to do their job, so they trip Kubernetes' strictest built-in tier by design -- everything still gets created, nothing is misconfigured.
- Talos separately, actually *enforces* the more permissive `baseline` tier on every namespace except `kube-system` -- and `baseline`, unlike `restricted`'s warn-only posture, silently blocks anything needing `hostNetwork`, `hostPath` volumes, or `privileged: true` outright. This is a real, install-blocking problem, not noise, and it hit three separate places while writing this guide: **local-path-provisioner's own storage-creation helper pods** (below), **the ingress controllers** (Part 4b, `hostNetwork`), and **ArkCase's own `ldap` component** (Part 5, `privileged: true`). None of these show up as a helpful error -- `helm install`/`kubectl apply` reports success either way, and the actual blocking happens silently, later, when a controller tries to create the Pod object itself (`kubectl describe` on the stuck Deployment/StatefulSet/PVC is what actually surfaces it, citing PodSecurity `baseline`). The fix is the same each time: label the affected namespace `pod-security.kubernetes.io/enforce=privileged` before -- or, where the namespace is created as a side effect of applying a manifest rather than a separate step, immediately after -- whatever creates it.

### 4a. Storage

ArkCase-CE's dozen-odd pods need somewhere to persistently store data (databases, search indices, uploaded documents). Kubernetes calls the thing that provisions storage on request a `StorageClass`; **local-path-provisioner** is the simplest option for a single-node setup like this one -- it just carves storage out of the node's own disk:
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl label namespace local-path-storage pod-security.kubernetes.io/enforce=privileged
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
The label matters here specifically: local-path-provisioner doesn't create storage directories itself -- for every PVC, it spins up a short-lived "helper pod" (in this same `local-path-storage` namespace) that mounts a `hostPath` volume to actually create the directory on the node's disk. Without the label, every single one of those helper pods gets silently blocked by Talos's default `baseline` enforcement, which means *every* PVC in the cluster sits stuck `Pending` forever -- confirmed live: ArkCase's entire pod set (Part 5) stalled on this simultaneously, since virtually all of it needs storage. If you already ran the `kubectl apply` above before adding the label, either add it now and give the provisioner a minute to retry on its own, or force an immediate retry with `kubectl delete pod -n local-path-storage -l app=local-path-provisioner` (it restarts itself and re-queues everything right away, rather than waiting out its own retry backoff).

(If you later want volume snapshots/backups at the storage layer, [Longhorn](https://longhorn.io) is a common upgrade path -- it needs extra Talos-level setup to work, so it's out of scope for this guide's first pass.)

### 4b. Ingress controller: Traefik or HAProxy

Install the controller that will actually answer HTTP/HTTPS requests and route them into the cluster. (This deliberately isn't ingress-nginx, historically the most common default -- the Kubernetes project [retired that project in March 2026](https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/), no further security patches ever, so it's a poor foundation to build a fresh install on.) **Pick one** of the two paths below; every later step marked "Traefik path" or "HAProxy path" follows whichever you choose here.

**Both paths need one thing first**: Talos enforces the Kubernetes `baseline` Pod Security Standard on every namespace except `kube-system` by default, and `baseline` disallows `hostNetwork` outright (only the more permissive `privileged` tier allows it). Since both controllers below need `hostNetwork: true` to bind directly to the node's ports 80/443, create their namespace and exempt it *before* installing -- `helm install --create-namespace` alone creates a namespace with no such exemption, and the Deployment will sit at `0/1` forever with no pod at all (`kubectl describe deployment` would show `ReplicaFailure`/`FailedCreate` citing PodSecurity `baseline`, confirmed against a real Talos test cluster while writing this guide):
```bash
kubectl create namespace traefik   # or haproxy-ingress, depending on your choice below
kubectl label namespace traefik pod-security.kubernetes.io/enforce=privileged
```
(Drop `--create-namespace` from the `helm install` commands below, since the namespace already exists.)

#### Traefik path

Traefik is under active, well-resourced development and is what this guide's own reference production cluster already runs. This guide uses its native `IngressRoute` resource (rather than standard `Ingress`), which supports things standard `Ingress` can't -- specifically, skipping certificate verification on a single backend, needed because ArkCase's core service presents a certificate from ArkCase's own internal CA that nothing else in the cluster is set up to trust.
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik \
  --set hostNetwork=true \
  --set deployment.dnsPolicy=ClusterFirstWithHostNet \
  --set service.type=ClusterIP \
  --set ports.web.port=80 \
  --set ports.websecure.port=443 \
  --set providers.kubernetesIngress.enabled=false \
  --set podSecurityContext.runAsNonRoot=false \
  --set podSecurityContext.runAsUser=0 \
  --set podSecurityContext.runAsGroup=0 \
  --set securityContext.readOnlyRootFilesystem=false \
  --set securityContext.allowPrivilegeEscalation=true \
  --set securityContext.capabilities=null
```
`hostNetwork=true` binds Traefik directly to the node's own network interface on ports 80/443 -- the simplest option for a single-node cluster with no cloud load balancer to hand out an external IP. `providers.kubernetesIngress.enabled=false` tells Traefik to ignore standard `Ingress` resources entirely (including the one ArkCase's own chart creates by default -- see Part 5 step 5 -- which would otherwise sit there as an ambiguous, incorrectly-configured second route to the same backend, since it carries annotations written for a different controller entirely; see the HAProxy path below).

The `podSecurityContext`/`securityContext` overrides are needed for a less obvious reason: Traefik's chart runs its container as a non-root user with all Linux capabilities dropped by default (sensible hardening on its own), but binding to ports below 1024 needs the `NET_BIND_SERVICE` capability specifically. Adding just that one capability back (`securityContext.capabilities.add: [NET_BIND_SERVICE]`) while staying non-root was the first thing tried against a real Talos test cluster while writing this guide, and it did *not* work -- the container still failed with `bind: permission denied`. Only dropping to root and clearing the capability restriction entirely got it running. If you'd rather not run as root and want to chase a tighter fix, `kubectl logs -n traefik deployment/traefik --previous` will show the same `permission denied` error if you're still hitting it; the namespace is already labeled `privileged` above, so nothing about Kubernetes' own policy is what's stopping a lower-privilege configuration -- it's specific to how Talos's container runtime propagates that one capability for a `hostNetwork` pod, and wasn't fully root-caused here.

Confirm it's running: `kubectl get pods -n traefik`

#### HAProxy path

[jcmoraisjr/haproxy-ingress](https://github.com/jcmoraisjr/haproxy-ingress) -- note the exact project; "HAProxy Kubernetes Ingress Controller" from HAProxy Technologies themselves is a separate, similarly-named project with a different annotation scheme. This one is worth calling out specifically because ArkCase's chart-bundled `Ingress` resource already carries annotations (`haproxy-ingress.github.io/backend-protocol`, `haproxy-ingress.github.io/secure-backends`, etc.) written for exactly this controller -- meaning once it's installed, the chart's own default `Ingress` object (Part 5) just works, with no hand-authored routing resource needed at all.
```bash
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm repo update
helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
  --namespace haproxy-ingress \
  --set controller.hostNetwork=true \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set controller.service.type=ClusterIP \
  --set controller.ingressClassResource.enabled=true \
  --set controller.ingressClassResource.default=true
```
Same reasoning on `hostNetwork`/`service.type` as the Traefik path above; this controller listens on 80/443 by default, no extra port configuration needed, and confirmed live: unlike Traefik's chart, this one leaves its container `securityContext` empty by default (no non-root/capability restrictions imposed), so it binds ports 80/443 without needing the same root/capability overrides Traefik did.

The `ingressClassResource` flags matter more than they look: this chart does *not* create a Kubernetes `IngressClass` object by default. ArkCase's own bundled `Ingress` (Part 5) doesn't set an explicit `ingressClassName` either, so without a default `IngressClass` registered somewhere, haproxy-ingress has nothing telling it to claim that `Ingress` at all -- confirmed live: `kubectl get ingress` showed `CLASS: <none>` indefinitely, and the site 404'd, until this was fixed. If you installed without these flags and are already past this step, `kubectl get ingressclass` will come back empty -- add the flags via `helm upgrade` with the same command above, then patch the existing `Ingress` directly, since Kubernetes only auto-assigns a newly-created default class to *new* `Ingress` objects, not ones that already existed: `kubectl patch ingress arkcase-ce-app -n arkcase-ce --type=merge -p '{"spec":{"ingressClassName":"haproxy"}}'`.

Confirm it's running: `kubectl get pods -n haproxy-ingress`

*(A note on maintenance: jcmoraisjr/haproxy-ingress is actively released as of this writing, but -- like ingress-nginx was -- it's maintained by a small, mostly-volunteer team rather than a company. If that risk profile concerns you, HAProxy Technologies' own commercially-backed [haproxytech/kubernetes-ingress](https://github.com/haproxytech/kubernetes-ingress) is a third option, though it doesn't match the chart's bundled annotations and would need the same kind of hand-authored routing as the Traefik path -- not covered separately here to keep this guide to two tracks.)*

---

## Part 5: Phase 1 -- Install ArkCase-CE With Defaults

1. **Add the ArkCase Helm repo, and check you have the current chart version** (this guide was written against 0.9.53 -- check for anything newer, since ArkCase ships real bugfixes between releases):
   ```bash
   helm repo add arkcase https://arkcase.github.io/ark_helm_charts/
   helm repo update
   helm search repo arkcase/app --versions | head -5
   ```

2. **Create the namespace and exempt it from Talos's default PodSecurity enforcement first.** As covered in Part 4b, Talos enforces the `baseline` Pod Security Standard on every namespace except `kube-system`. ArkCase's own `ldap` component runs its container with `securityContext.privileged: true` (baked into the chart itself, not something you configure), which `baseline` disallows outright -- confirmed against a real Talos test cluster: `helm install` reports success, but the `ldap` StatefulSet never gets far enough to even create a Pod object, and everything else that depends on a reachable LDAP (most of the rest of ArkCase) sits stuck at `Pending` behind it.
   ```bash
   kubectl create namespace arkcase-ce
   kubectl label namespace arkcase-ce pod-security.kubernetes.io/enforce=privileged
   ```

3. **A decision to make before this install, only if you already know the answer.** ArkCase's chart seeds a handful of default LDAP users -- including the built-in `arkcase-admin` account you'll use in step 7 -- during this very first `helm install`, baked to whichever login domain is configured *at that moment*. If you already know you'll want a custom login domain later (e.g. matching your real domain from Part 6, rather than the chart's default `dev.arkcase.com`), this is the point to decide -- not Part 6, and not after you've created users you want to keep.

   Confirmed live against a real Talos test cluster: changing this after the fact doesn't quietly move existing users to the new domain. The `ldap` container checks the domain already baked into its persisted data against whatever's newly configured, and if they don't match, it refuses to start at all --
   ```
   This Samba persistent state belongs to the domain [dev.arkcase.com], but this pod is configured for the domain example.com
   ```
   -- rather than silently reprovisioning. Since `core` has nothing to authenticate anyone against while `ldap` is down, this blocks *all* logins, not just the admin account. Recovering means either reverting the domain back, or deliberately wiping the `ldap` StatefulSet's PersistentVolumeClaim and letting it provision fresh under the new domain -- which recreates `arkcase-admin` and the other seeded users from scratch, losing anyone added since.

   If you want a custom domain, create a `values.yaml` file now with the following (`rootDn` and the Kerberos realm are both derived from it automatically, unless you override them separately). **This is the one `values.yaml` this entire guide uses** -- Part 6 and Part 7 both add more keys to this same file later rather than starting new ones, since `helm upgrade -f values.yaml` replaces the whole set of customizations with whatever's in the file you pass it, not just the keys you're currently thinking about. Keep it around (and under version control, if you're the type) for the life of this install:
   ```yaml
   global:
     subsys:
       ldap:
         settings:
           arkcase:
             domain: "example.com"
           portal:
             domain: "example.com"
   ```
   **Both entries matter, even though this guide never uses the portal/FOIA feature.** The chart seeds a separate `portal` LDAP sub-tree unconditionally, regardless of whether the portal application itself is enabled -- confirmed live: overriding only `arkcase.domain` left `portal` on the old default, so the seed script tried to create the portal admin group under `dc=dev,dc=arkcase,dc=com`, which no longer existed once the actual provisioned domain became the new one. The result was a `Failed to get the SID for LDAP administrator ARKCASE_PORTAL_ADMINISTRATOR` error, and `ldap` crash-looping permanently -- wiping and retrying every time, hitting the identical failure on every attempt, never becoming `Ready` on its own. If you only need one domain and don't care about the portal feature, setting both entries to the same value (as above) is all that's needed.

   and include `-f values.yaml` on the install below. If you're not sure yet, or just want to confirm the install works first, the chart's default is completely fine for Phase 1 -- you're only closing this door once you've created users you actually want to keep.

4. **Install it**:
   ```bash
   helm install arkcase-ce arkcase/app -n arkcase-ce
   ```
   (No `--create-namespace` this time -- it already exists, from step 2. Add `-f values.yaml` here if you made the domain decision above.) `arkcase-ce` here is the *release name* -- Helm's label for this particular installed instance of the chart, which you'll reuse for every future `helm upgrade`. This will take a while: a dozen containers pulling images and starting up in dependency order. Watch it with:
   ```bash
   kubectl get pods -n arkcase-ce -w
   ```
   (Ctrl-C once everything shows `Running`/`1/1` -- the core application itself can take several more minutes after that to finish its own internal startup. If `ldap-0` doesn't appear within a minute or so even with the namespace correctly labeled, a `kubectl scale statefulset arkcase-ce-ldap -n arkcase-ce --replicas=0` followed immediately by `--replicas=1` will nudge it past the StatefulSet controller's retry backoff, the same fix used in Part 4b for Traefik.)

   By default, the chart also creates a self-signed certificate and a standard `Ingress` resource pointing at its own hostname (`server.dev.arkcase.com`).

5. **Set up routing to the `core` service.**

   **Traefik path**: the chart's auto-generated `Ingress` just sits there unused (Traefik's standard-`Ingress` handling is switched off, per Part 4b) -- but the self-signed certificate it generated alongside that `Ingress` is exactly what you need next, so it isn't wasted. Save this as `ingressroute.yaml`:
   ```yaml
   apiVersion: traefik.io/v1alpha1
   kind: ServersTransport
   metadata:
     name: arkcase-ce-core-insecure
     namespace: arkcase-ce
   spec:
     insecureSkipVerify: true
   ---
   apiVersion: traefik.io/v1alpha1
   kind: IngressRoute
   metadata:
     name: arkcase-ce
     namespace: arkcase-ce
   spec:
     entryPoints:
       - websecure
     routes:
       - kind: Rule
         match: Host(`server.dev.arkcase.com`)
         services:
           - name: core
             port: 8443
             scheme: https
             serversTransport: arkcase-ce-core-insecure
     tls:
       secretName: arkcase-ce-app-ingress
   ```
   `insecureSkipVerify: true` is what lets Traefik connect to `core` at all -- its certificate is signed by ArkCase's own bundled internal CA, which Traefik has no reason to trust, and there's no simple way to make it trust that CA without an ongoing maintenance burden (that CA reissues its own certificate periodically). This only affects the internal hop from Traefik to the `core` pod -- it has no effect on the certificate your browser sees. `tls.secretName` points at the self-signed certificate the chart generated in step 4 (named `<release-name>-app-ingress`; adjust if you used a different release name).
   ```bash
   kubectl apply -f ingressroute.yaml
   ```

   **HAProxy path**: nothing to do here beyond the `ingressClassResource` flags from Part 4b. The chart's auto-generated `Ingress` already has the right annotations for this controller, and it's already pointed at `server.dev.arkcase.com` with the self-signed certificate the chart generated -- haproxy-ingress picks it up automatically.

   **A second, separate cert-formatting issue to watch for**: confirmed live, the chart's self-signed certificate secret (`arkcase-ce-app-ingress`) sometimes renders its `tls.crt` without a trailing newline after `-----END CERTIFICATE-----`. Traefik never notices, since it reads the certificate and key as separate fields -- but haproxy-ingress concatenates them into one combined file for its `crt-list`, and the missing newline glues the cert's last line directly onto the key's `-----BEGIN ... KEY-----` line, which HAProxy rejects outright with `unable to load certificate ... bad end line`. Because HAProxy refuses to reload on *any* config error, this doesn't just break the new route -- the whole controller keeps serving whatever it last loaded successfully, which if this is your first Ingress ever, is nothing, hence a flat 404 for a host that otherwise looks correctly configured (`kubectl get ingress` shows the right class and host; the actual failure only shows up in `kubectl logs -n haproxy-ingress deploy/haproxy-ingress`).

   Since the chart has no way to check whether an existing self-signed cert is still valid under `helm template`-style rendering, it regenerates this certificate from scratch on every `helm upgrade` -- meaning this can recur after *any* upgrade you run while still on Phase 1's self-signed cert (it stops being possible once Phase 2 replaces it with a real, cert-manager-issued one). If you hit it, add the missing newline directly:
   ```bash
   kubectl get secret -n arkcase-ce arkcase-ce-app-ingress -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/tls.crt
   printf '\n' >> /tmp/tls.crt
   kubectl patch secret -n arkcase-ce arkcase-ce-app-ingress --type=json \
     -p "[{\"op\":\"replace\",\"path\":\"/data/tls.crt\",\"value\":\"$(base64 -i /tmp/tls.crt | tr -d '\n')\"}]"
   ```
   haproxy-ingress picks up the change and reloads automatically within a few seconds -- no restart needed.

6. **Point your workstation at the chart's default hostname.** Add a line mapping `server.dev.arkcase.com` to your VM's IP in your hosts file:
   - macOS/Linux: `/etc/hosts` (needs `sudo` to edit)
   - Windows: `C:\Windows\System32\Drivers\etc\hosts`
   ```
   192.168.1.50 server.dev.arkcase.com
   ```
   (substitute your VM's actual IP.)

7. **Retrieve the auto-generated admin password.** ArkCase generates this at install time and stores it in a Kubernetes Secret -- nothing to configure, just to read back out:
   ```bash
   kubectl get secret -n arkcase-ce arkcase-ce-core-main-admin -o jsonpath='{.data.password}' | base64 -d
   echo
   ```
   The username is fixed: `arkcase-admin`. The login domain defaults to `dev.arkcase.com` (so the full login is `arkcase-admin@dev.arkcase.com`) **unless you set a custom one in step 3**, in which case use that domain instead -- e.g. `arkcase-admin@example.com`. Either way, this login domain is separate from `baseUrl`/your Part 6 web hostname -- it's entirely possible, and normal, for the two to differ.

8. **Browse to `https://server.dev.arkcase.com/arkcase/login`.** Your browser will warn you about the certificate -- this is expected and is exactly what "uses a self-signed cert" means in practice: the chart generated this certificate itself, on the spot, and no browser trusts it by default. Click through the warning (usually "Advanced" -> "Proceed anyway", wording varies by browser). Log in with the credentials from step 7.

If you land on the ArkCase home screen, Phase 1 is done -- the install itself works. Everything from here forward is about how it's *exposed*, not whether it *works*.

---

## Part 6: Phase 2 -- Your Own Domain, With a Real Certificate

This phase needs, ahead of time: a domain name you control and the ability to create DNS records for it.

**Two placeholders recur throughout every file in this Part**: `arkcase.example.com` (your chosen hostname) and `you@example.com` (an email address Let's Encrypt uses for expiry notices, not anything ArkCase-related). Replace both with your real values everywhere they appear -- across `clusterissuer.yaml`, `certificate.yaml`, `values.yaml`, `ingressroute.yaml`/`root-redirect.yaml`, and `middleware.yaml` alike, not just the first file you happen to edit. A domain that's correct in `certificate.yaml` but still reads `arkcase.example.com` in `ingressroute.yaml` will fail quietly (usually as a hostname that doesn't match anything, or a certificate that doesn't match the hostname being served) rather than with an obvious error pointing at the mismatch.

### 6a. Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

### 6b. Issue a certificate, using a DNS-01 challenge

Let's Encrypt needs proof you control the domain before it issues a certificate. This example uses a **DNS-01 challenge**, which proves ownership via a temporary DNS record rather than a reachable web server -- it works identically regardless of which ingress controller you chose in Part 4b, and doesn't require opening any port to the public internet just to get the certificate issued. (If you're on the HAProxy path and would rather use the more common HTTP-01 challenge instead, since haproxy-ingress does process standard `Ingress` resources natively, that works too -- point `solvers[].http01.ingress.ingressClassName` at haproxy-ingress's class instead of the block below. Traefik path readers should stick with DNS-01, since Part 4b intentionally disabled the standard-`Ingress` handling HTTP-01 depends on.)

The exact solver block depends on your DNS provider ([cert-manager's docs](https://cert-manager.io/docs/configuration/acme/dns01/) list them all); this example uses Cloudflare, a common and free-tier-friendly choice. Create an API token in Cloudflare scoped to `Zone:DNS:Edit` for your domain, then:
```bash
kubectl create secret generic cloudflare-api-token -n cert-manager \
  --from-literal=api-token=<your-cloudflare-api-token>
```
Save this as `clusterissuer.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```
Then save this as `certificate.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: arkcase-ce-custom-tls
  namespace: arkcase-ce
spec:
  secretName: arkcase-ce-custom-tls
  dnsNames:
    - arkcase.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```
```bash
kubectl apply -f clusterissuer.yaml -f certificate.yaml
kubectl get certificate -n arkcase-ce -w
```
Wait for `READY: True`.

### 6c. Point DNS at your VM, and update ArkCase's configuration

Create an A record for your chosen hostname (e.g. `arkcase.example.com`) pointing at whatever address reaches your VM from wherever your actual users are -- your router's public IP with port 443 forwarded to the VM, if you want this reachable from the internet; the VM's LAN IP if this only needs to work inside your own network. (This is separate from the DNS-01 challenge above, which only needed a temporary TXT record cert-manager created and removed on its own.)

**This is the same `values.yaml` from Part 5 step 3** -- if you set a custom LDAP domain there, keep that block in the file and add `baseUrl` alongside it, in the same file, rather than starting a new one (see the note in Part 5 step 3 on why: `helm upgrade -f values.yaml` replaces your whole set of customizations with whatever's in the file, not just the keys you're adding right now). If you skipped that step, start a fresh `values.yaml` with just the block below.

**Traefik path** (the `ldap` block only applies if you customized the domain in Part 5 -- omit it otherwise):
```yaml
global:
  settings:
    baseUrl: "https://arkcase.example.com/arkcase"
  subsys:
    ldap:
      settings:
        arkcase:
          domain: "example.com"   # only if set in Part 5 step 3
        portal:
          domain: "example.com"   # only if set in Part 5 step 3
```

**HAProxy path** (same, plus one extra key pointing the chart's own `Ingress` at the real certificate instead of its self-signed one):
```yaml
global:
  settings:
    baseUrl: "https://arkcase.example.com/arkcase"
  ingress:
    secret: "arkcase-ce-custom-tls"
  subsys:
    ldap:
      settings:
        arkcase:
          domain: "example.com"   # only if set in Part 5 step 3
        portal:
          domain: "example.com"   # only if set in Part 5 step 3
```
Note `baseUrl`'s hostname (`arkcase.example.com`, matching your Part 6 domain) and the LDAP `domain` (`example.com`, matching your Part 5 step 3 decision) are two independent settings and don't need to be the same string -- a typical setup has users logging in as `user@example.com` while the application itself lives at `arkcase.example.com`.

`baseUrl` is the one setting ArkCase derives almost everything external-facing from -- its own bundled Ingress hostname, and the link embedded in "reset password" emails (Part 7 depends on this being right). It is **not additive**: changing it retires `server.dev.arkcase.com` rather than adding this domain alongside it.
```bash
helm upgrade arkcase-ce arkcase/app -n arkcase-ce -f values.yaml
```
From here on, **all further changes go through `helm upgrade` with this same values file** -- not by hand-editing anything live in the cluster. This is the single biggest reason this guide doesn't use GitOps for this app: real Helm release tracking (which `helm upgrade` provides and ArgoCD's render-and-diff style never does) is what lets the chart correctly reuse already-generated credentials instead of minting new ones on every change. See Appendix A for the incident that taught this lesson.

**HAProxy path**: that's it -- the chart's own `Ingress` picks up the new hostname and real certificate automatically. Skip to 6e.

### 6d. Update the IngressRoute for the new domain and certificate (Traefik path only)

Update your `ingressroute.yaml` from Part 5 step 5 to match the new hostname and point at the new, real certificate instead of the chart's self-signed one -- replace the `IngressRoute` block's contents with:
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: arkcase-ce
  namespace: arkcase-ce
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`arkcase.example.com`)
      middlewares:
        - name: arkcase-ce-root-to-login
      services:
        - name: core
          port: 8443
          scheme: https
          serversTransport: arkcase-ce-core-insecure
  tls:
    secretName: arkcase-ce-custom-tls
```
(The `ServersTransport` from Part 5 is unchanged and still referenced by name -- no need to recreate it. Don't apply this file yet -- the `middlewares` entry references a `Middleware` that doesn't exist until the next step, and the `kubectl apply` there covers both files together.)

### 6e. The mandatory bare-root redirect

This step isn't about your custom domain specifically -- it fixes a bug in ArkCase's own webapp, confirmed present with no SSO/OIDC configuration of any kind: an unauthenticated visit to the bare domain root, `/arkcase`, or `/arkcase/` unconditionally redirects into `/arkcase/oauth-login`, which itself redirects the browser into a broken, unfilled template placeholder and fails with a server error. Since visiting the bare domain is exactly what a real user does first, skipping this step means your users' first experience is an unexplained error page. `/arkcase/login` itself is unaffected and works fine -- the bug is purely in what happens before a user reaches it.

**Traefik path**: a `Middleware` intercepts just those three paths and redirects them straight to the working login page, before ArkCase's own (buggy) handling ever sees the request (this is the `middlewares:` reference already added to the `IngressRoute` above). Save this as `middleware.yaml`:
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: arkcase-ce-root-to-login
  namespace: arkcase-ce
spec:
  redirectRegex:
    regex: "^https://arkcase\\.example\\.com/(arkcase/?)?$"
    replacement: "https://arkcase.example.com/arkcase/login"
    permanent: false
```
```bash
kubectl apply -f middleware.yaml -f ingressroute.yaml
```

**HAProxy path**: haproxy-ingress lets you inject raw HAProxy configuration directly into a backend's config section via the `haproxy-ingress.github.io/config-backend` annotation. A small, separate `Ingress` pointing at the same `core` service carries the fix -- HAProxy merges annotations from every `Ingress` object referencing a given backend, so this doesn't disturb the chart's own routing. Save this as `root-redirect.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: arkcase-ce-root-redirect
  namespace: arkcase-ce
  annotations:
    haproxy-ingress.github.io/config-backend: |
      acl is-bare-root path / /arkcase /arkcase/
      http-request redirect code 302 location https://arkcase.example.com/arkcase/login if is-bare-root
spec:
  rules:
    - host: arkcase.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: core
                port:
                  number: 8443
  tls:
    - secretName: arkcase-ce-custom-tls
      hosts:
        - arkcase.example.com
```
The ACL only matches those three exact paths, so every other request to `core` (in particular `/arkcase/login` itself) passes through completely unaffected.
```bash
kubectl apply -f root-redirect.yaml
```

### 6f. Verify

Browse to `https://arkcase.example.com/` (no certificate warning this time) -- it should redirect straight to a clean login form. Log in with the same admin credentials from Part 5.

---

## Part 7: Setting Up Outgoing Email

New users get an email containing a password-reset link -- without working outgoing email, that link never arrives and new accounts are stuck. ArkCase's own documentation ([ArkCase/arkcase-ce Section Temporary Fix for Outgoing Email](https://github.com/ArkCase/arkcase-ce#set-up-outgoing-email)) covers this for its older, single-VM Vagrant-based install, where it means SSH-ing in and hand-editing a config file on disk. This Helm chart exposes the identical setting as ordinary values, which is both simpler and survives upgrades correctly (hand-editing a file inside a running pod, in a Kubernetes install, gets silently discarded the moment that pod restarts).

Same `values.yaml` from Parts 5 and 6 -- add the `email` block below to whatever's already in the file (the same field names as the original doc -- `connect` accepts `plaintext`, `ssl`, or `starttls`; the example below matches Office 365's settings, where you'd only need to change `username`, `password`, and `from`). The full, cumulative file looks like this by now:
```yaml
global:
  settings:
    baseUrl: "https://arkcase.example.com/arkcase"
    email:
      send:
        connect: starttls
        host: "smtp.office365.com"
        port: 587
        username: "you@example.com"
        password: "your-mail-password"
        from: "you@example.com"
  ingress:
    secret: "arkcase-ce-custom-tls"   # HAProxy path only
  subsys:
    ldap:
      settings:
        arkcase:
          domain: "example.com"   # only if set in Part 5 step 3
        portal:
          domain: "example.com"   # only if set in Part 5 step 3
```
```bash
helm upgrade arkcase-ce arkcase/app -n arkcase-ce -f values.yaml
```
Unlike `baseUrl` (which changes an environment variable on `app-proxy`'s own pod spec, triggering an automatic restart), this setting only changes the *content* of a Secret `core` mounts via subPath -- and subPath-mounted files don't refresh just because their backing Secret changed, confirmed live: the rendered Secret had the right values immediately after `helm upgrade`, but the running `core` pod kept serving the old `localhost:25` config indefinitely until manually restarted. Restart it explicitly:
```bash
kubectl delete pod arkcase-ce-core-0 -n arkcase-ce
```
(Kubernetes recreates it automatically -- give it a few minutes for `core`'s own startup, same as any other core restart in this guide.) Skipping this step is exactly the kind of thing that looks like the values didn't work, when they actually did -- worth checking the Secret directly (`kubectl get secret -n arkcase-ce arkcase-ce-core-files -o jsonpath='{.data.arkcase-server\.yaml}' | base64 -d`) before assuming a values.yaml mistake if email still isn't going out after this.

---

## Part 8: Adding a New User

This part is pure ArkCase application behavior -- identical regardless of how or where it's deployed. From [the same ArkCase-CE doc](https://github.com/ArkCase/arkcase-ce#how-to-add-a-new-user), adapted only where the login domain differs from the original (which assumes the older install's fixed `arkcase.org` domain). This chart's own default is `dev.arkcase.com` -- **but if you set a custom domain in Part 5 step 3, use that domain everywhere below instead.** The examples here show the chart default; substitute your own domain (uppercased, for the LDAP group name specifically) if you customized it.

1. Log in as the admin user from Part 5.
2. Click **Admin** in the left-hand navigation.
3. Click **Security / Organizational Hierarchy**.
4. Click the name of the group you want to add the user to (e.g. `ARKCASE_ADMINISTRATOR@DEV.ARKCASE.COM` to add another administrator -- `ARKCASE_ADMINISTRATOR@YOURDOMAIN.COM` if you customized the domain).
5. Click the **Add New Member** icon (the plus sign).
6. Fill out and submit the form.

The new user receives an email with a password-reset link (this is why Part 7 needs to happen first). **Regardless of the email address entered on the form, they log in as `userid@dev.arkcase.com`** (or `userid@yourdomain.com`, if you customized it in Part 5 step 3), where `userid` is whatever user ID was typed into the form -- not their email address. This is an ArkCase-wide quirk, not specific to this deployment.

---

## Part 9: Post-Install Verification

- `kubectl get pods -n arkcase-ce` -- everything `Running`/`Ready`, no crash loops. Give it several minutes after any restart; the core application's own startup alone commonly takes 2-5 minutes.
- Visit your site in a private/incognito browser window (no leftover cookies from testing) -- confirm it lands cleanly on the login form.
- Log in, then log out, then confirm revisiting the site afterward returns you cleanly to the login form rather than an error.

---

## Part 10: Ongoing Operations

- **Upgrades and any config change**: edit `values.yaml`, then `helm upgrade arkcase-ce arkcase/app -n arkcase-ce -f values.yaml`. This is the entire story -- no separate re-install, no manual patching.
- **subPath-mounted config doesn't live-update**: several ArkCase config files are mounted into the core pod in a way that doesn't refresh automatically when their backing config changes, even via a proper `helm upgrade`. If a change doesn't seem to take effect, `kubectl delete pod arkcase-ce-core-0 -n arkcase-ce` (Kubernetes recreates it automatically) and give it a few minutes to restart.
- **Backups**: at minimum, you need point-in-time coverage of the PostgreSQL data and the MinIO content-storage data -- that's where actual case data and documents live. With local-path-provisioner, you're responsible for your own backup mechanism at that layer (e.g. a scheduled `pg_dump`, `mc mirror` for the MinIO buckets); Longhorn (mentioned in Part 4a) adds built-in volume snapshots if you migrate to it later.

---

## Appendix A: Why Not ArgoCD

This section exists because the natural instinct -- "everything else I run is GitOps-managed, why not this too" -- is exactly the instinct that caused a real, multi-day production incident on the deployment this guide is drawn from. If you're tempted to bring this install under ArgoCD (or Flux, or any other continuous-reconciliation GitOps tool) later, read this first.

**The core problem**: GitOps controllers reconcile by re-rendering the chart from scratch on every sync and applying the result -- effectively `helm template` followed by `kubectl apply`, not a real `helm upgrade`. Helm's release-history mechanism, which lets a chart's templates ask "is this a fresh install, or an upgrade of something that already exists," never engages under that model. The ArkCase chart uses exactly that mechanism (internally, via a helper called `__arkcase.get-existing`) to decide whether to reuse a previously-generated random credential or mint a new one -- and under GitOps-style reconciliation, that check always resolves to "fresh install," so it mints a new one. Every single sync.

Concretely, on the reference deployment, this meant:
- **All ~23 auto-generated credential Secrets** (LDAP bind password, database passwords, message-queue credentials, the internal CA's provisioner password) regenerated on every sync, while the actual backing services kept running with whatever they'd been given at their last real pod restart -- a growing mismatch that broke authentication unpredictably.
- **`ignoreDifferences`**, ArgoCD's supported mechanism for "don't touch this field," turned out unreliable in ways that got progressively worse: it protected whole-Secret data most of the time, failed outright for ConfigMap sub-keys and StatefulSet array fields, and even failed for whole-Secret data during a chart-version-changing sync (never fully root-caused).
- The eventual mitigation -- a separate "vault" holding the 23 credentials as the real source of truth, reconciled back into place by a post-sync hook -- worked, but introduced its own deadlock: the hook can't run until the main sync's resources report healthy, but some of those resources can't become healthy while running on the sync's freshly-regenerated, wrong credentials. This recurred on *any* sync of the app, including ones that touched nothing but an unrelated ingress resource.

None of this is a criticism of ArgoCD in general -- it's a structural mismatch between how GitOps controllers render Helm charts and how this particular chart, reasonably on its own terms, expects to be operated. Real `helm upgrade`, as used throughout this guide, makes the entire problem not exist in the first place.

**If you use GitOps here anyway**, at minimum: expect every sync to be a coin-flip on credential correctness and plan a reconciliation mechanism from day one; never trust `ignoreDifferences` without testing real authentication after every sync; and watch for the deadlock above on *any* sync, not just ones that look credential-related.

## Appendix B: Other Chart Quirks Worth Knowing

- **`ARKCASE_APPLICATION_ACTIVE`** (an environment variable the chart sets on the core container, hardcoding an `arkcase-oidc` entry into its profile list regardless of whether SSO is configured) is a red herring if you ever go looking for the bare-root redirect bug from Part 6e -- removing it has no effect on that bug, and it can break the login form's own POST handling in a way that looks like a regression but isn't (the login form posts to `/arkcase/login_post`, not `/arkcase/login` -- testing the wrong URL will make a perfectly healthy install look broken). Leave this variable alone.
- **Chart-version-to-version bugs are real, and get fixed upstream** -- the reference production deployment hit a ZooKeeper config bug, a hardcoded TLS-1.2-only messaging config, and a deployer image that never installed fetched CA certs into the OS trust store, all fixed by a chart upgrade rather than a workaround. Check for a newer chart version before working around something that looks like a bug -- it may already be fixed.