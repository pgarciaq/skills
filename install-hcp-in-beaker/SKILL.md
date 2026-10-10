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

> Verified with: kcli 99.0, OCP 4.22.14 (`version=stable` + `tag=4.22`),
> MCE stable-2.17 (CSV `multicluster-engine.v2.17.2`), local-path-provisioner
> v0.0.37, RHEL 10.2 hypervisor. Phase 10 verified 2026-09-27 with
> `koku-metrics-operator` at `c3a769e4` (`pgarciaq-rosocp-superpowers-phase17`),
> image `quay.io/pgarciaq/koku-metrics-operator:w0.1-20260927-0828` (arm64).
> Resolve versions on the box (patterns below) instead of trusting these pins.
>
> **`tag=4.22` is not a fixed target — it floats.** Between 4.22.12 and 4.22.14
> the OLM OperatorGroup reconciler started stripping `installModes`, and
> `AgentServiceConfig` dropped its `Ready` condition. Both broke steps below
> that had previously passed. Re-verify the version-sensitive steps (Phases 3,
> 4, 6) when the resolved z-stream moves, and record what you find here.

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

Disk defaults total 840 GB nominal (3×200 + 2×120) against the ~1 TiB pool
assumption in the budget check, leaving little headroom on a smaller pool —
drop to 3×150 + 2×100 (650 GB) or 3×120 + 2×80 (520 GB) when
`virsh pool-info default` shows less. qcow2 is sparse, so real usage runs well
below nominal, but the margin is what protects you later.

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
df -h / /home   # `/` matters separately: local-path-provisioner defaults to /opt (on /)
# need ≥ ~64 threads / ~208 GiB free / ~1 TiB in pool for the defaults above
# need ≥ ~60 GiB free on `/` unless you relocate nodePath (Phase 3) — assisted
# service alone requests 3x20Gi PVCs, and a 70 GB `/` will fill during Phase 6
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

`nodePath` defaults to `/opt/local-path-provisioner`, i.e. on `/` — **not** on
the libvirt pool. If `df -h /` has less than ~60 GiB free, relocate it onto the
pool filesystem before applying, or Phase 6 will ENOSPC. The path occurs exactly
once in the manifest (ConfigMap `local-path-config`), so a single `sed` is safe
and complete. Set `LPP=/opt/local-path-provisioner` normally, or
`LPP=<pool-fs>/local-path-provisioner` when `/` is small:

```bash
# [hypervisor] resolve the tag first (never hardcode), then verify arch:
LPP_TAG=v0.0.37  # replace with newest after checking the repo releases
podman manifest inspect docker.io/rancher/local-path-provisioner:$LPP_TAG | grep -o '"architecture": *"[^"]*"' | sort -u  # must list the hypervisor arch
LPP=/opt/local-path-provisioner   # relocate only if `/` is small
curl -s https://raw.githubusercontent.com/rancher/local-path-provisioner/$LPP_TAG/deploy/local-path-storage.yaml \
  | sed "s#/opt/local-path-provisioner#$LPP#g" | oc apply -f -
oc -n local-path-storage get cm local-path-config -o jsonpath='{.data.config\.json}' | grep -o '"paths":\[[^]]*\]'  # PROVE the path took
SA=$(oc -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.spec.template.spec.serviceAccountName}')
oc adm policy add-scc-to-user privileged -z $SA -n local-path-storage
oc patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
for n in $(oc get nodes --no-headers -o custom-columns=N:.metadata.name); do oc debug node/$n -- chroot /host sh -c "mkdir -p $LPP && chcon -Rt container_file_t $LPP"; done
oc debug node/<any-node> -- chroot /host ls -Zd $LPP   # must read system_u:object_r:container_file_t:s0
oc get sc  # local-path present and default
```

Without the `chcon`, helper pods fail `mkdir: Permission denied`, PVCs stay
Pending, and assisted-service never starts.

### Phase 4: MCE + HyperShift + hcp CLI

