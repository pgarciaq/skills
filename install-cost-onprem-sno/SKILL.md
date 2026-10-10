---
name: install-cost-onprem-sno
description: >-
  Use when the user asks to install, deploy, or redeploy Cost Management
  on-prem on an existing Single-Node OpenShift cluster, including SNO, Apollo,
  or Beaker OpenShift, using local branches and cluster-local images.
disable-model-invocation: true
---

# Install Cost Management On-Prem on SNO

Deploy Cost Management on-prem onto an existing SNO cluster using **images
built from the user's current local branches** — never stock amd64 chart
defaults on aarch64. Also deploy **koku-metrics-operator** from the current
branch so the cluster collects and uploads OCP/ROS metrics into the stack.

## Parameters

| Parameter | Required | Default |
|-----------|----------|---------|
| Hypervisor SSH target | Yes | — |
| Cluster name | Yes | `sno` |
| Workspace root | No | `~/dev/koku` |
| Namespace | No | `cost-onprem` |
| Image tag | No | `branch-$(date -u +%Y%m%d%H%M)` |
| Enable Kruize | No | `false` (native ROS engine) |
| Install metrics operator through OLM | No | `true` |
| Values file | No | `sno-arm64-values.yaml` (arm64) or `openshift-values.yaml` (amd64) |
| Run chart tests after deploy | No | `false` |

Ask for any missing required parameter. Infer cluster name / hypervisor from
`~/rh/kcli/*/ACCESS.md` when the user points at an existing SNO.

## Hard Rules

1. **Check architecture first** — build host and cluster nodes must match.
   ```bash
   uname -m
   oc get nodes -o custom-columns=NAME:.metadata.name,ARCH:.status.nodeInfo.architecture
   ```
   - Cluster `arm64` / `aarch64` → build **natively on the aarch64 hypervisor** (no `--platform`).
   - Cluster `amd64` → build natively on x86_64 (e.g. UXSNO / Dell R640).
2. **Build images from current branches BEFORE Helm install.** Do not install
   chart defaults then patch — on aarch64 that pulls amd64 images and fails.
3. **Unique image tags every deploy** (`IfNotPresent` otherwise keeps stale images).
4. **SSH:** `-F /dev/null -o StrictHostKeyChecking=no` (local
   `20-systemd-ssh-proxy.conf` breaks sshuttle/ssh on some laptops). Request
   `full_network` / `all` for every Shell call that hits the hypervisor or cluster.
5. For aarch64 pitfalls, read [reference.md](reference.md) before deploying.
6. **Metrics operator: upstream OLM, not an unmanaged kustomize deployment.**
   Build the current-branch operator image and bundle/catalog, install or
   update the Subscription, and wait for CSV `Succeeded`. Never patch a CSV's
   binary path or run a second kustomize-managed operator beside the OLM one.
   Preserve any existing CR, auth Secret, and PVC before an operator migration.
7. **Sources API path is the compatibility API root.** Set
   `sources_path: /api/cost-management/v1/` (including the trailing slash).
   The operator appends `source_types`, `application_types`, and `sources` to
   this root. Do not use `/api/cost-management/v1/sources/`; that makes
   `source_types` resolve to the protected Sources view and returns 403.
8. **Prove auth at each boundary.** A valid Keycloak token is not enough:
   verify the token's realm and `org-admin` role, the gateway's JWT-auth log,
   Koku's parsed identity log, and 200 responses from the compatibility
   endpoints before diagnosing upload failures.

## Repos (current branches)

From `$WORKSPACE` (default `~/dev/koku`), use whatever branch is checked out:

| Repo | Image(s) | Build context |
|------|----------|---------------|
| `koku` | `koku` | `Dockerfile` |
| `ros-ocp-backend` | `ros-ocp-backend` | `Dockerfile` |
| `insights-ingress-go` | `insights-ingress-go` | `Dockerfile` |
| `koku-ui` | `koku-ui` | `apps/koku-ui-onprem/Containerfile` |
| `cost-onprem-chart` | chart + values | Helm only |
| `koku-metrics-operator` | `koku-metrics-operator` | `Dockerfile` (or `make docker-build`) |
| `autotune` | `autotune` / `kruize` | `Dockerfile.autotune` (only if Kruize enabled) |

Also need registry tags for:

| Image | Source |
|-------|--------|
| `postgresql:16` | `docker.io/library/postgres:16` (multi-arch pull + retag) |
| `valkey:8` | build from chart/scripts or multi-arch Valkey image (see values) |

Record each repo's `git branch --show-current` and short SHA in the final report.

## Workflow

Copy this checklist and track progress:

```
Task Progress:
- [ ] Phase 0: Verify cluster + arch + network
- [ ] Phase 1: Detect local branches; sync repos to build host
- [ ] Phase 2: Build + push images (unique tag)
- [ ] Phase 3: Update values file with image tags
- [ ] Phase 4: Deploy infra (RHBK, Kafka, S3) + Helm chart
- [ ] Phase 5: Post-install fixes + smoke check
- [ ] Phase 6: Install/update OLM-managed koku-metrics-operator from branch
- [ ] Phase 7: Update ACCESS.md; report credentials
```

### Phase 0: Verify Cluster + Architecture

```bash
ssh -F /dev/null -o StrictHostKeyChecking=no root@HYPERVISOR "
  export KUBECONFIG=/root/.kcli/clusters/CLUSTER/auth/kubeconfig
  oc whoami
  oc get nodes -o custom-columns=NAME:.metadata.name,ARCH:.status.nodeInfo.architecture,STATUS:.status.conditions[-1].type
  oc get sc
  uname -m
"
```

Requirements before continuing:

- Node Ready; StorageClass present (`lvms-vg1` on Apollo aarch64)
- Build host arch matches cluster arch
- Tools on hypervisor: `podman`, `oc`, `helm`, `rsync`, `jq` (`dnf install -y` if missing)

Ensure registry default route:

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge \
  -p '{"spec":{"defaultRoute":true}}'
```

Add cluster API + `*.apps.CLUSTER.karmalabs.corp` routes to hypervisor `/etc/hosts`
(node IP). Include at least: API, console, oauth, registry route, keycloak,
gateway, masu, UI. See [reference.md](reference.md).

Laptop access: `sshuttle -r root@HYPERVISOR 192.168.122.0/24`.

### Phase 1: Sync Current Branches to Build Host

On the laptop, capture branches:

```bash
cd "$WORKSPACE"
for r in koku koku-ui ros-ocp-backend cost-onprem-chart insights-ingress-go koku-metrics-operator; do
  echo "$r $(git -C "$r" branch --show-current) $(git -C "$r" rev-parse --short HEAD)"
done
```

Rsync to hypervisor `/root/<repo>` (exclude bulky/irrelevant trees):

```bash
SSH='ssh -F /dev/null -o StrictHostKeyChecking=no'
RSYNC="rsync -a --delete -e '$SSH' --exclude=.git --exclude=node_modules --exclude=.tox --exclude=__pycache__"
# autotune: --exclude='/target' (leading slash!) — bare 'target' breaks Java sources
# metrics-operator: keep .git so make/deploy can report operator_commit / GIT_COMMIT
RSYNC_KMO="rsync -a --delete -e '$SSH' --exclude=node_modules --exclude=.tox --exclude=__pycache__ --exclude=bin"
$RSYNC "$WORKSPACE/koku/" root@HYPERVISOR:/root/koku/
$RSYNC "$WORKSPACE/ros-ocp-backend/" root@HYPERVISOR:/root/ros-ocp-backend/
$RSYNC "$WORKSPACE/insights-ingress-go/" root@HYPERVISOR:/root/insights-ingress-go/
$RSYNC "$WORKSPACE/koku-ui/" root@HYPERVISOR:/root/koku-ui/
$RSYNC "$WORKSPACE/cost-onprem-chart/" root@HYPERVISOR:/root/cost-onprem-chart/
$RSYNC_KMO "$WORKSPACE/koku-metrics-operator/" root@HYPERVISOR:/root/koku-metrics-operator/
```

### Phase 2: Build + Push Images

Use one unique tag for app images:

```bash
TAG="branch-$(date -u +%Y%m%d%H%M)"
```

**Create namespace + image-pusher SA** (kubeconfig cert auth cannot login to registry):

```bash
oc new-project cost-onprem 2>/dev/null || true
oc create serviceaccount image-pusher -n cost-onprem 2>/dev/null || true
oc adm policy add-cluster-role-to-user registry-editor -z image-pusher -n cost-onprem
TOKEN=$(oc create token image-pusher -n cost-onprem --duration=1h)
REGISTRY_EXT=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
podman login "$REGISTRY_EXT" -u image-pusher -p "$TOKEN" --tls-verify=false
```

**Build + push** (native; no `--platform` when arches match):

```bash
build_push() {
  local name=$1 context=$2 dockerfile=${3:-Dockerfile}
  podman build -t "${name}:${TAG}" -f "${context}/${dockerfile}" "${context}"
  podman tag "${name}:${TAG}" "${REGISTRY_EXT}/cost-onprem/${name}:${TAG}"
  podman push --tls-verify=false "${REGISTRY_EXT}/cost-onprem/${name}:${TAG}" \
    || skopeo copy --dest-tls-verify=false --dest-creds "image-pusher:${TOKEN}" \
         "containers-storage:localhost/${name}:${TAG}" \
         "docker://${REGISTRY_EXT}/cost-onprem/${name}:${TAG}"
}

