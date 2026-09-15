# ACCESS.md Template (HCP lab)

Use this template when generating the ACCESS.md for a new HCP lab
(management + hosted cluster). Replace all `{{placeholders}}` with actual
values. Passwords and tokens live here and NOWHERE else (never in chat,
issues, or repos).

```markdown
# {{HOSTED_NAME}} — HCP lab on {{HYPERVISOR_SHORT}} ({{ARCH}})

Created {{DATE}}. Management `{{MGMT_NAME}}` ({{MGMT_VERSION}} compact) +
hosted `{{HOSTED_NAME}}` ({{HOSTED_VERSION}}, {{CP_POLICY}} CP, {{WORKERS}}
workers) on the Agent platform. {{SUPPORT_NOTE}}

## Hypervisor

| Field | Value |
|-------|-------|
| Host | `{{HYPERVISOR_FQDN}}` |
| Arch | {{ARCH}} |
| OS | RHEL {{RHEL_VERSION}} |
| SSH | `ssh root@{{HYPERVISOR_FQDN}}` |
| kcli | {{KCLI_VERSION}} |
| Pool | `default` → `{{POOL_PATH}}` |

## Management cluster `{{MGMT_NAME}}`

| Field | Value |
|-------|-------|
| Version | {{MGMT_VERSION}} |
| Topology | Compact {{MGMT_NODES}} nodes (control-plane,master,worker) |
| Nodes | {{MGMT_NODE_IPS}} ({{MGMT_CPU}} vCPU / {{MGMT_RAM}} GiB / {{MGMT_DISK}} GiB each) |
| API + ingress | keepalived VIP **{{API_VIP}}** |
| API URL | `https://api.{{MGMT_NAME}}.{{BASE_DOMAIN}}:6443` |
| Console | `https://console-openshift-console.apps.{{MGMT_NAME}}.{{BASE_DOMAIN}}` |
| Kubeadmin | `{{MGMT_KUBEADMIN}}` |
| Kubeconfig (box) | `/root/.kcli/clusters/{{MGMT_NAME}}/auth/kubeconfig` |
| MCE | {{MCE_CHANNEL}} (`hcp` CLI at `/root/hcp`) |
| Storage | `{{STORAGECLASS}}` (default) |

## Hosted cluster `{{HOSTED_NAME}}`

| Field | Value |
|-------|-------|
| Version | {{HOSTED_VERSION}} (`-multi` payload), `controlPlaneTopology=External` |
| Control plane | {{CP_POLICY}}, namespace **`{{HCP_NAMESPACE}}`** on mgmt, etcd on `{{STORAGECLASS}}` |
| Workers | {{WORKER_VMS}} ({{WORKER_CPU}} vCPU / {{WORKER_RAM}} GiB / {{WORKER_DISK}} GiB each) |
| API URL | `https://api.{{HOSTED_NAME}}.{{BASE_DOMAIN}}:{{HOSTED_API_NODEPORT}}` (NodePort on VIP; discover via `oc -n {{INFRA_NAMESPACE}}-{{HOSTED_NAME}} get svc kube-apiserver -o jsonpath='{.spec.ports[0].nodePort}' on mgmt) |
| Console | `https://console-openshift-console.apps.{{HOSTED_NAME}}.{{BASE_DOMAIN}}` → worker {{ROUTER_WORKER_IP}}:443 (HostNetwork) |
| Kubeadmin | `{{HOSTED_KUBEADMIN}}` |
| Kubeconfig (box) | `/root/{{HOSTED_NAME}}-kubeconfig` |
| Infra namespace | `{{INFRA_NAMESPACE}}` |
| Secrets (mgmt ns) | `{{HOSTED_NAME}}-pull-secret`, `{{HOSTED_NAME}}-ssh-key`, `{{HOSTED_NAME}}-etcd-encryption-key` |

## DNS (explicit records on all three layers — no wildcards)

| Name | IP | Why |
|------|----|-----|
| `api.{{MGMT_NAME}}…`, `*.apps.{{MGMT_NAME}}…`, assisted hostnames | {{API_VIP}} | mgmt serving endpoint |
| `api.{{HOSTED_NAME}}…`, `api-int.{{HOSTED_NAME}}…` | {{API_VIP}} | hosted API NodePort |
| `console/oauth/canary .apps.{{HOSTED_NAME}}…` | {{ROUTER_WORKER_IP}} | hosted HostNetwork router (DHCP — re-check on rebuild) |

## Laptop access

\`\`\`bash
# /etc/hosts (mgmt + hosted entries per DNS table above)
sshuttle -r root@{{HYPERVISOR_FQDN}} 192.168.122.0/24
oc login https://api.{{MGMT_NAME}}.{{BASE_DOMAIN}}:6443 -u kubeadmin -p '{{MGMT_KUBEADMIN}}'
\`\`\`

## Operator images (if deployed)

Robot `{{QUAY_ROBOT}}` (push on `{{QUAY_REPO}}`, private). Token ONLY in
root's `podman login` on the hypervisor. Tags unique per build
(`{{TAG_SCHEME}}`). Current: `{{CURRENT_TAG}}` on both planes.

## kcli management (from hypervisor)

\`\`\`bash
kcli list cluster
kcli info kube {{MGMT_NAME}}
virsh list --all
kcli delete kube -y {{MGMT_NAME}}            # DESTROYS management (and hosted CP with it)
kcli delete vm -y {{WORKER_VM_NAMES}}
oc -n {{INFRA_NAMESPACE}} delete hostedcluster {{HOSTED_NAME}}   # removes hosted only
\`\`\`
```