Resolve, don't guess — newest `stable-2.x` from the box, e.g.:

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
oc get packagemanifest multicluster-engine -n openshift-marketplace -o jsonpath='{.status.channels[*].name}'
oc get packagemanifest multicluster-engine -n openshift-marketplace -o jsonpath='{.status.defaultChannel}'
oc get pods -n hypershift   # after install
oc get cm supported-versions -n hypershift -o jsonpath='{.data}'  # must list the target minor; stop if absent
```

**Two version-sensitive traps in the Subscription/OperatorGroup pair.** Both
bit a 4.22.14 run; the older YAML that circulates for this phase cannot work.

1. **The two CRDs use different API versions.** `Subscription` is still
   `operators.coreos.com/v1alpha1`, but `OperatorGroup` stopped serving
   `v1alpha1` and is now `v1`/`v1alpha2`. Using `v1alpha1` for the OperatorGroup
   fails with `no matches for kind "OperatorGroup" in version
   operators.coreos.com/v1alpha1`. Confirm on the box with
   `oc get crd operatorgroups.operators.coreos.com -o jsonpath='{.spec.versions[*].name}'`.
2. **MCE 2.17's CSV supports only `OwnNamespace`.** Check what the CSV actually
   declares before choosing a mode — `oc get csv -n multicluster-engine
   -o jsonpath='{.items[0].spec.installModes}'` returned
   `[{"supported":true,"type":"OwnNamespace"}, ...all others false]` on
   v2.17.2. `targetNamespaces: [openshift]` infers `SingleNamespace` → CSV fails
   with `UnsupportedOperatorGroup: SingleNamespace InstallModeType not
   supported`. Do **not** substitute `installModes: [AllNamespaces]` either: the
   4.22 OLM reconciler strips `installModes` from the spec on create (leaving
   only `upgradeStrategy`), so you get `AllNamespaces InstallModeType not
   supported` instead. Point the OperatorGroup at its own namespace:

```yaml
# [hypervisor] KUBECONFIG = mgmt kubeconfig
apiVersion: operators.coreos.com/v1alpha2      # v1alpha2, NOT v1alpha1
kind: OperatorGroup
metadata:
  name: mce-operator
  namespace: multicluster-engine
spec:
  targetNamespaces:
  - multicluster-engine                        # OwnNamespace — the only mode MCE 2.17 supports
---
apiVersion: operators.coreos.com/v1alpha1      # v1alpha1 is still correct for Subscription
kind: Subscription
metadata:
  name: mce-subscription
  namespace: multicluster-engine
spec:
  channel: stable-2.17                         # newest stable-2.x resolved above
  name: multicluster-engine
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Wait for the CSV to reach `Succeeded`, then create the empty `MultiClusterEngine`
CR (several minutes to `Available`):

```bash
# [hypervisor]
oc apply -f - <<'EOF'
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata: {name: multiclusterengine}
spec: {}
EOF
oc get multiclusterengine   # wait for STATUS=Available
oc -n hypershift wait pod --all --for=condition=Ready --timeout=420s
```

If the CSV lands `Failed`, delete the Subscription, OperatorGroup and CSVs
together and re-apply — a failed CSV is not re-resolved by editing the
OperatorGroup in place. Consequence of OwnNamespace: the MCE **console plugin
is not served**. Irrelevant for this CLI/CRD-driven flow; don't reach for the
console.

hcp CLI per arch comes from `oc get ConsoleCLIDownload hcp-cli-download -o
json`; the route hostname won't resolve (explicit DNS records only), so
download through a port-forward (service port is 80, not 8080):

