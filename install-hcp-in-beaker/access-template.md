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
| MCE | {{MCE_CHANNEL}} (CSV `{{MCE_CSV}}`, **{{MCE_INSTALL_MODE}}** — console plugin not served), `hcp` CLI at `/root/hcp` ({{HCP_CLI_VERSION}}) |
| Storage | `{{STORAGECLASS}}` (default, `nodePath={{LPP_NODEPATH}}`) |
| NTP | {{NTP_NOTE}} |

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

## DNS (explicit records on all three layers — no real DNS)

| Name | IP | Why |
|------|----|-----|
| `api.{{MGMT_NAME}}…`, `*.apps.{{MGMT_NAME}}…`, assisted hostnames | {{API_VIP}} | mgmt serving endpoint |
| `api.{{HOSTED_NAME}}…`, `api-int.{{HOSTED_NAME}}…` | {{API_VIP}} | hosted API NodePort |
| **`*.apps.{{HOSTED_NAME}}…`** (wildcard — mandatory) plus `console/oauth/canary .apps.{{HOSTED_NAME}}…` | {{ROUTER_WORKER_IP}} | hosted HostNetwork router (DHCP — re-check on rebuild) |

The hosted `*.apps.{{BASE_DOMAIN}}` wildcard is what the assisted agent
validates; without it the workers never install. After any `virsh net-update`,
`kill -HUP` dnsmasq. Verify guest-side with
`oc debug node/<n> -- chroot /host getent hosts <name>` (`dig`/`nslookup`/`host`
are not installed on a stock box, and empty output from them means *not
installed*, not NXDOMAIN).

## Deviations from the skill (record what you hit)

Anything you had to change, with the symptom it caused. Future runs read this
section first — see the "Deviations" list in `SKILL.md` for the 4.22-era
entries, and add new ones here as you find them.

## Laptop access

\`\`\`bash
# /etc/hosts (mgmt + hosted entries per DNS table above)
sshuttle -r root@{{HYPERVISOR_FQDN}} 192.168.122.0/24
oc login https://api.{{MGMT_NAME}}.{{BASE_DOMAIN}}:6443 -u kubeadmin -p '{{MGMT_KUBEADMIN}}'
\`\`\`

## Metrics operator (authenticated OLM mode, if deployed)

The operator is installed independently in each target cluster. It runs on
that cluster's workers; it is not installed separately on each worker VM.

| Plane | Kubeconfig | Subscription / CSV | CR | Source | Last upload |
|-------|------------|--------------------|----|--------|-------------|
| Management | `/root/.kcli/clusters/{{MGMT_NAME}}/auth/kubeconfig` | `{{MGMT_KMO_SUBSCRIPTION}}` / `{{MGMT_KMO_CSV}}` | `{{MGMT_KMO_CR}}` | `{{MGMT_KMO_SOURCE}}` | `{{MGMT_KMO_UPLOAD}}` |
| Hosted | `/root/{{HOSTED_NAME}}-kubeconfig` | `{{HOSTED_KMO_SUBSCRIPTION}}` / `{{HOSTED_KMO_CSV}}` | `{{HOSTED_KMO_CR}}` | `{{HOSTED_KMO_SOURCE}}` | `{{HOSTED_KMO_UPLOAD}}` |

Namespace: `koku-metrics-operator`. Both CSVs must be `Succeeded` and own their
deployment. Each CR uses `sources_path: /api/cost-management/v1/`, a unique
source name, and the Keycloak `cost-management` realm. The auth Secret in each
cluster contains only `client_id` and `client_secret`; the client-credentials
service-account user has the `org-admin` realm role.

| Field | Value |
|-------|-------|
| Manager image tag | `{{CURRENT_TAG}}` |
| Manager image digest | `{{KMO_IMAGE_DIGEST}}` |
| Bundle image | `{{KMO_BUNDLE_IMAGE}}` |
| Operator commit | `{{KMO_SHA_FULL}}` |
| Auth Secret | `{{KMO_AUTH_SECRET}}` in each operator namespace |
| Token URL realm | `cost-management` |
| Sources path | `/api/cost-management/v1/` |
| Cost Management gateway | `{{COST_MANAGEMENT_GATEWAY}}` |
| RBAC mode | `rbac.enabled=false`, `ENHANCED_ORG_ADMIN=True` when RBAC is absent |

Verify each plane:

```bash
oc get subscription,csv -n koku-metrics-operator
oc get costmanagementmetricsconfig -n koku-metrics-operator {{MGMT_KMO_CR}} \
  -o jsonpath='commit={.status.operator_commit}{"\n"}prom={.status.prometheus.prometheus_connected}{"\n"}source={.status.source.source_defined}{"\n"}uploaded={.status.upload.last_successful_upload_time}{"\n"}status={.status.upload.last_upload_status}{"\n"}'
# Repeat with the hosted kubeconfig and {{HOSTED_KMO_CR}}.
```

## Metrics operator (air-gapped validation mode, if deployed)

This is mutually exclusive with authenticated OLM mode. It uses direct
kustomize deployment on the hosted cluster, empty authentication, and
`upload_toggle: false`; it validates collection/package contents locally only.

## Operator images

Robot `{{QUAY_ROBOT}}` (push on `{{QUAY_REPO}}`, private). Token ONLY in root's
`podman login` on the hypervisor. The manager image and OLM bundle/catalog use
unique tags (`{{TAG_SCHEME}}`) and the CSV records the manager image digest.

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
