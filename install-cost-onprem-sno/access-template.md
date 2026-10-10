# ACCESS.md Template — Cost Management On-Prem

Append or replace the Cost Management section in the cluster ACCESS.md.
Replace `{{placeholders}}`.

```markdown
## Cost Management On-Prem

Deployed {{DATE}} from local branches onto cluster `{{CLUSTER_NAME}}`
({{ARCH}}). Namespace: `cost-onprem`.

### Branches / Images

| Component | Repo branch | SHA | Image |
|-----------|-------------|-----|-------|
| Koku | `{{KOKU_BRANCH}}` | `{{KOKU_SHA}}` | `cost-onprem/koku:{{TAG}}` |
| ROS | `{{ROS_BRANCH}}` | `{{ROS_SHA}}` | `cost-onprem/ros-ocp-backend:{{TAG}}` |
| Ingress | `{{INGRESS_BRANCH}}` | `{{INGRESS_SHA}}` | `cost-onprem/insights-ingress-go:{{TAG}}` |
| UI | `{{UI_BRANCH}}` | `{{UI_SHA}}` | `cost-onprem/koku-ui:{{TAG}}` |
| Chart | `{{CHART_BRANCH}}` | `{{CHART_SHA}}` | Helm `cost-onprem` |
| Metrics operator | `{{KMO_BRANCH}}` | `{{KMO_SHA}}` | `cost-onprem/koku-metrics-operator:{{TAG}}` (ns `koku-metrics-operator`) |
| PostgreSQL | — | — | `cost-onprem/postgresql:16` |
| Valkey | — | — | `cost-onprem/valkey:8` |
| Kruize | {{KRUIZE_STATUS}} | | |

Values file: `{{VALUES_FILE}}`

### URLs

| Service | URL |
|---------|-----|
| API Gateway | https://cost-onprem-gateway-cost-onprem.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}/api/cost-management/v1/ |
| UI | https://cost-onprem-ui-cost-onprem.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}/ |
| Masu | https://cost-onprem-masu-cost-onprem.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}/ |
| Keycloak | https://keycloak-keycloak.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}/ |

### Keycloak

| Field | Value |
|-------|-------|
| Realm | `cost-management` |
| UI client | `cost-management-ui` |
| Test user | `{{TEST_USER}}` / `{{TEST_PASSWORD}}` |
| org_id | `1234567` (bare; schema `org1234567`) |
| account_number | `10001` |
| Admin | from secret `keycloak-initial-admin` in `keycloak` (often `temp-admin`) |

### Quick API Test

```bash
KEYCLOAK_URL="https://keycloak-keycloak.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}"
JWT=$(curl -sk "${KEYCLOAK_URL}/realms/cost-management/protocol/openid-connect/token" \
  -d "grant_type=password" -d "client_id=cost-management-ui" \
  -d "client_secret={{CLIENT_SECRET}}" \
  -d "username={{TEST_USER}}" -d "password={{TEST_PASSWORD}}" -d "scope=openid" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['access_token'])")

curl -sk -H "Authorization: Bearer ${JWT}" \
  "https://cost-onprem-gateway-cost-onprem.apps.{{CLUSTER_NAME}}.{{BASE_DOMAIN}}/api/cost-management/v1/status/"
```

### Metrics operator

| Field | Value |
|-------|-------|
| Namespace | `koku-metrics-operator` (upstream OLM-managed) |
| CSV | `{{KMO_CSV}}` (`Succeeded`) |
| CR | `{{KMO_CR}}` |
| Auth | service-account Secret `{{KMO_AUTH_SECRET}}` (`client_id` + `client_secret`) |
| token_url realm | `cost-management` |
| operator_commit | `{{KMO_SHA_FULL}}` |
| sources_path | `/api/cost-management/v1/` |

```bash
oc get costmanagementmetricsconfig -n koku-metrics-operator {{KMO_CR}} \
  -o jsonpath='commit={.status.operator_commit}{"\n"}prom={.status.prometheus.prometheus_connected}{"\n"}source={.status.source.source_defined}{"\n"}uploaded={.status.upload.last_successful_upload_time}{"\n"}error={.status.upload.error}{"\n"}'
```

### Notes

- RBAC disabled; Koku `ENHANCED_ORG_ADMIN=True`; operator service-account token has `org-admin`
- Object storage: {{STORAGE_BACKEND}} (buckets: `insights-upload-perma`, `koku-bucket`, `ros-data`)
- Hypervisor values/chart path: `/root/cost-onprem-chart/`
- Metrics operator source/upload: verify `/source_types` and `/sources` from the API root; do not use `/sources/source_types`
```