```bash
# [hypervisor]
oc -n multicluster-engine port-forward svc/hcp-cli-download 8080:80 >/tmp/pf.log 2>&1 &
sleep 5; curl -s --max-time 180 -o /root/hcp.tar.gz http://localhost:8080/linux/arm64/hcp.tar.gz; kill %1
tar xzf /root/hcp.tar.gz -C /root && chmod +x /root/hcp && /root/hcp version  # prints openshift/hypershift
# "Latest supported OCP: <minor>" must cover $TAG
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
# GATE: do NOT wait on a "Ready" condition — AgentServiceConfig does not publish
# one in 4.22 (DeploymentsHealthy/LocalClusterManaged/... only), so an
# `oc wait --for=jsonpath=...Ready` gate always times out on a healthy install.
# Gate on these three instead:
oc get agentserviceconfig agent -o jsonpath='{range .status.conditions[*]}{.type}={.status}{" "}{end}'   # want DeploymentsHealthy=True
oc get pvc -n multicluster-engine            # 3x Bound (assisted-service, postgres, image-service)
oc get pods -n multicluster-engine | grep assisted   # assisted-service + assisted-image-service-0 Running
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
| **`*.apps.lab-hc01…`** (wildcard) **plus** `console-…`, `oauth-…`, `canary-…` | a worker IP from `virsh net-dhcp-leases` (HostNetwork router, NOT NodePort — DHCP, re-check on rebuild) | hosted ingress lives on workers |

**The hosted `*.apps.<domain>` wildcard is mandatory, not optional.** The
assisted agent validates `apps-domain-name-resolved-correctly` by resolving
`*.apps.<hosted-domain>`; without that exact wildcard entry both agents report
`Validated=False / ValidationsFailing`, never install, and the NodePool machines
sit `Pending` / `WaitingForNodeRef` indefinitely while the hosted control plane
reports Available — a confusing split. Note the asymmetry: the *management* row
above already carries a wildcard, the hosted row historically did not. The
wildcard goes into the dnsmasq host-record even though `getent` cannot look up a
literal `*` name — the agent issues a real DNS query for it.

Add the hosted record **before** the workers install, using the worker's DHCP
lease. The lease survives the RHCOS install, so the address you point at now is
the address the router will bind later. Map agent → MAC → lease to pick a
specific worker (`oc -n $INFRA get agent -o jsonpath='{..status.inventory.interfaces[*].macAddress}'`),
then confirm the router's `hostIP` after install
(`oc -n openshift-ingress get pod -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName,HOSTIP:.status.hostIP --no-headers`).

```bash
# [hypervisor] VIP block — one <host> element holding every VIP-served name.
# Use `modify` to replace an IP's existing block, `add-last` to create a new one
# (`modify` on an IP with no existing record fails: "couldn't locate a matching
# DNS HOST record").
virsh net-update default add-last dns-host "<host ip='192.168.122.253'><hostname>api.lab-mgmt.labtest.corp</hostname><hostname>api-int.lab-mgmt.labtest.corp</hostname><hostname>console-openshift-console.apps.lab-mgmt.labtest.corp</hostname><hostname>oauth-openshift.apps.lab-mgmt.labtest.corp</hostname><hostname>canary-openshift-ingress-canary.apps.lab-mgmt.labtest.corp</hostname><hostname>assisted-image-service-multicluster-engine.apps.lab-mgmt.labtest.corp</hostname><hostname>assisted-service-multicluster-engine.apps.lab-mgmt.labtest.corp</hostname><hostname>agent-registration-multicluster-engine.apps.lab-mgmt.labtest.corp</hostname><hostname>api.lab-hc01.labtest.corp</hostname><hostname>api-int.lab-hc01.labtest.corp</hostname></host>" --live --config
# Worker block — the wildcard FIRST; the agent will not install without it
virsh net-update default add-last dns-host "<host ip='192.168.122.36'><hostname>*.apps.lab-hc01.labtest.corp</hostname><hostname>console-openshift-console.apps.lab-hc01.labtest.corp</hostname><hostname>oauth-openshift.apps.lab-hc01.labtest.corp</hostname><hostname>canary-openshift-ingress-canary.apps.lab-hc01.labtest.corp</hostname></host>" --live --config
# mirror every name into box /etc/hosts with the same IPs
```

**`virsh net-update --live` is not enough — signal dnsmasq.** It rewrites
`/var/lib/libvirt/dnsmasq/default.addnhosts`, but dnsmasq does not reliably
re-read that file. Symptom: the records are visibly in the file, `getent` still
returns nothing, and the agents stall on DNS. After every net-update:

```bash
# [hypervisor]
kill -HUP $(ss -lnup | grep '192.168.122.1:53' | sed -E 's/.*pid=([0-9]+).*/\1/' | head -1)
```

**Verify DNS the way a guest sees it.** `dig`, `nslookup` and `host` are not
installed on a stock RHEL box — empty output from them means *not installed*,
not NXDOMAIN, and reading it as NXDOMAIN sends you chasing a DNS problem that
doesn't exist. Use a management node's resolver, which points at the lab
dnsmasq exactly as the workers' does:

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
oc debug node/<mgmt-node> -- chroot /host getent hosts api.lab-hc01.labtest.corp
# a literal '*' will never resolve via getent — that's expected; trust the
# agent's own validation for the wildcard.
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
Opaque). Then apply and watch `HostedCluster` Available + NodePool machines
ready (30–60+ min on slow cores).

**The etcd encryption key must hold 32 RAW bytes.** The apiserver reads the
mounted `key` file and requires exactly 16, 24 or 32 bytes. Do **not** use
`head -c 32 /dev/urandom | base64 -w0` with `--from-literal` — that stores a
44-character ASCII string, and the apiserver crash-loops for the whole install:

```
error while parsing file: resources[0].providers[0].aescbc.keys[0].secret:
Invalid value: "REDACTED": secret is not of the expected length, got 44,
expected one of [16 24 32]
```

Symptom to recognise: `HostedCluster` Available stalls on `Waiting for Kube
APIServer deployment to become available` with `kube-apiserver` in
`CrashLoopBackOff`. `data.key` is 44 base64 chars in the API *when it is
correct* — the mounted file is what must be 32 bytes, so use `--from-file`:

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
head -c 32 /dev/urandom > /tmp/etcdkey.bin
oc -n $INFRA create secret generic $HOSTED-etcd-encryption-key --type=Opaque --from-file=key=/tmp/etcdkey.bin
shred -u /tmp/etcdkey.bin   # key material, don't leave it in /tmp
oc -n $INFRA get secret $HOSTED-etcd-encryption-key -o jsonpath='{.data.key}' | wc -c   # 44 = base64 of 32 raw bytes, correct
```

