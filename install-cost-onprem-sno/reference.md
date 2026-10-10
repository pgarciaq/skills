# Cost On-Prem SNO — Pitfalls Reference

Read before / during aarch64 deploys. Full detail also lives in
`cost-onprem-chart/.cursor/rules/aarch64-sno-deployment.mdc`.

## Architecture

| Cluster example | ARCH | Build where |
|-----------------|------|-------------|
| Apollo SNO (`hpe-apollo-cn99xx-*`) | `arm64` | Native on aarch64 hypervisor |
| UXSNO (Dell R640) | `amd64` | Native on x86_64 |

SNO ≠ aarch64. Always `oc get nodes` before choosing build flags.

## Image / Registry

| Issue | Fix |
|-------|-----|
| Stock chart images are amd64 | Build all custom images before Helm |
| `IfNotPresent` keeps old image | New unique tag every deploy |
| `podman login` fails with kubeconfig | SA `image-pusher` + `oc create token` |
| `podman push` blob auth fails | `skopeo copy` with `--dest-creds image-pusher:$TOKEN` |
| UI wrong container name | `oc set image ... app=...` (not `ui`/`nginx`) |
| UI wrong build | Use `apps/koku-ui-onprem/Containerfile` + `build:onprem` artifacts (`plugin-manifest.json`) |

## Required aarch64 images

Always build/push before Helm:

1. `koku`
2. `ros-ocp-backend`
3. `insights-ingress-go`
4. `koku-ui`
5. `postgresql:16` (retag `docker.io/library/postgres:16`)
6. `valkey:8`

Also build (Phase 6, same unique tag):

7. `koku-metrics-operator`

`autotune` / Kruize only if `kruize.enabled: true`.

## koku-metrics-operator

| Issue | Fix |
|-------|-----|
| OLM CSV image patch | Do **not** hand-edit the CSV; publish the current-branch image through the OLM bundle/catalog and verify CSV `Succeeded` |
| Two operator deployments | Stop the old kustomize deployment before OLM reconciliation; exactly one operator may own the CR |
| Keycloak realm `kubernetes` | Use `.../realms/cost-management/protocol/openid-connect/token` |
| Stale leader lease after rollout | `oc get lease -n koku-metrics-operator`, then remove only the stale lease identified for this installation |
| ImagePull auth from other project | Grant `system:image-puller` to `koku-metrics-controller-manager` on the image ns |
| Prometheus queries denied | `oc adm policy add-cluster-role-to-user cluster-monitoring-view -z koku-metrics-controller-manager -n koku-metrics-operator` |
| `source_types` returns 403 | `sources_path` is wrong. Use exactly `/api/cost-management/v1/`; do not use `/api/cost-management/v1/sources/` |
| Upload deferred / source 424 | Verify client-credentials JWT has `org-admin`; with `ENHANCED_ORG_ADMIN=True`, Koku can bypass the absent RBAC API. Otherwise deploy healthy RBAC |
| Koku logs show `account=None`, `org_id=None`, `is_admin=False` | Trace gateway `JWT auth ok` → generated identity → Koku identity parsing; do not guess at header casing |
| Source exists but `source_defined=false` | Check stale `last_check_time`; temporarily set `check_cycle: 1`, reconcile, then restore the normal interval |
| Source POST times out | Check whether Koku database migrations are still running; wait for readiness before treating it as auth failure |
| Upstream insights-rbac arm64 broken | `COPY /usr/lib64` into `hi/core-runtime` → bash stack smash; use UBI9 single-stage Dockerfile |
| Operator patches own Deployment volumes | Expects a brief rollout; wait for Ready + clear lease if stuck |

Deploy path: build/push image + OLM bundle/catalog → Subscription in
`koku-metrics-operator` → auth Secret → CR → verify CSV owner,
`status.operator_commit`, source status, and successful upload.

## S3 Buckets (exact names)

