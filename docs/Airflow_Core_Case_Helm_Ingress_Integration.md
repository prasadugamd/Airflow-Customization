# Core Case — Merge Airflow Ingress Helm Changes into Bitbucket Core helm-charts

| Field | Value |
|-------|--------|
| **Customer** | A1 Austria (Billing TC) |
| **Environment** | INT01 validated; PROD Stage 3 cutover planned |
| **Requestor** | TC / Airflow team |
| **Target repo** | Bitbucket **core helm-charts** (Account DevOps / Core) |
| **Reference** | INT live: `https://airflow.int.corp.amdocs.azr` → nginx `10.57.234.10` |
| **Local workspace** | `Airflow/helm-charts/` and `Airflow/PROD/prod-helm-charts/helm-charts/` |

---

## 1. Summary / description (for core case)

Airflow on AKS is today exposed via a **dedicated internal Azure LoadBalancer** (`airflow-srv-tcp`, port **1031**). This OOB pattern does not support standard HTTPS on port 443, customer-signed TLS at the edge, or alignment with the platform **nginx Ingress** used by API Gateway and other TCMS services.

We implemented and **validated on INT01** a Helm-based pattern that:

1. Adds an optional **Ingress** resource to chart **`services/airflow-srv`**
2. Switches **`airflow-srv-tcp`** from LoadBalancer to **ClusterIP** when ingress is enabled
3. Terminates TLS on the shared **nginx ingress controller** (`ms360-platform-crd`)
4. Uses **customer-signed certificate** via Kubernetes secret `airflow-tls`
5. Integrates with **external-dns** for automatic A record creation (requires platform prerequisite documented separately)

We request a **permanent merge** of the **template and values schema changes** below into **core helm-charts**, so INT/PROD (and future envs) deploy from the standard pipeline without maintaining a fork. Environment-specific hostnames, subnets, and OAuth settings remain in **`custom/override-values.yaml`** per env.

**Out of scope for this core case:** Platform external-dns / ClusterRole changes (separate Platform case). NFS `Op9_webserver_config.py` for Ping SSO (Stage 4 — application config on SHARED_PBIN).

---

## 2. Business justification

| Today (OOB LB) | Target (Ingress) |
|----------------|------------------|
| `https://<LB-IP>:1031` | `https://airflow.<env>.corp.amdocs.azr` (443) |
| Per-app Azure LB + IP management | Shared ingress IP (e.g. INT `10.57.234.10`, PROD `10.57.230.141`) |
| Citrix rules per LB IP:1031 | Standard 443 to ingress IP |
| Harder cert lifecycle | TLS secret on Ingress (`airflow-tls`) |

INT reference implementation: `Airflow_INGRESS_IMPLEMENTATION.txt`, `Airflow_Ingress_Deployment_SOP.md`.

---

## 3. Architecture

```text
Browser / VDI
  → https://airflow.<env>.corp.amdocs.azr:443
  → Azure Private DNS (external-dns on Ingress)
  → ingress-nginx-controller (ms360-platform-crd)
  → Ingress airflow-ingress (namespace int01-k8s / prod01-k8s)
  → Service airflow-srv-tcp:1031 (ClusterIP)
  → Airflow webserver pod (HTTPS on 1031)
```

**Important:** Airflow webserver speaks **HTTPS** on port **1031** (not HTTP 8080). Ingress must use `backend-protocol: HTTPS` and `proxy-ssl-verify: false`.

---

## 4. Changes to merge into CORE (templates + default values)

### 4.1 Chart: `services/airflow-srv`

| File | Change type | Description |
|------|-------------|-------------|
| **`templates/ingress.yaml`** | **NEW** | Conditional Ingress when `ingress.enabled: true`. Backend `{{ app.name }}-srv-tcp`, **`port.number`** (not port name — K8s 15-char limit on `airflow-webserver-port`). TLS from `ingress.tls.secretName`. |
| **`templates/service-tcp.yaml`** | **MODIFIED** | `spec.type`: **`ClusterIP` if `ingress.enabled`**, else `service.tcpports.type` (LoadBalancer default). Preserves LB annotations only when type is LoadBalancer. |
| **`values.yaml`** | **MODIFIED** | Add `ingress:` block (disabled by default). |
| **`.values-airflow.yaml`** | **MODIFIED** | Add `ingress:` block with Airflow-specific defaults (host placeholder, annotations, `backendPortNumber: 1031`). |

#### Ingress values schema (proposed core default)

```yaml
ingress:
  enabled: false
  name: airflow-ingress
  host: ""                    # set per env in custom/override-values.yaml
  path: /
  pathType: Prefix
  backendPortNumber: 1031     # required: port.name exceeds 15 chars in Service
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "false"
    external-dns.alpha.kubernetes.io/hostname: ""   # per-env FQDN
  tls:
    enabled: true
    secretName: airflow-tls
```