Fixing the secret after the fact is safe — the apiserver re-reads it on its next
restart, so a CrashLoopBackOff pod recovers without a reinstall.

Note the render maps `--control-plane-availability-policy=SingleReplica` to
`spec.controllerAvailabilityPolicy`, **not** `controlPlaneAvailabilityPolicy`.
Grep for the right key when inspecting a render. Likewise `hcp create kubeconfig`
writes to **stdout**, not to a file:

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig
/root/hcp create kubeconfig --name=$HOSTED --namespace=$INFRA > /root/$HOSTED-kubeconfig
chmod 600 /root/$HOSTED-kubeconfig
```

Acceptance: 2 Ready nodes, `controlPlaneTopology=External`, hosted
ClusterVersion complete. Hosted CP namespace is `<infra>-<hosted>`
(e.g. `hc01-infra-hc01`), NOT `clusters-*`. The hosted API NodePort is *not* the
Ignition NodePort — read the real one from
`oc -n <infra>-<hosted> get svc kube-apiserver -o jsonpath='{.spec.ports[0].nodePort}'`.
The agents' `api-vip-connectivity-check` targets the separate
`ignition-server-proxy` NodePort, which is why the two differ and why the
ignition port in the assisted-service log is not the API port you use.

Pin worker NTP (InfraEnv `additionalNTPSources` from Phase 6 does not survive
to installed workers — observed 2026-10-09: hc01/hc02 shipped stock
public-pool chrony, all sources Reach 0, `Not synchronised`). Per hosted
cluster:

```bash
# [hypervisor] KUBECONFIG = mgmt kubeconfig; HIP from the Phase 6 InfraEnv step
CONF=$(printf 'server %s iburst\ndriftfile /var/lib/chrony/drift\nmakestep 1.0 3\nrtcsync\nkeyfile /etc/chrony.keys\nleapsectz right/UTC\nlogdir /var/log/chrony\n' "$HIP" | base64 -w0)
printf 'apiVersion: machineconfiguration.openshift.io/v1\nkind: MachineConfig\nmetadata:\n  labels:\n    machineconfiguration.openshift.io/role: worker\n  name: 99-worker-chrony\nspec:\n  config:\n    ignition: {version: 3.2.0}\n    storage:\n      files:\n      - contents:\n          source: "data:text/plain;charset=utf-8;base64,%s"\n        mode: 0644\n        overwrite: true\n        path: /etc/chrony.conf\n' "$CONF" > /tmp/99-worker-chrony.yaml
oc create configmap $HOSTED-chrony -n $INFRA --from-file=config=/tmp/99-worker-chrony.yaml
oc patch nodepool -n $INFRA $HOSTED --type=merge -p '{"spec":{"config":[{"name":"'$HOSTED-chrony'"}]}}'
```

Hosted clusters expose no MachineConfig/MCP API — NodePool `spec.config` is
the only path. Nodes cycle one at a time (~10 min for 2 workers); the
SingleReplica drain hang from the pitfalls table did not materialize on
hc01/hc02, but a node parked `Ready,SchedulingDisabled` past 15 min means it
did — recover per that row. Verify per worker (`KUBECONFIG` = hosted
kubeconfig): `oc debug node/<n> -- chroot /host chronyc tracking` shows
`Leap status: Normal` and `chronyc sources` selects `^* _gateway`. Pool-wide,
so future workers inherit it.

### Phase 10 (optional): operator for HCP validation

Validates the `koku-metrics-operator` end to end on the hosted cluster:
native arm64 build → air-gapped deploy → ROS CSV rows → downloadable package
→ recommendations via `robne`. Proven 2026-09-27 on `hc01`
(`pgarciaq-rosocp-superpowers-phase17`, image
`quay.io/pgarciaq/koku-metrics-operator:w0.1-20260927-0828`).

**Source + registry (decide with the human first).** Which branch to build,
which Quay repo to push to. Previous run: current feature branch (carries the
HCP-relevant operator work), private `quay.io/pgarciaq/koku-metrics-operator`,
tags unique per build (`w0.1-$(date -u +%Y%m%d-%H%M)`). The human runs
`podman login quay.io` on the box as root — robot tokens never transit chat,
issues, or files. Nothing past the build proceeds without it.

```bash
# [hypervisor] clone the agreed ref, native arm64 build (box is aarch64):
rm -rf /root/koku-metrics-operator
git clone --branch <branch> --depth 50 https://github.com/<fork>/koku-metrics-operator.git /root/koku-metrics-operator
cd /root/koku-metrics-operator && git rev-parse HEAD && git status --short  # record both; tree must be clean
TAG="w0.1-$(date -u +%Y%m%d-%H%M)"; IMG="quay.io/<repo>:$TAG"
podman build --platform=linux/arm64 -t $IMG -f Dockerfile .   # tmux; Go build takes ~10 min
podman image inspect $IMG --format '{{.Architecture}}'        # must print arm64
podman push $IMG   # needs the human's login; verify with podman login --get-login (prints the robot name, not the token)
```

**Deploy.** The Makefile assumes `docker`/`kustomize`; neither is needed.
Copy the repo's `config/` tree to the box, sed-replace the placeholder image,
apply with the *hosted* kubeconfig (`oc` 4.22 has builtin kustomize):

```bash
# [hypervisor] KUBECONFIG=/root/$HOSTED-kubeconfig
sed -i "s#image: controller:latest#image: $IMG#" /root/opcfg/manager/manager.yaml
oc apply -k /root/opcfg/default   # creates ns koku-metrics-operator, CRDs, RBAC, deployment
```

Air-gapped CR (shape: the repo's `config/samples/air-gapped.yaml`). The
controller floors `upload_cycle` at 60, so 60 is the fastest first package:

```yaml
# hc01-cr.yaml — apply into the operator namespace
apiVersion: costmanagement-metrics-cfg.openshift.io/v1beta1
kind: CostManagementMetricsConfig
metadata: {name: hc01-air-gapped, namespace: koku-metrics-operator}
spec:
  authentication: {}
  packaging: {max_reports_to_store: 30, max_size_MB: 100}
  prometheus_config: {}   # defaults to thanos-querier.openshift-monitoring.svc:9091
  source: {}
  upload: {upload_cycle: 60, upload_toggle: false}
