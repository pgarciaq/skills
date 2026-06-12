---
name: install-sno-in-beaker
description: >-
  Deploy a single-node OpenShift (SNO) cluster on a Red Hat Beaker lab
  hypervisor using kcli. Installs LVMS storage, fixes NTP, and produces an
  ACCESS.md with all connection details. Use when the user asks to create,
  install, or deploy an SNO or OpenShift cluster on a Beaker machine.
disable-model-invocation: true
---

# Install SNO in Beaker

Deploy a single-node OpenShift cluster on a Beaker hypervisor with kcli,
LVMS storage, and NTP synchronization.

## Parameters

| Parameter | Required | Default |
|-----------|----------|---------|
| Hypervisor SSH target | Yes | — |
| Cluster name | Yes | — |
| CPU cores | No | 30 |
| RAM (GiB) | No | 80 |
| System disk (GiB) | No | 300 |
| Data disk (GiB) | No | 400 |
| Base domain | No | `karmalabs.corp` |

## Prerequisites on Local Laptop

The pull secret and SSH keys live locally and must be copied to the Beaker
machine. Default location:

```
~/rh/kcli/
├── openshift_pull.json
├── id_rsa
└── id_rsa.pub
```

If files are not at the default location, ask the user. The skill copies them
to `/root/.kcli/` on the hypervisor.

## Workflow

All SSH commands use `-o StrictHostKeyChecking=no`. Request `full_network`
permissions for every Shell call.

### Phase 0: Prepare the Hypervisor

Fresh Beaker machines need EPEL, kcli, libvirt, and a storage pool before
they can create OpenShift clusters.

1. **SSH in and check machine state**

```bash
ssh -o StrictHostKeyChecking=no root@HYPERVISOR "
  cat /etc/redhat-release
  nproc && free -g && df -h /
  which kcli 2>/dev/null && kcli --version || echo 'kcli not installed'
"
```

2. **Install EPEL for RHEL 10**

```bash
ssh root@HYPERVISOR "
  dnf install -y \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
"
```

3. **Install kcli**

```bash
ssh root@HYPERVISOR "
  dnf -y copr enable karmab/kcli
  dnf -y install kcli
"
```

If `dnf copr` is not available, install the `dnf-plugins-core` package first:

```bash
ssh root@HYPERVISOR "dnf -y install dnf-plugins-core"
```

4. **Run kcli prerequisites** (installs libvirt, qemu-kvm, configures networking)

```bash
ssh root@HYPERVISOR "kcli create host kvm -P pool=/home/libvirt/images"
```

This enables and starts libvirtd, creates the default network, and sets up
the necessary permissions.

5. **Create the libvirt storage pool**

Use `/home/libvirt/images` (more space than `/var/lib/libvirt/images` on
Beaker machines where `/home` is the largest partition):

```bash
ssh root@HYPERVISOR "
  mkdir -p /home/libvirt/images
  kcli create pool -p /home/libvirt/images default
  setfacl -m u:qemu:rwx /home/libvirt/images
"
```

If the `default` pool already exists pointing elsewhere, delete and recreate:

```bash
ssh root@HYPERVISOR "
  kcli delete pool default -y 2>/dev/null
  kcli create pool -p /home/libvirt/images default
"
```

6. **Verify libvirt is working**

```bash
ssh root@HYPERVISOR "
  systemctl is-active libvirtd
  virsh pool-list
  virsh net-list
  kcli list pool
  kcli list network
"
```

### Phase 1: Copy Credentials to the Hypervisor

7. **Copy pull secret and SSH keys** (if not already on the hypervisor)

```bash
ssh root@HYPERVISOR "mkdir -p /root/.kcli"

# Check what exists
ssh root@HYPERVISOR "ls /root/.kcli/openshift_pull.json /root/.kcli/id_rsa 2>/dev/null"

# Copy missing files
scp ~/rh/kcli/openshift_pull.json root@HYPERVISOR:/root/.kcli/openshift_pull.json
scp ~/rh/kcli/id_rsa root@HYPERVISOR:/root/.kcli/id_rsa
scp ~/rh/kcli/id_rsa.pub root@HYPERVISOR:/root/.kcli/id_rsa.pub
ssh root@HYPERVISOR "chmod 600 /root/.kcli/id_rsa"
```

### Phase 2: Create the SNO Cluster

8. **Create the cluster with kcli**

```bash
ssh root@HYPERVISOR "
  kcli create cluster openshift \
    -P sno=true \
    -P version=stable \
    -P cpus=30 \
    -P memory=81920 \
    -P disk_size=300 \
    -P extra_disks=[400] \
    -P pull_secret=/root/.kcli/openshift_pull.json \
    CLUSTER_NAME
"
```

This takes 30-60 minutes. Use `block_until_ms: 0` and monitor with
`notify_on_output` matching `cluster.*ready|installed|error|failed`.

9. **Verify the cluster is up**

```bash
ssh root@HYPERVISOR "
  export KUBECONFIG=/root/.kcli/clusters/CLUSTER_NAME/auth/kubeconfig
  oc get clusterversion
  oc get nodes
  oc get co
"
```

If `oc` is not found, install it:

```bash
ssh root@HYPERVISOR "kcli download oc -P version=stable"
# or: dnf install openshift-clients
# or: download from mirror.openshift.com
```

### Phase 3: Install LVMS Operator

10. **Create Namespace, OperatorGroup, and Subscription**

