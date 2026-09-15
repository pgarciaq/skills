---
name: install-hcp-in-beaker
description: Use when creating an OpenShift Hosted Control Planes lab on a Beaker hypervisor with kcli: KVM virtual machines, compact management cluster, Agent-platform hosted cluster with NodePool workers, assisted-service discovery ISO, HCP DNS records, and optional air-gapped metrics-operator deploy for HyperShift development and testing.
---

# Install HCP in Beaker

Deploy an OpenShift Hosted Control Planes (HyperShift) lab on a Beaker
hypervisor: compact management cluster plus at least one Agent-platform
hosted cluster with worker VMs, using kcli on KVM.

Proven path only: Agent platform, compact management cluster. KubeVirt-platform
hosted clusters and multi-hosted-cluster fleets are untested with this flow —
do not present them as supported here. Nothing in the flow is ARM-specific
except the live-ISO URL pattern, which parameterizes cleanly.

> Verified with: kcli 99.0, OCP 4.22.12 (`version=stable` + `tag=4.22`),
> MCE stable-2.17, local-path-provisioner v0.0.31, RHEL 10 hypervisor.
> Resolve versions on the box (patterns below) instead of trusting these pins.

## Parameters

| Parameter | Required | Default | Example |
|-----------|----------|---------|---------|
| Hypervisor SSH target | Yes | — | `root@hpe-apollo-cn99xx-16.khw.eng.rdu2.dc.redhat.com` |
| Management cluster name | No | `hcp-mgmt` | `lab-mgmt` |
| Hosted cluster name | No | `hc01` | `lab-hc01` |
| Base domain (lab-only) | No | `hcplab.corp` | `labtest.corp` (needs `/etc/hosts` + sshuttle, no real DNS) |
| OCP minor | No | `4.22` | `4.23` |
| Mgmt nodes | No | 3 compact × 16 vCPU / 48 GiB / 200 GiB | — |
| Hosted workers | No | 2 × 8 vCPU / 32 GiB / 120 GiB | — |

Set shell variables once per session and use them in every command below —
never hardcode one scenario's names into a reusable step:

```bash
HYP=root@<hypervisor-fqdn>; MGMT=hcp-mgmt; HOSTED=hc01; DOMAIN=hcplab.corp; TAG=4.22; INFRA=${HOSTED}-infra
```

Every `[hypervisor]` block below assumes
`export KUBECONFIG=/root/.kcli/clusters/$MGMT/auth/kubeconfig`
(management plane) unless it says otherwise; hosted-plane steps set
`KUBECONFIG=/root/$HOSTED-kubeconfig` (created in Phase 9).

Budget check before the 45–90 min install (abort if short):

```bash
# [hypervisor]
nproc; free -g | head -2; virsh pool-info default | grep -E 'State|Available'
# need ≥ ~64 threads / ~208 GiB free / ~1 TiB in pool for the defaults above
```

## Prerequisites

**Laptop paths below are laptop-only** (`~/rh/kcli/`); box paths are absolute
(`/root/.kcli/`). Never mix them: `~` on the box is `/root`, not your home.

- Laptop: OpenShift pull secret + SSH keypair (`~/rh/kcli/openshift_pull.json`,
  `~/rh/kcli/id_rsa[.pub]`), VPN + `sshuttle` for `192.168.122.0/24`.
- Hypervisor (RHEL 10): kcli, libvirtd active. If `default` net/pool are not
  `active`, create them first (see install-sno-in-beaker Phase 0):
  `virsh net-list --all; virsh pool-list --all`.
- Tools on the box — verify, install what's missing (all three have bitten):
  `which oc tmux; rpm -q edk2-aarch64` (ARM) then
  `dnf install -y podman tmux edk2-aarch64`; `kcli download oc` drops the
  binary in CWD — `mv ./oc /usr/local/bin/ && oc version --client`.
- Operator phase only (optional, last): Quay robot with push on a repo; a
  human runs `podman login` on the box. Tokens never transit chat, issues,
  or files.

All SSH uses `-o StrictHostKeyChecking=no`. Long runs go under
`tmux new -s <name>` on the box. For backgrounded commands, wrap:
`tmux new -d -s <name> sh -c '<cmd> > /root/<name>.log 2>&1'` (a bare
`tmux new -d` leaves no session when the command exits fast), then poll the
log file. Ground truth for installs is `oc get clusterversion`, never wrapper
log tails; `pgrep -f` patterns self-match the monitoring shell itself.

## Workflow