build_push koku /root/koku
build_push ros-ocp-backend /root/ros-ocp-backend
build_push insights-ingress-go /root/insights-ingress-go
build_push koku-ui /root/koku-ui apps/koku-ui-onprem/Containerfile
build_push koku-metrics-operator /root/koku-metrics-operator

# Multi-arch base images (retag into internal registry)
podman pull docker.io/library/postgres:16
podman tag docker.io/library/postgres:16 "${REGISTRY_EXT}/cost-onprem/postgresql:16"
podman push --tls-verify=false "${REGISTRY_EXT}/cost-onprem/postgresql:16"

# Valkey: pull multi-arch or build per chart docs; tag cost-onprem/valkey:8
```

If Kruize enabled: `build_push autotune /root/autotune Dockerfile.autotune` and
tag as the chart expects (`kruize` / values image name).

Internal pull reference used in values:

`image-registry.openshift-image-registry.svc:5000/cost-onprem/<name>:<tag>`

Metrics operator uses the same tag under `cost-onprem/koku-metrics-operator:${TAG}`
(or a dedicated project if preferred — grant `system:image-puller` to the
operator SA for that namespace).

UI container name in the deployment is `app` (not `ui` / `nginx`).

### Phase 3: Point Values at Built Tags

Edit `/root/cost-onprem-chart/sno-arm64-values.yaml` (aarch64) — or the amd64
values file — so **all** custom images use `${TAG}` (and postgresql `16`,
valkey `8`). Keep:

- `rbac.enabled: false` + `ENHANCED_ORG_ADMIN: "True"` on aarch64 (insights-rbac is amd64-only)
- `global.storageClass: lvms-vg1` (or cluster default)
- `objectStorage.endpoint: minio.cost-onprem.svc.cluster.local` (no ODF)
- `kruize.enabled: false` unless user requested Kruize
- `global.pullPolicy` / image `pullPolicy: IfNotPresent`

### Phase 4: Deploy Infra + Helm

Prefer the chart deploy script from the synced chart repo **on a host with
`oc` access** (hypervisor is fine):

```bash
cd /root/cost-onprem-chart

# Storage credentials secret must exist before install-helm-chart.sh
oc create secret generic cost-onprem-storage-credentials -n cost-onprem \
  --from-literal=access-key=minioadmin \
  --from-literal=secret-key=minioadmin \
  --dry-run=client -o yaml | oc apply -f -

# Absolute VALUES_FILE path is required
OPENSHIFT_VALUES_FILE=sno-arm64-values.yaml \
VALUES_FILE=/root/cost-onprem-chart/sno-arm64-values.yaml \
  ./scripts/deploy-test-cost-onprem.sh \
    --namespace cost-onprem \
    --verbose \
    --skip-chart-tests
```

Ensure the script path deploys RHBK + Kafka (+ MinIO/S4 as required for the
cluster). If using a split flow:

1. `./scripts/deploy-rhbk.sh --namespace keycloak`
2. Kafka / AMQ Streams
3. MinIO / S4 with **exact** buckets: `insights-upload-perma`, `koku-bucket`, `ros-data`
4. `VALUES_FILE=/root/cost-onprem-chart/sno-arm64-values.yaml ./scripts/install-helm-chart.sh --namespace cost-onprem`

**Keycloak (RHBK v26+):**

- Hostname fields must be full `https://...` URLs
- Admin user from secret (`temp-admin`), not hardcoded `admin`
- Enable `ADMIN_EDIT` unmanaged attribute policy before setting custom attrs
- Test user `org_id` must be bare `"1234567"` (never `"org1234567"`)