```

Gate the CR on `prometheus_connected: true` and the topology detection
(`controlPlaneTopology: External`, `managedByHypershift: true`) before
proceeding — both are in `.status`.

**Hosted storage (the step this phase used to miss).** Agent-platform HCP
ships **zero StorageClasses**, and the operator's own docs list a default
StorageClass with dynamic provisioning as a deploy prerequisite. Without one,
the operator's 10Gi `koku-metrics-operator-data` PVC stays Pending forever
(`pod has unbound immediate PersistentVolumeClaims`) and nothing else in this
phase can proceed. Check first: `oc get sc` on the *hosted* cluster. If empty,
deploy `local-path-provisioner` there exactly as in Phase 3, except the
nodePath **must** live under `/var` (e.g. `/var/local-path-provisioner`):
`/` on RHCOS workers is a ~6 MB composefs, so the `/opt` default is unusable;
`/var` has the disk. Same privileged-SCC grant, same `chcon`, same default-SC
marking. The PVC binds within seconds of the provisioner going Ready.

**Labels (the skill's old instruction is half-stale).** "Label HCP namespaces,
auto-include does not exist" predates operator #405, which OR-matches the
hypershift hosted-control-plane label inside the ROS *queries* — but the
collection **gate still requires at least one namespace carrying the manual
`cost_management_optimizations=true` label**, else the CR reports
`No namespaces contain the cost_management_optimizations="true" label, so no
resource optimization metrics were collected` and no ROS CSVs are generated.
And the query exclusion `namespace!~'kube-.*|openshift|openshift-.*'` means
labeling `openshift-*` namespaces is pointless. Use a dedicated workload
namespace outside those patterns, with **Running** pods that declare
cpu/memory requests *and* limits (the ROS joins require
`phase='Running'`). A bare `ubi9/nginx-126` image exits immediately when run
directly — use a sleep workload:

```yaml
# ros-demo.yaml
apiVersion: v1
kind: Namespace
metadata: {name: ros-demo, labels: {cost_management_optimizations: "true"}}
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: ros-demo, namespace: ros-demo}
spec:
  replicas: 2
  selector: {matchLabels: {app: ros-demo}}
  template:
    metadata: {labels: {app: ros-demo}}
    spec:
      containers:
      - name: work
        image: registry.access.redhat.com/ubi9/ubi-minimal:latest
        command: ["sleep", "infinity"]
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits: {cpu: 500m, memory: 512Mi}
```

**Collection → packaging → download.** The operator backfills recent hours
(`collect_previous_data`), so labeled-namespace rows appear in the next hourly
CSV. Packaging follows `upload_cycle` (≤ ~1h at the 60 floor); watch
`.status.packaging.packaged_files` for a new `*-cost-mgmt.tar.gz`. The
operator image is distroless (no shell — `oc exec` on it fails), so inspect
and download through a helper pod mounting the same PVC:

```yaml
# volume-shell.yaml — ubi-minimal + `microdnf install tar` inside, mount at /data
apiVersion: v1
kind: Pod
metadata: {name: volume-shell, namespace: koku-metrics-operator}
spec:
  containers:
  - {name: shell, image: registry.access.redhat.com/ubi9/ubi-minimal:latest,
     command: ["sleep", "infinity"],
     volumeMounts: [{name: reports, mountPath: /data}]}
  volumes:
  - {name: reports, persistentVolumeClaim: {claimName: koku-metrics-operator-data}}
