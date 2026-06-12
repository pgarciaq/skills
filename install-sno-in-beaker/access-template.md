# ACCESS.md Template

Use this template when generating the ACCESS.md for a new SNO cluster.
Replace all `{{placeholders}}` with actual values.

```markdown
# {{CLUSTER_NAME}} — Single-Node OpenShift on {{HYPERVISOR_SHORT}}

## Cluster Overview

| Field | Value |
|-------|-------|
| Cluster Name | `{{CLUSTER_NAME}}` |
| Type | Single-Node OpenShift (SNO) |
| OpenShift Version | {{OCP_VERSION}} |
| Base Domain | `{{BASE_DOMAIN}}` |
| Node | `{{NODE_NAME}}` |
| Node IP | {{NODE_IP}} |
| Created | {{DATE}} |

## Hypervisor

| Field | Value |
|-------|-------|
| Host | `{{HYPERVISOR_FQDN}}` |
| OS | RHEL {{RHEL_VERSION}} |
| SSH | `ssh root@{{HYPERVISOR_FQDN}}` |

## Cluster Access

| Field | Value |
|-------|-------|
| API URL | `https://api.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}:6443` |
| Console URL | `https://console-openshift-console.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}` |
| Kubeadmin Password | `{{KUBEADMIN_PASSWORD}}` |
| Kubeconfig (on hypervisor) | `/root/.kcli/clusters/{{CLUSTER_NAME}}/auth/kubeconfig` |

### Login via oc CLI

From the hypervisor:

\`\`\`bash
export KUBECONFIG=/root/.kcli/clusters/{{CLUSTER_NAME}}/auth/kubeconfig
oc whoami
\`\`\`

Or login with credentials:

\`\`\`bash
oc login https://api.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}:6443 -u kubeadmin -p '{{KUBEADMIN_PASSWORD}}' --insecure-skip-tls-verify
\`\`\`

### Copy kubeconfig to your workstation

\`\`\`bash
scp root@{{HYPERVISOR_FQDN}}:/root/.kcli/clusters/{{CLUSTER_NAME}}/auth/kubeconfig ~/.kube/{{CLUSTER_NAME}}-kubeconfig
export KUBECONFIG=~/.kube/{{CLUSTER_NAME}}-kubeconfig
\`\`\`

## VM Resources

| Resource | Value |
|----------|-------|
| CPU | {{CPU}} cores |
| RAM | {{RAM}} GiB |
| System Disk | {{SYSTEM_DISK}} GiB (`/dev/vda`) |
| Data Disk | {{DATA_DISK}} GiB (`{{DATA_DISK_DEV}}`) |

## Storage

| Field | Value |
|-------|-------|
| LVMS Operator | `{{LVMS_CSV}}` |
| LVMCluster | `lvmcluster` (Ready) |
| Device Class | `vg1` → `{{DATA_DISK_DEV}}` ({{DATA_DISK}} GiB) |
| StorageClass | `lvms-vg1` (default) |
| Thin Pool | `thin-pool-1` (90%, overprovision ratio 10) |

## SSH Keys

| File | Location (on hypervisor) |
|------|--------------------------|
| Private key | `/root/.kcli/id_rsa` |
| Public key | `/root/.kcli/id_rsa.pub` |
| Pull secret | `/root/.kcli/openshift_pull.json` |

## kcli Management

\`\`\`bash
ssh root@{{HYPERVISOR_FQDN}}

kcli list cluster
kcli info cluster openshift {{CLUSTER_NAME}}
kcli stop vm {{CLUSTER_NAME}}-sno
kcli start vm {{CLUSTER_NAME}}-sno
kcli delete cluster {{CLUSTER_NAME}}   # destructive!
\`\`\`
```