### Phase 5: Post-Install Fixes + Smoke

1. Wait for pods: `oc get pods -n cost-onprem`
2. After first tenant / identity use: restart Koku API + flush Valkey
   ```bash
   oc rollout restart deploy/cost-onprem-koku-api -n cost-onprem
   oc exec deploy/cost-onprem-valkey -n cost-onprem -- redis-cli FLUSHALL
   ```
3. Smoke:
   ```bash
   curl -sk https://cost-onprem-gateway-cost-onprem.apps.CLUSTER.karmalabs.corp/api/cost-management/v1/status/
   ```

### Phase 6: Install OLM-Managed koku-metrics-operator From Branch

Skip only if the user set **Install metrics operator through OLM = false**. This
is **not** part of the Helm chart. Build the upstream operator from the current
branch, publish its image and OLM bundle/catalog, then install it through OLM
after cost-onprem is up.

#### 6a. Preserve existing operator state

Before changing the operator, save the existing CR, auth Secret, and PVC. Do
not delete an existing OLM Subscription/CSV merely to replace the image; update
the catalog or Subscription and let OLM roll the Deployment. If a previous
kustomize deployment exists, stop it only after confirming the OLM deployment
will own the CR and there will be exactly one active reconciler.

```bash
mkdir -p /tmp/kmo-migrate
oc get costmanagementmetricsconfig -n koku-metrics-operator -o yaml \
  > /tmp/kmo-migrate/cr.yaml 2>/dev/null || true
oc get secret -n koku-metrics-operator cost-mgmt-operator-keycloak -o yaml \
  > /tmp/kmo-migrate/auth-secret.yaml 2>/dev/null || true
oc get pvc -n koku-metrics-operator koku-metrics-operator-data -o yaml \
  > /tmp/kmo-migrate/pvc.yaml 2>/dev/null || true
```

#### 6b. Build and install the upstream OLM bundle

```bash
export KUBECONFIG=/root/.kcli/clusters/CLUSTER/auth/kubeconfig
cd /root/koku-metrics-operator

IMG_INT="image-registry.openshift-image-registry.svc:5000/cost-onprem/koku-metrics-operator:${TAG}"
# Build/push IMG_INT, resolve its immutable digest, then use the repository's
# `bundle`, `bundle-build`, `bundle-push`, and `bundle-deploy` targets (or the
# equivalent catalog workflow) from this checkout. Install/update the
# CatalogSource and Subscription in koku-metrics-operator; do not hand-edit the
# CSV. The generated CSV must reference IMG_INT by digest, not a mutable tag.

oc get subscription,csv -n koku-metrics-operator

oc adm policy add-cluster-role-to-user cluster-monitoring-view \
  -z koku-metrics-controller-manager -n koku-metrics-operator
# If image lives in another project, grant pull:
oc adm policy add-role-to-user system:image-puller \
  system:serviceaccount:koku-metrics-operator:koku-metrics-controller-manager \
  -n cost-onprem

oc rollout status deployment/koku-metrics-operator -n koku-metrics-operator --timeout=180s
oc get deployment koku-metrics-operator -n koku-metrics-operator -o json \
  | jq -e '[.metadata.ownerReferences[]? | select(.kind == "ClusterServiceVersion")] | length == 1'
```

If the pod sits on leader election after an OLM rollout, inspect the lease and
delete only the stale lease identified for this installation; do not assume a
fixed lease name:

```bash
oc get lease -n koku-metrics-operator
```

#### 6c. Auth secret + CostManagementMetricsConfig

Create/copy the operator auth Secret into `koku-metrics-operator` with exactly
the `client_id` and `client_secret` keys for the Keycloak client
`cost-management-operator` (or the cluster’s existing operator client). Keep
the Secret name used by the CR; the upstream operator does not read arbitrary
environment variables.

**Required on aarch64 (`rbac.enabled: false`):** grant the Keycloak realm
role `org-admin` to the operator client’s **service account** user
(`service-account-cost-management-operator`). The gateway sets
`X-Rh-Identity.is_org_admin` from that role; with `ENHANCED_ORG_ADMIN=True`,
Koku then skips calling `cost-onprem-rbac-api` (which is not deployed).
Without `org-admin`, protected Sources operations can fail with **424 Rbac
unavailable** when RBAC is disabled. A 403 from `source_types` usually means
the operator is using the wrong path, not that the role is missing.