### Phase 1: Pre-flight (hypervisor)

```bash
HYP=root@HYPERVISOR  # then: ssh -o StrictHostKeyChecking=no $HYP "<cmd>"
systemctl is-active libvirtd; virsh net-list --all; kcli list pool; kcli version
mkdir -p /root/.kcli; ls /root/.kcli/openshift_pull.json /root/.kcli/id_rsa || echo MISSING
# if MISSING, from LAPTOP: scp ~/rh/kcli/openshift_pull.json ~/rh/kcli/id_rsa{,.pub} $HYP:/root/.kcli/ ; chmod 600 on the box id_rsa
grep -q '^allow 192.168.122' /etc/chrony.conf || echo 'allow 192.168.122.0/24' >> /etc/chrony.conf; systemctl restart chronyd
setfacl -m u:qemu:rwx /home/libvirt/images && restorecon -Rv /home/libvirt/images >/dev/null && virsh pool-info default | grep -E 'State|Available'
# aarch64 live ISO URL for $TAG (kcli defaults to x86_64) — assert exactly one match:
curl -s https://mirror.openshift.com/pub/openshift-v4/aarch64/dependencies/rhcos/$TAG/latest/ | grep -o 'rhcos-live-iso.aarch64.iso' | head -1
# full URL: https://mirror.openshift.com/pub/openshift-v4/aarch64/dependencies/rhcos/$TAG/latest/rhcos-live-iso.aarch64.iso
# if $TAG/latest/ is absent, list .../rhcos/$TAG/ and use the newest versioned dir
```

### Phase 2: Management compact cluster (SNO is unsupported as management; need ≥3 workers)