| Bucket | Consumer |
|--------|----------|
| `insights-upload-perma` | Ingress |
| `koku-bucket` | Listener / Masu |
| `ros-data` | ROS |

Wrong names → silent upload failures. Create
`cost-onprem-storage-credentials` before `install-helm-chart.sh`.

No ODF: set `objectStorage.endpoint` to
`minio.cost-onprem.svc.cluster.local` (or S4 endpoint).

## Keycloak / RHBK v26

| Pitfall | Fix |
|---------|-----|
| Bare hostname in CR | Use full `https://keycloak-....apps...` |
| Admin user `admin` | Read `username`+`password` from `keycloak-initial-admin` |
| Custom attrs dropped | Set realm user profile `unmanagedAttributePolicy: ADMIN_EDIT` |
| Schema `orgorg1234567` | User attr `org_id: "1234567"` only |
| "Account is not fully set up" | Delete/recreate user; do not add org_id as managed profile attrs |

`insights-rbac` is amd64-only → `rbac.enabled: false` + `ENHANCED_ORG_ADMIN`.
The role must be on the Keycloak service-account user, not only the human test
user. Verify `realm_access.roles` in the client-credentials token.

## Helm / Scripts

| Pitfall | Fix |
|---------|-----|
| `Values file not found` | Absolute `VALUES_FILE=/root/cost-onprem-chart/sno-arm64-values.yaml` |
| Relative path after `cd` | Never pass bare `sno-arm64-values.yaml` alone |
| First API 500 `cost_model_map` | Restart `cost-onprem-koku-api` + Valkey `FLUSHALL` |

## Hypervisor /etc/hosts

Include node IP for:

```
api.CLUSTER.DOMAIN
console-openshift-console.apps.CLUSTER.DOMAIN
oauth-openshift.apps.CLUSTER.DOMAIN
default-route-openshift-image-registry.apps.CLUSTER.DOMAIN
keycloak-keycloak.apps.CLUSTER.DOMAIN
cost-onprem-gateway-cost-onprem.apps.CLUSTER.DOMAIN
cost-onprem-masu-cost-onprem.apps.CLUSTER.DOMAIN
cost-onprem-ui-cost-onprem.apps.CLUSTER.DOMAIN
```

## Sync Caveats

- `autotune`: rsync `--exclude='/target'` (leading `/`). Bare `target` deletes
  valid Java package paths under `.../common/target/...`.
- Prefer syncing current working tree (including uncommitted fixes the user
  expects) unless they ask for clean `git archive` only.

## Nise (if seeding data)

Use `--write-monthly --ros-ocp-info`, not `--daily-reports` (needs Insights env
vars and can produce empty output).

## Checklist (aarch64)

- [ ] Node ARCH is `arm64`
- [ ] Built + pushed all 6 base images with unique app tag
- [ ] Built + pushed `koku-metrics-operator` with same tag
- [ ] Values file tags match pushed images
- [ ] `/etc/hosts` has registry + app routes
- [ ] RHBK hostnames are `https://...`
- [ ] `ADMIN_EDIT` unmanaged attrs; `org_id=1234567`
- [ ] Operator Secret has `client_id` + `client_secret`; token realm is `cost-management`
- [ ] Client-credentials token contains `org-admin`; gateway/Koku identity logs are consistent
- [ ] CR `source.sources_path` is exactly `/api/cost-management/v1/`
- [ ] Three S3 buckets with exact names
- [ ] Storage credentials secret exists
- [ ] Helm install with absolute values path
- [ ] Restart Koku API after first tenant; flush Valkey
- [ ] Metrics operator via upstream OLM; CSV `Succeeded` and Deployment owned by CSV
- [ ] CR token_url realm `cost-management`; `source_types`, `sources` return 200
- [ ] `source_defined=true` and `last_successful_upload_time` is populated
- [ ] `status.operator_commit` matches branch SHA; prometheus connected