```bash
# After Keycloak realm exists — assign org-admin to operator SA
KC=https://keycloak-keycloak.apps.CLUSTER.karmalabs.corp
# ... get admin token, CLIENT_UUID, SA_ID for cost-management-operator ...
curl -sk -X POST -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" \
  "$KC/admin/realms/cost-management/users/$SA_ID/role-mappings/realm" \
  -d "[$(curl -sk -H \"Authorization: Bearer $ADMIN_TOKEN\" \
       $KC/admin/realms/cost-management/roles/org-admin)]"
# Confirm JWT realm_access.roles contains org-admin
```

On aarch64, do **not** use the Quay `insights-rbac` image (amd64-only).
Build from source with
`cost-onprem-chart/containers/insights-rbac.Dockerfile.ubi9` on the aarch64
hypervisor, push to the SNO registry, then set `rbac.enabled: true` and
`ENHANCED_ORG_ADMIN: "False"`. Upstream `Dockerfile` (hi/core-runtime + full
`/usr/lib64` copy) produces a broken arm64 image (bash stack smash).

Apply a CR (migrate the saved one, or create fresh). **Must** use realm
`cost-management` (not `kubernetes`) and the compatibility API root below:

```yaml
apiVersion: costmanagement-metrics-cfg.openshift.io/v1beta1
kind: CostManagementMetricsConfig
metadata:
  name: costmanagementmetricscfg
  namespace: koku-metrics-operator
spec:
  api_url: https://cost-onprem-gateway-cost-onprem.apps.CLUSTER.karmalabs.corp
  authentication:
    type: service-account
    secret_name: cost-mgmt-operator-keycloak
    token_url: https://keycloak-keycloak.apps.CLUSTER.karmalabs.corp/realms/cost-management/protocol/openid-connect/token
  packaging:
    max_reports_to_store: 30
    max_size_MB: 100
  prometheus_config:
    collect_previous_data: true
    context_timeout: 120
    disable_metrics_collection_cost_management: false
    disable_metrics_collection_resource_optimization: false
    service_address: https://thanos-querier.openshift-monitoring.svc:9091
    skip_tls_verification: false
  source:
    create_source: true
    # Use a short interval during bootstrap so a failed first check does not
    # leave reports waiting for a day. Restore the normal interval after the
    # source is defined.
    check_cycle: 1
    sources_path: /api/cost-management/v1/
  upload:
    ingress_path: /api/ingress/v1/upload
    upload_cycle: 360
    upload_toggle: true
    validate_cert: false
```

#### 6d. Verify authentication and source routing

```bash
EXPECTED=$(git -C /root/koku-metrics-operator rev-parse HEAD)
oc get pods -n koku-metrics-operator
oc get costmanagementmetricsconfig -n koku-metrics-operator \
  costmanagementmetricscfg \
  -o jsonpath='commit={.status.operator_commit}{"\n"}prom={.status.prometheus.prometheus_connected}{"\n"}auth={.status.authentication.credentials_found}{"\n"}packaged={.status.packaging.number_reports_stored}{"\n"}'
# Expect: operator_commit == $EXPECTED, prometheus_connected=true, credentials_found=true

API=https://cost-onprem-gateway-cost-onprem.apps.CLUSTER.karmalabs.corp/api/cost-management/v1
# Obtain a client-credentials token without printing it. Use the same Secret
# and token_url referenced by the CR.
AUTH_SECRET=cost-mgmt-operator-keycloak
KC=https://keycloak-keycloak.apps.CLUSTER.karmalabs.corp
CLIENT_ID=$(oc get secret "$AUTH_SECRET" -n koku-metrics-operator \
  -o jsonpath='{.data.client_id}' | base64 -d)
CLIENT_SECRET=$(oc get secret "$AUTH_SECRET" -n koku-metrics-operator \
  -o jsonpath='{.data.client_secret}' | base64 -d)
TOKEN=$(curl -skS --fail "$KC/realms/cost-management/protocol/openid-connect/token" \
  --data-urlencode grant_type=client_credentials \
  --data-urlencode client_id="$CLIENT_ID" \
  --data-urlencode client_secret="$CLIENT_SECRET" | jq -er .access_token)
# Verify only non-secret claims; do not log the token or full identity header.
python3 - "$TOKEN" <<'PY'
import base64, json, sys
payload = sys.argv[1].split('.')[1]
payload += '=' * (-len(payload) % 4)
claims = json.loads(base64.urlsafe_b64decode(payload))
print('issuer=', claims.get('iss'))
print('org_admin=', 'org-admin' in claims.get('realm_access', {}).get('roles', []))
PY
curl -skS -o /tmp/source-types.json -w 'source_types_http=%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" "$API/source_types?filter[name]=openshift"
curl -skS -o /tmp/application-types.json -w 'application_types_http=%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" "$API/application_types"
curl -skS -o /tmp/sources.json -w 'sources_http=%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" "$API/sources"
# source_types and application_types must be rooted at API, not /sources/.
# The operator's target is sources_path + {source_types,application_types,sources}.
```