kcli 99 syntax is `create kube` / `ctlplanes` / `numcpus` (not
`create cluster` / `masters` / `cpus`); `version` takes only streams
(`stable`), the minor comes from `tag`; `domain` must be set explicitly
(the default is the plan author's domain); `keys` accepts a path or key text.

```bash
# [hypervisor] 45–90 min on slow cores — tmux, do not interrupt
tmux new -d -s $MGMT-install sh -c "kcli create kube openshift -P cluster=$MGMT -P domain=$DOMAIN -P version=stable -P tag=$TAG -P ctlplanes=3 -P workers=0 -P numcpus=16 -P memory=49152 -P disk_size=200 -P keys=[/root/.kcli/id_rsa.pub] -P liveiso_url=https://mirror.openshift.com/pub/openshift-v4/aarch64/dependencies/rhcos/$TAG/latest/rhcos-live-iso.aarch64.iso -P pull_secret=/root/.kcli/openshift_pull.json $MGMT > /root/$MGMT-install.log 2>&1"
# workers=0 makes the 3 ctlplanes schedulable = compact
```

Verify (gate — stop on any degraded CO; pass = empty output from the grep):

```bash
# [hypervisor]
export KUBECONFIG=/root/.kcli/clusters/$MGMT/auth/kubeconfig
oc get clusterversion; oc get nodes  # 3 Ready, roles control-plane,master,worker
test -z "$(oc get co --no-headers | grep -vE 'True +False +False')" && echo ALL_COS_HEALTHY
```

Pin node NTP (compact nodes carry the `master` role). The ignition
data-URL **must be double-quoted** — unquoted, the comma ends the
flow-mapping value, the payload drops silently, and the pool sits
`RenderDegraded` for hours with nodes unaffected:

```bash
# [hypervisor] full 99-master-chrony.yaml:
HIP=$(ip -4 addr show virbr0 | grep -oP '(?<=inet )[\d.]+')
CONF=$(printf 'server %s iburst\ndriftfile /var/lib/chrony/drift\nmakestep 1.0 3\nrtcsync\nkeyfile /etc/chrony.keys\nleapsectz right/UTC\nlogdir /var/log/chrony\n' "$HIP" | base64 -w0)
printf 'apiVersion: machineconfiguration.openshift.io/v1\nkind: MachineConfig\nmetadata:\n  labels:\n    machineconfiguration.openshift.io/role: master\n  name: 99-master-chrony\nspec:\n  config:\n    ignition: {version: 3.2.0}\n    storage:\n      files:\n      - contents:\n          source: "data:text/plain;charset=utf-8;base64,%s"\n        mode: 0644\n        overwrite: true\n        path: /etc/chrony.conf\n' "$CONF" > /tmp/99-chrony.yaml
oc apply -f /tmp/99-chrony.yaml
oc get mc 99-master-chrony -o jsonpath='{.spec.config.storage.files[0].contents.source}' | cut -c1-100  # payload MUST follow base64,
oc wait --for=condition=Updated mcp/master --timeout=900s  # nodes reboot rolling, ~10 min
```

MCP/MCO warning (proven): on a compact mgmt cluster hosting SingleReplica
control planes, rollouts hang in `FailedToDrain` — HCP PDBs (minAvailable 1,
0 allowed) forbid eviction, recreate ~30s after deletion (owned by
HostedControlPlane/CPO; pausing the HostedCluster does NOT stop it), and the
drain controller does not retry. Recovery: pause HC → scale CPO to 0 →
delete the 5 PDBs → verify hold ~3 min → bounce machine-config-controller →
reboot → rejoin → scale CPO up → unpause. Expect a hosted-API outage on the
reboot; mgmt quorum holds.

Laptop access (repo sshuttle pattern). API + ingress share one keepalived
VIP — discover it, don't assume node IPs:
`virsh net-dhcp-leases default` won't show VIPs; read the kcli install log
(`Using <ip> for api vip`) or `virsh net-dhcp-leases` + `.253` convention
(`Using 192.168.122.253 for api vip` observed). Then on LAPTOP:

```bash
# /etc/hosts, e.g.: 192.168.122.253 api.lab-mgmt.labtest.corp console-openshift-console.apps.lab-mgmt.labtest.corp oauth-openshift.apps.lab-mgmt.labtest.corp
sshuttle -r $HYP 192.168.122.0/24
oc login https://api.$MGMT.$DOMAIN:6443 -u kubeadmin # password: /root/.kcli/clusters/$MGMT/auth/kubeadmin-password on box
oc whoami  # must print kubeadmin — if not, fix VPN/sshuttle, no workarounds
```

### Phase 3: Storage for hosted etcd (lab-only host path, no redundancy)

```bash
# [hypervisor] resolve the tag first (never hardcode), then verify arch:
LPP_TAG=v0.0.31  # replace with newest after checking the repo releases
podman manifest inspect docker.io/rancher/local-path-provisioner:$LPP_TAG | grep -o '"architecture": *"[^"]*"' | sort -u  # must list the hypervisor arch
oc apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/$LPP_TAG/deploy/local-path-storage.yaml
SA=$(oc -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.spec.template.spec.serviceAccountName}')
oc adm policy add-scc-to-user privileged -z $SA -n local-path-storage
oc patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
for n in $(oc get nodes --no-headers -o custom-columns=N:.metadata.name); do oc debug node/$n -- chroot /host sh -c 'mkdir -p /opt/local-path-provisioner && chcon -Rt container_file_t /opt/local-path-provisioner'; done
oc get sc  # local-path present and default
```

Without the `chcon`, helper pods fail `mkdir: Permission denied`, PVCs stay
Pending, and assisted-service never starts.

### Phase 4: MCE + HyperShift + hcp CLI

Resolve, don't guess — newest `stable-2.x` from the box, e.g.:

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
oc get packagemanifest multicluster-engine -n openshift-marketplace -o jsonpath='{.status.channels[*].name}'
# namespace + OperatorGroup + Subscription(channel=<newest>) → wait CSV Succeeded (≤5 min)
# empty MultiClusterEngine CR → wait phase Available (several min)
oc get pods -n hypershift
oc get cm supported-versions -n hypershift -o jsonpath='{.data}'  # must list the target minor; stop if absent
```

hcp CLI per arch comes from `oc get ConsoleCLIDownload hcp-cli-download -o
json`; the route hostname won't resolve (explicit DNS records only), so
download through a port-forward (service port is 80, not 8080):

```bash
# [hypervisor]
oc -n multicluster-engine port-forward svc/hcp-cli-download 8080:80 >/tmp/pf.log 2>&1 &
sleep 5; curl -s --max-time 180 -o /root/hcp.tar.gz http://localhost:8080/linux/arm64/hcp.tar.gz; kill %1
tar xzf /root/hcp.tar.gz -C /root && chmod +x /root/hcp && /root/hcp version  # prints openshift/hypershift
```

### Phase 5: Admission recon (5 min, de-risks off-matrix early)

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
oc get validatingwebhookconfiguration | grep -iE 'hypershift|hosted|nodepool'
oc get crd nodepools.hypershift.openshift.io -o yaml | grep -B1 -A4 'only supported for'
# expect: arm64 allowed with platform agent (rule mentions has(self.platform.agent)); record findings, proceed
```

### Phase 6: Assisted service + InfraEnv

`AgentServiceConfig` is cluster-scoped (no namespace). Full manifests:

```yaml
# agent-service-config.yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata: {name: agent}
spec:
  databaseStorage: {storageClassName: local-path, accessModes: [ReadWriteOnce], resources: {requests: {storage: 20Gi}}}
  filesystemStorage: {storageClassName: local-path, accessModes: [ReadWriteOnce], resources: {requests: {storage: 20Gi}}}
  imageStorage: {storageClassName: local-path, accessModes: [ReadWriteOnce], resources: {requests: {storage: 20Gi}}}
```

```bash
# [hypervisor]
oc apply -f agent-service-config.yaml
oc wait --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}=True' agentserviceconfig/agent --timeout=600s
oc create namespace $INFRA
oc -n $INFRA create secret generic pull-secret --from-file=.dockerconfigjson=/root/.kcli/openshift_pull.json --type=kubernetes.io/dockerconfigjson
```

```yaml
# infra-env.yaml — render with the shell so $INFRA/$HIP expand (no literal placeholders):
# SSHKEY=$(cat /root/.kcli/id_rsa.pub); HIP=$(ip -4 addr show virbr0 | grep -oP '(?<=inet )[\d.]+')
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata: {name: ${INFRA}-env, namespace: ${INFRA}}
spec:
  pullSecretRef: {name: pull-secret}
  sshAuthorizedKey: ${SSHKEY}
  additionalNTPSources: ["${HIP}"]
  cpuArchitecture: arm64
```

```bash
# [hypervisor]
oc apply -f infra-env.yaml
oc -n $INFRA get infraenv -o jsonpath='{.items[0].status.isoDownloadURL}'  # retry until non-empty
```

### Phase 7: Worker VMs + agents (defaults 2× 8 vCPU / 32 GiB / 120 GiB)

```bash
# [hypervisor]
curl -skL --max-time 300 -o /home/libvirt/images/$HOSTED-discovery.iso "<isoDownloadURL>"  # -k: self-signed route CA
file /home/libvirt/images/$HOSTED-discovery.iso  # must say ISO 9660 bootable, ~99 MB minimal — a ~2.5 K file is an HTML error page, re-resolve DNS/download
for i in 0 1; do kcli create vm -P memory=32768 -P numcpus=8 -P disks=[120] -P nets=[default] -P iso=/home/libvirt/images/$HOSTED-discovery.iso $HOSTED-worker-$i; done
virsh list --all  # both running; VMs from a bad ISO need destroy + start (reboot may not re-read it); diagnose headlessly with virsh screenshot <vm> /tmp/x.ppm
# wait for 2 Agents, then approve + unique hostnames (both boot hostname-less) + installation disk from inventory if misdetected:
oc -n $INFRA get agents
i=0; for a in $(oc -n $INFRA get agents -o jsonpath='{.items[*].metadata.name}'); do oc -n $INFRA patch agent $a --type merge -p "{\"spec\":{\"approved\":true,\"hostname\":\"$HOSTED-worker-$i\"}}"; i=$((i+1)); done
```

### Phase 8: DNS (explicit records on all three layers — no wildcards exist)

kcli dnsmasq, box `/etc/hosts`, and laptop `/etc/hosts` all need the same
table; `virsh net-update` keeps one `<host>` block per IP (duplicate
hostnames across blocks are rejected — delete-then-add on change):

| Names (example values) | Target | Why |
|---|---|---|
| `api.lab-mgmt.labtest.corp`, `*.apps.lab-mgmt…`, assisted/agent-registration/image-service | keepalived VIP (.253) | mgmt serving endpoint |
| `api.lab-hc01.labtest.corp`, `api-int.lab-hc01.labtest.corp` | keepalived VIP | hosted API NodePort answers there |
| `console-…`, `oauth-…`, `canary-… .apps.lab-hc01…` | a worker IP from `virsh net-dhcp-leases` (HostNetwork router, NOT NodePort — DHCP, re-check on rebuild) | hosted ingress lives on workers |

```bash
# [hypervisor] example (substitute discovered IPs):
virsh net-update default add-last dns-host "<host ip='192.168.122.253'><hostname>api.lab-hc01.labtest.corp</hostname><hostname>api-int.lab-hc01.labtest.corp</hostname></host>" --live --config
virsh net-update default add-last dns-host "<host ip='192.168.122.36'><hostname>console-openshift-console.apps.lab-hc01.labtest.corp</hostname><hostname>oauth-openshift.apps.lab-hc01.labtest.corp</hostname><hostname>canary-openshift-ingress-canary.apps.lab-hc01.labtest.corp</hostname></host>" --live --config
# mirror every name into box /etc/hosts with the same IPs; verify with getent
```

### Phase 9: Hosted cluster (render first, never blind)

`--arch` defaults amd64 — the trap. `--release-image` needs the `-multi`
payload matching the minor — resolve, don't guess:
`RELEASE_IMAGE=$(oc adm release info quay.io/openshift-release-dev/ocp-release:$TAG.0-multi ...)` — verify with
`oc adm release info $RELEASE_IMAGE -o jsonpath='{.metadata.version}'`
then pass `--release-image=$RELEASE_IMAGE` (example: `...:4.22.12-multi`):

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
/root/hcp create cluster agent --name=$HOSTED --namespace=$INFRA --agent-namespace=$INFRA --base-domain=$DOMAIN --api-server-address=api.$HOSTED.$DOMAIN --pull-secret=/root/.kcli/openshift_pull.json --ssh-key=/root/.kcli/id_rsa.pub --etcd-storage-class=local-path --release-image=$RELEASE_IMAGE --node-pool-replicas=2 --control-plane-availability-policy=SingleReplica --arch arm64 --render > /tmp/$HOSTED-render.yaml
# inspect: NodePool arch reads arm64; release matches; namespaces right
```

`--render` omits generated secrets — create all three up front or reconcile
fails one at a time (`<hosted>-pull-secret` dockerconfigjson from the pull
secret file; `<hosted>-ssh-key` with key `id_rsa.pub`; `<hosted>-etcd-encryption-key`
Opaque with `data.key` = base64 of 32 random bytes:
`head -c 32 /dev/urandom | base64 -w0`). Then apply and watch
`HostedCluster` Available + NodePool machines ready (30–60+ min on slow
cores); `hcp create kubeconfig --name=$HOSTED --namespace=$INFRA`.
Acceptance: 2 Ready nodes, `controlPlaneTopology=External`, hosted
ClusterVersion complete. Hosted CP namespace is `<infra>-<hosted>`
(e.g. `hc01-infra-hc01`), NOT `clusters-*`.

### Phase 10 (optional): operator for HCP validation

Needs the Quay robot login on the box (human runs `podman login`;
MCE-console download link is a dead end — route DNS is explicit-only).
Deploy the arm64 operator air-gapped (`upload_toggle: false`), label HCP
namespaces `cost_management_optimizations=true` (auto-include does not
exist), confirm ROS CSV rows, download packages via a `volume-shell` pod
(ubi-minimal + `microdnf install tar`) + `oc cp`. Bare helper pods die on
node reboots — recreate as needed.

### Phase 11: Runbook

Write `~/rh/kcli/<hosted>-<hypervisor-short>/ACCESS.md` on the LAPTOP
(`~` = laptop home, not `/root`; short = hostname before the first dot)
from `access-template.md` — current facts only (versions, URLs, both
kubeadmin passwords, kubeconfig paths, node/VM IPs, StorageClass,
MCE/hcp versions, what was off-matrix).

## Pitfalls (each burned real time; inline sections have the fixes)

| Symptom | Cause |
|---|---|
| `Incorrect version 4.22`, instant exit | `version` takes streams only (`stable` + `tag`) |
| `create cluster` usage error | kcli 99 renamed to `create kube`; `ctlplanes`/`numcpus` too |
| Cluster under wrong domain | `domain` defaults to author's — always override |
| NodePool wrong arch | `--arch` defaults amd64 |
| Discovery ISO won't boot / 2.5 K file | x86 default / self-signed route / HTML error page |
| Agents never register | Unbootable ISO or missing assisted DNS in dnsmasq |
| Hosted API/console unreachable | Apps pointed at VIP instead of worker (HostNetwork) |
| `RenderDegraded: unterminated parameter sequence` | Unquoted ignition data URL (comma ends the value) |
| MCP stuck Updating, `FailedToDrain`, no retry | SingleReplica HCP PDBs block MCO drain (see NTP section) |
| Topology stays `{}` silently | RBAC applied to unprefixed ClusterRole — patch the live prefixed one, prove with `oc auth can-i` |
| No ROS rows / empty collection | HCP namespaces lack the label; check CR status message first |
| `oc cp` fails in helper pod | ubi-minimal has no `tar` |