#### Key template logic (`ingress.yaml`)

- `metadata.name`: `{{ .Values.ingress.name }}`
- Backend service: `{{ .Values.app.name }}-srv-tcp`
- Backend port: `{{ .Values.ingress.backendPortNumber | default .Values.ports.airflow_webserver_port }}`

#### Key template logic (`service-tcp.yaml` line 27)

```yaml
type: {{ if .Values.ingress.enabled }}ClusterIP{{ else }}{{ .Values.service.tcpports.type }}{{ end }}
```

### 4.2 Chart: `deployments/airflow`

No **template** changes required for Stage 3 ingress if chart already supports `env_variables` / `acd_variables` from override files.

**Recommend core documentation** (README or values comments) for standard env vars when behind ingress:

| Env variable | Purpose |
|--------------|---------|
| `AIRFLOW__WEBSERVER__BASE_URL` | Public URL (must match Ingress host) |
| `AIRFLOW__WEBSERVER__ENABLE_PROXY_FIX` | Trust nginx `X-Forwarded-*` headers |
| `AIRFLOW__CORE__AUTH_MANAGER` | FabAuthManager (Stage 4 SSO) |
| `AIRFLOW__AMDOCS_OAUTH__AMDOCS_OAUTH_ENABLED` | Enable Ping OIDC (Stage 4 only) |
| `AIRFLOW_WEBSERVER_CONFIG_CUSTOMIZATION` | Use SHARED_PBIN `Op9_webserver_config.py` (Stage 4) |

---

## 5. Environment-specific CUSTOM overrides (NOT core — stay in account repo)

These files parameterize core charts per env; **do not hardcode INT/PROD FQDNs in core `values.yaml`**.

### `services/airflow-srv/custom/`

| File | Env | Purpose |
|------|-----|---------|
| `override-values.yaml` | INT | `ingress.enabled: true`, host `airflow.int.corp.amdocs.azr`, ClusterIP, INT Access subnet |
| `override-values-prod.yaml` | PROD | `ingress.enabled: true`, host `airflow.prod.corp.amdocs.azr`, PROD Access subnet |

### `deployments/airflow/custom/`

| File | Env | Stage | Purpose |
|------|-----|-------|---------|
| `override-values.yaml` | INT | 3+4 | BASE_URL, proxy fix, OAuth env (full SSO) |
| `override-values-prod.yaml` | PROD | 4 | Full prod hosts_aliases + OAuth |
| `override-values-stage3-ingress.yaml` | PROD | **3 only** | BASE_URL + proxy fix; **no OAuth**; `WEBSERVER_CONFIG_CUSTOMIZATION: False` |

---

## 6. INT custom values (reference for core reviewers)

### `services/airflow-srv/custom/override-values.yaml` (INT)

```yaml
ingress:
  enabled: true
  host: airflow.int.corp.amdocs.azr
  tls:
    secretName: airflow-tls
service:
  tcpports:
    type: ClusterIP
```

### `deployments/airflow/custom/override-values.yaml` (INT — Stage 4 / SSO)

```yaml
acd_variables:
  AIRFLOW_WEBSERVER_CONFIG_CUSTOMIZATION: "True"
env_variables:
  AIRFLOW__WEBSERVER__BASE_URL: "https://airflow.int.corp.amdocs.azr"
  AIRFLOW__WEBSERVER__ENABLE_PROXY_FIX: "True"
  AIRFLOW__CORE__AUTH_MANAGER: "airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager"
  AIRFLOW__AMDOCS_OAUTH__AMDOCS_OAUTH_ENABLED: "True"
```

---

## 7. PROD custom values (reference for core reviewers)

### `services/airflow-srv/custom/override-values.yaml` (PROD)

```yaml
ingress:
  enabled: true
  host: airflow.prod.corp.amdocs.azr
  tls:
    secretName: airflow-tls
service:
  tcpports:
    type: ClusterIP
service_annotations:
  global:
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "snet-at-a1g-Billing-PROD-we-001-Access"
```

### Stage 3 only — `override-values-stage3-ingress.yaml` (PROD)

```yaml
acd_variables:
  AIRFLOW_WEBSERVER_CONFIG_CUSTOMIZATION: "False"
env_variables:
  AIRFLOW__WEBSERVER__BASE_URL: "https://airflow.prod.corp.amdocs.azr"
  AIRFLOW__WEBSERVER__ENABLE_PROXY_FIX: "True"
```

---

## 8. Deploy commands (post-merge)

### INT