Expect `source_types_http=200`, `application_types_http=200`, and
`sources_http=200`. Then verify the CR reaches both `source_defined=true` and a
non-empty `last_successful_upload_time`:

```bash
oc get costmanagementmetricsconfig -n koku-metrics-operator costmanagementmetricscfg -o json \
  | jq '{source: .status.source, upload: .status.upload}'
```

If a source was manually registered or created by a timed-out retry, set
`source.create_source: false` and reconcile against the existing source rather
than creating a duplicate. If `source_defined=false` but the API already shows
the source, inspect `status.source.last_check_time`; temporarily set
`check_cycle: 1`, wait for a fresh check, then restore the normal interval.

Troubleshoot in this order:

1. `source_types` is 403: fix `sources_path` to exactly
   `/api/cost-management/v1/`; never use `/api/cost-management/v1/sources/`.
2. `sources` is 424: verify `org-admin` is in the client-credentials token and
   either enable `ENHANCED_ORG_ADMIN=True` or deploy a healthy RBAC API.
3. Koku logs show `account=None`, `org_id=None`, or `is_admin=False`: inspect
   gateway `JWT auth ok` logs and Koku identity parsing; do not change header
   casing or enable development auth without evidence. Useful redacted checks:
   ```bash
   oc logs -n cost-onprem -l app.kubernetes.io/component=gateway --since=5m \
     | grep -E 'JWT auth ok|Lua filter error'
   oc logs -n cost-onprem deployment/cost-onprem-koku-api --since=5m \
     | grep -E 'account|org_id|username|is_admin|403|424'
   ```
4. POST/create times out while Koku migrations are running: wait for the API
   migrations/readiness, then retry; this is not an authentication failure.

Do not declare collection complete until the operator has both packaged data
and uploaded at least one report successfully.

### Phase 7: ACCESS.md + Report

Update `~/rh/kcli/<cluster>-<hypervisor-short>/ACCESS.md` or the file the user
indicates (e.g. `~/rh/kcli/sno-apollo/ACCESS.md`). Use
[access-template.md](access-template.md).

Report to the user:

- Cluster / arch / namespace
- Branches + SHAs used (include `koku-metrics-operator`)
- Image tag (Helm apps + metrics operator)
- Gateway, UI, Keycloak URLs
- Metrics operator ns / `operator_commit` / upload-or-source status
- Keycloak test user / how to get admin password
- Any degraded pods or skipped steps (Kruize, tests, RBAC/source)

## Do NOT

- Install Helm with default Quay amd64 images on aarch64
- Reuse image tags
- Set Keycloak `org_id` to `org1234567`
- Cross-build with QEMU unless explicitly requested
- Tail logs with `--follow` unless the user asks
- Run pytest/IQE unless the user asks (default `--skip-chart-tests`)
- Patch OLM CSV to run the upstream metrics-operator image
- Use Keycloak realm `kubernetes` for the operator `token_url` (use `cost-management`)

## Additional Resources

- Pitfalls and aarch64 checklist: [reference.md](reference.md)
- ACCESS.md template: [access-template.md](access-template.md)
- Chart docs: `cost-onprem-chart/SNO-DEPLOYMENT-aarch64.md`
- Chart rules: `cost-onprem-chart/.cursor/rules/aarch64-sno-deployment.mdc`
- Metrics operator: `koku-metrics-operator/README.md`, `docs/local-development.md`