```

```bash
# [hypervisor] KUBECONFIG = hosted kubeconfig
oc -n koku-metrics-operator exec volume-shell -- ls -R /data                      # data/ holds live CSVs incl. ros-openshift-container-*.csv
oc -n koku-metrics-operator exec volume-shell -- microdnf install -y tar
oc -n koku-metrics-operator cp volume-shell:/data/upload/<pkg>.tar.gz /tmp/<pkg>.tar.gz
# then scp /tmp/<pkg>.tar.gz to the laptop. Bare helper pods die on node
# reboots — recreate as needed. MCE-console download link is a dead end
# (route DNS is explicit-only); the Quay robot login stays human-run.
```

**Validate end to end with `robne`** (repo root `make robne`, CGO_ENABLED=0):

```bash
ROBNE_NO_USER_CONFIG=1 ./bin/robne validate --input ./<pkg>.tar.gz --no-user-config    # expect: ok, container + namespace rows
ROBNE_NO_USER_CONFIG=1 ./bin/robne recommend --input ./ --plugins container --no-user-config --format json
# expect: recommendations for the labeled workload (cost + performance), 0 skipped
```

`--plugins namespace` returning 0 with 0 skipped is **expected** on a fresh
lab: the namespace short term needs 1 day minimum data. Only a namespace
result with *skipped* rows, or a container result with 0 recommendations,
indicates a real problem.

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
| `no matches for kind "OperatorGroup"` | CRD no longer serves `v1alpha1` — use `v1alpha2` (Subscription stays `v1alpha1`) |
| MCE CSV `Failed`: `UnsupportedOperatorGroup` | MCE 2.17 supports only `OwnNamespace`; `installModes` gets stripped by OLM 4.22 |
| `AgentServiceConfig` never goes Ready | It publishes no `Ready` condition — gate on `DeploymentsHealthy` + PVCs |
| `Waiting for Kube APIServer deployment`, `CrashLoopBackOff` | etcd encryption key holds 44 chars, not 32 raw bytes (Phase 9) |
| Machines `Pending` / `WaitingForNodeRef`, agents never install | Missing `*.apps.<hosted-domain>` wildcard record (Phase 8) |
| DNS records in `default.addnhosts` but guests NXDOMAIN | dnsmasq didn't re-read — `kill -HUP` the pid |
| `dig`/`nslookup`/`host` return empty | Not installed on a stock box; empty ≠ NXDOMAIN. Use `oc debug node/<n> -- chroot /host getent hosts` |
| `modify` fails: "couldn't locate a matching DNS HOST record" | No existing block for that IP — use `add-last` to create |
| `virsh net-update` on an IP with a block replaces all names | One `<host>` element per IP; re-send the complete hostname list |
| Hosted API/console unreachable | Apps pointed at VIP instead of worker (HostNetwork) |
| Confusion over two different NodePorts | `ignition-server-proxy` (agent check) ≠ `kube-apiserver` (your API) |
| `RenderDegraded: unterminated parameter sequence` | Unquoted ignition data URL (comma ends the value) |
| MCP stuck Updating, `FailedToDrain`, no retry | SingleReplica HCP PDBs block MCO drain (see NTP section) |
| Topology stays `{}` silently | RBAC applied to unprefixed ClusterRole — patch the live prefixed one, prove with `oc auth can-i` |
| PVCs Pending / `mkdir: Permission denied` | Missing `chcon -Rt container_file_t` on the local-path dir |
| `/` filesystem fills during assisted-service install | local-path `nodePath` defaults to `/opt` on `/` — relocate (Phase 3) |
| No ROS rows / empty collection | HCP namespaces lack the label; check CR status message first |
| Operator pod `Pending`: `unbound immediate PersistentVolumeClaims` | Hosted cluster has no StorageClass (Agent HCP ships none) — provision local-path on hosted first (Phase 10) |
| `local-path-provisioner` CrashLoop / no space on hosted workers | `/opt` default is unusable — RHCOS `/` is a ~6 MB composefs; nodePath must be under `/var` |
| `oc exec` on the operator pod fails (`sh` not found) | Distroless image — inspect/download via the volume-shell pod instead |
| `verify you have provided the correct credentials` on `podman push` | Human hasn't run `podman login quay.io` on the box, or robot lacks Write on the repo |
| `oc login ... -u kubeadmin` 401 on mgmt after touching the secret | Deleting `kube-system/kubeadmin` cascade-deletes User/Group/CRB and the operator won't recreate them — see runbook Known issue |
| Namespace plugin returns 0 recommendations, 0 skipped | Expected on a fresh lab — short term needs 1 day minimum data |
| `oc cp` fails in helper pod | ubi-minimal has no `tar` |