```bash
helm upgrade --install airflow-srv ./helm-charts/services/airflow-srv \
  -n int01-k8s \
  -f ./helm-charts/services/airflow-srv/.values-airflow.yaml \
  -f ./helm-charts/services/airflow-srv/custom/override-values.yaml

helm upgrade --install airflow ./helm-charts/deployments/airflow \
  -n int01-k8s \
  -f ./helm-charts/global/custom/global-override-values.yaml \
  -f ./helm-charts/deployments/airflow/custom/override-values.yaml
```

### PROD Stage 3 (ingress only)

```bash
helm upgrade --install airflow-srv ./helm-charts/services/airflow-srv \
  -n prod01-k8s \
  -f .values-airflow.yaml \
  -f custom/override-values-prod.yaml

helm upgrade --install airflow ./helm-charts/deployments/airflow \
  -n prod01-k8s \
  -f .../global-override-values.yaml \
  -f custom/override-values.yaml \
  -f custom/override-values-stage3-ingress.yaml
```

**Prerequisite:** `kubectl create secret tls airflow-tls ...` in release namespace.

---

## 9. Issues encountered on INT (include in core review)

| Issue | Resolution merged in charts |
|-------|----------------------------|
| Helm failed: `port.name` > 15 chars | Use `port.number: 1031` in Ingress backend |
| 502 Bad Gateway | `backend-protocol: HTTPS` (pod is HTTPS, not HTTP) |
| `airflow.cfg` overwritten on restart | Use `AIRFLOW__*` env vars in deployment override |
| Ingress CLASS deprecation warning | Annotation `kubernetes.io/ingress.class: nginx` still used (matches platform apigw) |

---

## 10. Dependencies (not part of core chart merge)

| Dependency | Owner |
|------------|--------|
| Platform external-dns `--source=ingress` | Platform / MS360 |
| ClusterRole `ingresses` on external-dns WI | Platform |
| Ingress SVC zone annotation (`int.corp` / `prod.corp`) | Platform |
| TLS secret `airflow-tls` (customer cert) | TC / cert pipeline |
| VDI DNS + firewall 443 → ingress IP | Network |

---

## 11. Acceptance criteria

- [ ] Core `airflow-srv` chart contains `templates/ingress.yaml` and conditional ClusterIP in `service-tcp.yaml`
- [ ] `ingress.enabled: false` by default — no behavior change for accounts not opting in
- [ ] INT deploy from core + custom override reproduces live Ingress (`airflow.int.corp.amdocs.azr` → ingress IP)
- [ ] PROD deploy from core + prod override reproduces planned Ingress (`airflow.prod.corp.amdocs.azr`)
- [ ] `helm template` renders valid Ingress with `port.number` (not invalid port name)
- [ ] Documentation in chart README: prerequisites, TLS secret, `backend-protocol: HTTPS`

---

## 12. Files to attach / PR diff summary

### Core merge (templates + values)

```
helm-charts/services/airflow-srv/
  templates/ingress.yaml          ADD
  templates/service-tcp.yaml      MODIFY (line ~27)
  values.yaml                     MODIFY (ingress block)
  .values-airflow.yaml            MODIFY (ingress block)
```

### Account custom (reference only — A1 repo)

```
helm-charts/services/airflow-srv/custom/
  override-values.yaml            INT
  override-values-prod.yaml       PROD

helm-charts/deployments/airflow/custom/
  override-values.yaml            INT
  override-values-prod.yaml         PROD
  override-values-stage3-ingress.yaml   PROD Stage 3

PROD/prod-helm-charts/helm-charts/   (mirror of above for prod pipeline)
```

### Related non-Helm (informational)

```
Op9_webserver_config.py           SHARED_PBIN — Ping SSO (Stage 4)
Airflow_IngresController_DNS_Implementation.txt
Airflow_INGRESS_IMPLEMENTATION.txt
```

---

## 13. Suggested core case title

```text
A1 Austria | Core helm-charts — Add optional Ingress to airflow-srv (replace OOB LoadBalancer pattern)
```

## 14. Suggested core case description (short — paste into ticket)

```text
Merge Airflow ingress support into core helm-charts (services/airflow-srv):

- New templates/ingress.yaml (optional, ingress.enabled flag)
- service-tcp.yaml: ClusterIP when ingress enabled (else LoadBalancer)
- values: ingress block (host, TLS secret, nginx annotations, backendPortNumber 1031)
- Backend HTTPS to pod port 1031; use port.number not port.name (K8s 15-char limit)

Validated on INT01: https://airflow.int.corp.amdocs.azr via shared nginx ingress.
PROD cutover planned: airflow.prod.corp.amdocs.azr.

Environment FQDNs and subnets remain in custom/override-values per env.
See attached Airflow_Core_Case_Helm_Ingress_Integration.md for full file list and diffs.
```