```bash
ssh root@HYPERVISOR "
  export KUBECONFIG=/root/.kcli/clusters/CLUSTER_NAME/auth/kubeconfig

  oc create namespace openshift-storage --dry-run=client -o yaml | oc apply -f -

  oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
  - openshift-storage
EOF

  oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: lvms-operator
  namespace: openshift-storage
spec:
  channel: stable-VERSION_XY
  installPlanApproval: Automatic
  name: lvms-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
"
```

**Channel pitfall:** The generic `stable` channel does NOT work on OCP 4.17+.
Detect the OCP minor version and use `stable-X.Y`:

```bash
VER=$(oc get clusterversion -o jsonpath='{.items[0].status.desired.version}')
CHANNEL="stable-${VER%.*}"   # e.g. "stable-4.21"
```

11. **Wait for CSV to succeed** (poll every 15s, up to 5 min)

```bash
for i in $(seq 1 20); do
  phase=$(oc get csv -n openshift-storage -o jsonpath='{.items[0].status.phase}' 2>/dev/null)
  echo "Attempt $i: $phase"
  [ "$phase" = "Succeeded" ] && break
  sleep 15
done
```

### Phase 4: Create LVMCluster

12. **Find the data disk**

```bash
oc debug node/NODE_NAME -- chroot /host lsblk -d -o NAME,SIZE,TYPE
```

The data disk (400G) is typically `/dev/vdb`.

13. **Create the LVMCluster CR**

```bash
oc apply -f - <<'EOF'
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
    - name: vg1
      default: true
      deviceSelector:
        paths:
        - /dev/vdb
      thinPoolConfig:
        name: thin-pool-1
        sizePercent: 90
        overprovisionRatio: 10
EOF
```

14. **Wait for LVMCluster to be Ready** and verify StorageClass:

```bash
oc get lvmcluster -n openshift-storage
oc get sc   # expect: lvms-vg1 (default)
```

### Phase 5: Fix NTP

15. **Enable the hypervisor as an NTP server**

```bash
ssh root@HYPERVISOR "
  grep -q '^allow 192.168.122' /etc/chrony.conf || \
    echo 'allow 192.168.122.0/24' >> /etc/chrony.conf
  systemctl restart chronyd
"
```

16. **Apply chrony MachineConfig to the SNO node**

Find the hypervisor's IP on virbr0 (typically `192.168.122.1`):

```bash
HYPERVISOR_LIBVIRT_IP=$(ssh root@HYPERVISOR "ip -4 addr show virbr0 | grep -oP '(?<=inet )[\d.]+'")
```

Create the MachineConfig:

```bash
CHRONY_CONF=$(echo "server ${HYPERVISOR_LIBVIRT_IP} iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony" | base64 -w0)

oc apply -f - <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-chrony
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${CHRONY_CONF}
        mode: 0644
        overwrite: true
        path: /etc/chrony.conf
EOF
```

17. **Wait for MCP to finish** (node reboots, ~5-10 min)

```bash
for i in $(seq 1 30); do
  updated=$(oc get mcp master -o jsonpath='{.status.conditions[?(@.type=="Updated")].status}')
  updating=$(oc get mcp master -o jsonpath='{.status.conditions[?(@.type=="Updating")].status}')
  echo "Attempt $i: Updated=$updated Updating=$updating"
  [ "$updated" = "True" ] && [ "$updating" = "False" ] && break
  sleep 20
done
```

18. **Verify NTP**

```bash
oc debug node/NODE_NAME -- chroot /host chronyc sources
# Expect: ^* _gateway (or the hypervisor IP) as selected source
```

### Phase 6: Create ACCESS.md

19. **Create a local ACCESS.md** at `~/rh/kcli/CLUSTER_NAME-HYPERVISOR_SHORT/ACCESS.md`

Derive `HYPERVISOR_SHORT` from the hostname (e.g., `dell-r640-041` from
`dell-r640-041.bkr.lab.eng.rdu2.dc.redhat.com`).

Include: cluster name, OCP version, API URL, console URL, kubeadmin password,
kubeconfig path, hypervisor SSH, node IP, VM resources, LVMS/StorageClass info,
SSH keys/pull secret locations, kcli management commands. See [access-template.md](access-template.md).

### Phase 7: Final Verification

20. **Report all of these to the user:**

- Cluster version
- Node status
- Degraded cluster operators (if any)
- LVMS CSV status
- LVMCluster status and StorageClass
- NTP sync status
- Console URL
- API URL
- Kubeadmin password
- Kubeconfig path

## Pitfalls

| Issue | Fix |
|-------|-----|
| `dnf copr` command not found | `dnf -y install dnf-plugins-core` first |
| `default` pool points to `/var/lib/libvirt/images` (small partition) | Delete and recreate: `kcli delete pool default -y && kcli create pool -p /home/libvirt/images default` |
| libvirtd not running after kcli install | `kcli create host kvm` handles this; verify with `systemctl is-active libvirtd` |
| `/home/libvirt/images` permission denied by qemu | `setfacl -m u:qemu:rwx /home/libvirt/images` |
| LVMS `stable` channel → `ConstraintsNotSatisfiable` | Use `stable-X.Y` matching OCP minor version |
| `oc` not installed on hypervisor | `kcli download oc` or `dnf install openshift-clients` |
| Hypervisor chrony `allow` commented out | Add `allow 192.168.122.0/24` and restart chronyd |
| MachineConfig triggers node reboot | Expected; wait for MCP Updated=True (~5-10 min) |
| Console unreachable from laptop | Need VPN + `ip route add 192.168.122.0/24 via HYPERVISOR_IP` + DNS resolution |
| `karmalabs.corp` DNS not resolving | kcli sets up dnsmasq; from laptop, either configure DNS forwarding or add `/etc/hosts` entries |
