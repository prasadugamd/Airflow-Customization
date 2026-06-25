# Firewall / Network Allow-List Request — PRODUCTION

## Airflow → PingFederate (OIDC) — PROD (AKS)

| Field | Value |
|---|---|
| **Ticket ID** | FW-A1-TC-AIRFLOW-PING-PROD-001 |
| **Requestor** | TC / Airflow Admin |
| **Reference (INT)** | FW-A1-TC-AIRFLOW-PING-001 — same pattern, INT01 approved/worked |
| **Priority** | High — blocks SSO go-live for Airflow on PROD |
| **Related Systems** | Airflow (AKS `prod-01-a1-prod-aks`), PingFederate, nginx ingress |

---

## 1. Summary (short — for ticket header)

**Environment:**  
AKS **`prod-01-a1-prod-aks`**, Worker subnet **`10.57.229.0/24`**, egress via UDR to NVA **`10.56.4.68`**.  
Airflow OIDC to **`moria.a1.group`** (**`10.89.169.241`**) — PROD IdP; INT uses `moria-t.a1.group` / `10.89.252.241`.

**Firewall rule request (on NVA `10.56.4.68`):**

| Field | Value |
|---|---|
| **Source** | **`10.57.229.0/24`** |
| **Destination** | **`10.89.169.241`** / **`moria.a1.group`** |
| **Service** | HTTPS (TCP 443) |
| **Purpose** | Airflow PingFederate OIDC (discovery / token / userinfo) |

---

## 2. Request

Open **outbound HTTPS (TCP 443)** from the **PROD AKS worker subnet** to **PingFederate** for Airflow OIDC SSO.

| Field | Value |
|---|---|
| **Source** | **`10.57.229.0/24`** — `snet-at-a1g-Billing-PROD-we-001-Worker` |
| **Source (optional)** | SNAT IP(s) seen by Ping from prod egress appliance — Platform to confirm |
| **Destination FQDN** | **`moria.a1.group`** |
| **Destination IP** | **`10.89.169.241`** ✓ confirmed (`getent hosts` on prod-01-tccli01) |
| **Port / protocol** | TCP 443 |
| **Direction** | Outbound from PROD VNet → IdP (allow on UDR next-hop NVA / Azure Firewall) |
| **Business purpose** | OIDC SSO for Airflow UI `https://airflow.prod.corp.amdocs.azr` |
| **Duration** | Permanent |

**Note:** Ingress UI traffic (`https://airflow.prod.corp.amdocs.azr` → nginx **`10.57.230.141`**) is **separate** from this OIDC egress rule. This ticket is **pod → Ping IdP** only.

**INT vs PROD IdP:** INT → **`moria-t.a1.group`** (`10.89.252.241`). PROD → **`moria.a1.group`** (`10.89.169.241`).

---

## 3. PROD environment details

| Item | PROD value |
|---|---|
| **Subscription** | `at-a1g-Billing-PROD` (`23c1efd9-a427-44ed-b37f-0781ea6703f0`) |
| **AKS cluster** | `prod-01-a1-prod-aks` |
| **API server** | `dns-prod-01-a1-prod-aks-essn3wsf.hcp.westeurope.azmk8s.io` |
| **Outbound type** | **`userDefinedRouting`** ✓ confirmed |
| **Network plugin** | `kubenet` |
| **Pod CIDR** | `172.18.0.0/16` |
| **Service CIDR** | `192.168.0.0/16` |
| **Private cluster** | Enabled |
| **Ingress controller** | `ingress-nginx-controller` in `ms360-platform-crd` |
| **Ingress LB IP** | **`10.57.230.141`** (443/80) |
| **external-dns** | `ms360-azr-external-dns-wi`, owner `prod-01-a1-prod-aks` |
| **Airflow namespace** | `<prod-k8s-ns>` — e.g. `prod01-k8s` *(confirm)* |
| **Airflow node pool** | **`prd1tc`** (pod CIDR e.g. `172.18.9.0/24` → node `10.57.229.60`) |
| **Worker subnet** | **`snet-at-a1g-Billing-PROD-we-001-Worker`** / **`10.57.229.0/24`** ✓ |
| **VNet** | `vnet-at-a1g-Billing-PROD-we-001` (`rg-at-a1g-Billing-PROD-we-network-001`) |
| **Route table** | **`route-vnet-at-a1g-Billing-PROD-we-001-worker`** ✓ |
| **Default route** | **`0.0.0.0/0` → VirtualAppliance `10.56.4.68`** (`default-route-internet`) ✓ |
| **NVA (egress appliance)** | **`10.56.4.68`** — same appliance as INT |
| **NSG** | **`nsg-snet-at-a1g-Billing-PROD-we-001-Worker`** ✓ |

---

## 4. Commands to fill blanks (run on `prod-01-tccli01-vmss000001`)

### 4a. Already confirmed

```bash
# outboundType → userDefinedRouting ✓
# Worker subnet ID ✓
SUBNET_ID="/subscriptions/23c1efd9-a427-44ed-b37f-0781ea6703f0/resourceGroups/rg-at-a1g-Billing-PROD-we-network-001/providers/Microsoft.Network/virtualNetworks/vnet-at-a1g-Billing-PROD-we-001/subnets/snet-at-a1g-Billing-PROD-we-001-Worker"
```

### 4b. Confirmed — subnet CIDR, route table, NSG

```json
{
  "subnet": "snet-at-a1g-Billing-PROD-we-001-Worker",
  "prefix": "10.57.229.0/24",
  "routeTable": "route-vnet-at-a1g-Billing-PROD-we-001-worker",
  "nsg": "nsg-snet-at-a1g-Billing-PROD-we-001-Worker"
}
```

### 4c. Confirmed — NVA IP (default route)

```text
AddressPrefix  Name                    NextHopIpAddress  NextHopType
0.0.0.0/0      default-route-internet  10.56.4.68        VirtualAppliance
```

**Note:** PROD uses the **same NVA (`10.56.4.68`)** as INT. The new rule must allow the **PROD worker subnet** (`10.57.229.0/24`), not INT (`10.57.232.0/24`).

Airflow prod node pool **`prd1tc`** — example pod→node hops from route table:

| Pod CIDR | Node IP |
|---|---|
| `172.18.9.0/24` | `10.57.229.60` |
| `172.18.210.0/24` | `10.57.229.6` |
| `172.18.63.0/24` | `10.57.229.65` |

### 4d. Verification (after rule is applied)
NS=<prod-k8s-ns>
POD=$(kubectl get po -n $NS -l app=airflow -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n $NS $POD -c airflow -- curl -sk -o /dev/null -w "discovery:%{http_code}\n" \
  https://moria-t.a1.group/.well-known/openid-configuration
# Expected after rule: 200
```

---

## 5. Application / SSO registration (PROD)

| Item | Value |
|---|---|
| **Airflow UI URL** | `https://airflow.<prod-fqdn>` |
| **Login URL** | `https://airflow.<prod-fqdn>/login` |
| **Logout URL** | `https://airflow.<prod-fqdn>/logout` |
| **Redirect URI** | `https://airflow.<prod-fqdn>/oauth-authorized/ping` |
| **OIDC client ID** | `<confirm with SSO>` — e.g. `AmdocsAirflow` / prod-specific |
| **Ping IdP host** | `moria-t.a1.group` *(confirm prod vs INT)* |
| **Scopes** | `openid email profile groups` |

---

## 6. Expected symptom before rule (same as INT)

If prod matches INT networking behaviour:

```text
$ curl -vk https://moria-t.a1.group/.well-known/openid-configuration
* Connected to moria-t.a1.group (10.89.252.241) port 443
* OpenSSL SSL_connect: SSL_ERROR_SYSCALL
curl: (35) ...
```

TCP 443 may connect; TLS handshake fails (`0 bytes read`, `Cipher (NONE)`).

Run from **prod Airflow pod** and **prod-01-tccli01** before/after rule for evidence.

---

## 7. Verification after rule is implemented

```bash
NS=<prod-k8s-ns>
POD=$(kubectl get po -n $NS -l app=airflow -o jsonpath='{.items[0].metadata.name}')

# (a) Discovery document
kubectl exec -n $NS $POD -c airflow -- curl -sk -o /dev/null \
  -w "openid-config HTTP=%{http_code}\n" \
  https://moria-t.a1.group/.well-known/openid-configuration
# Expected: 200

# (b) TLS handshake
kubectl exec -n $NS $POD -c airflow -- bash -lc \
  'echo | openssl s_client -connect moria-t.a1.group:443 -servername moria-t.a1.group 2>&1 | grep -E "subject=|Cipher is"'

# (c) End-to-end SSO
curl -vk https://airflow.<prod-fqdn>/login 2>&1 | grep -iE "HTTP/|Location:"
# Expected: redirect to Ping authorize URL
```

---

## 8. Related PROD prerequisites (not this ticket, but same cutover)

| Item | Owner | Status |
|---|---|---|
| external-dns `--source=ingress` on prod | Platform | Pending INT parity |
| ClusterRole `ingresses` on prod | Platform | Verify |
| Ingress SVC zone annotation (prod zone) | Platform | Pending |
| Airflow Ingress → `10.57.230.141` | TC / Helm | Pending |
| TLS secret `airflow-tls` (prod SAN) | TC | Pending |
| Ping OIDC client + groups (prod) | SSO | Pending |

---

## 9. Contacts

| Role | Action |
|---|---|
| **Network / Azure** | UDR / NVA rule; provide prod SNAT IP if required by Ping |
| **Ping / SSO** | Confirm prod IdP host, client ID, redirect URI, allow-list source IP |
| **TC / Airflow** | Run §7 verification after rule |

---

## 10. Copy-paste ticket body — Request 3 (OIDC egress) — ready to submit

```text
Subject: Request 3 — Allow HTTPS from AKS PROD Worker subnet to PingFederate — Airflow OIDC

Environment:
  Subscription: at-a1g-Billing-PROD (23c1efd9-a427-44ed-b37f-0781ea6703f0)
  AKS: prod-01-a1-prod-aks (outboundType: userDefinedRouting)
  Worker subnet: snet-at-a1g-Billing-PROD-we-001-Worker (10.57.229.0/24)
  Route table: route-vnet-at-a1g-Billing-PROD-we-001-worker
  Default egress: 0.0.0.0/0 → VirtualAppliance 10.56.4.68 (default-route-internet)
  NSG: nsg-snet-at-a1g-Billing-PROD-we-001-Worker
  Airflow node pool: prd1tc (pod CIDR e.g. 172.18.9.0/24 → node 10.57.229.60)
  Ingress (UI, separate — Request 1/2): nginx 10.57.230.141

Firewall rule request (on NVA 10.56.4.68):
  Source:      10.57.229.0/24
  Destination: 10.89.169.241 (moria.a1.group)
  Service:     HTTPS (TCP 443)
  Purpose:     Airflow PingFederate OIDC (discovery / token / userinfo)

Application:
  Airflow UI: https://airflow.prod.corp.amdocs.azr
  OIDC redirect: https://airflow.prod.corp.amdocs.azr/oauth-authorized/ping
  Client: AdmocsAirflow (confirm exact spelling with SSO)

Reference: INT01 — 10.57.232.0/24 → NVA 10.56.4.68 → moria-t.a1.group (10.89.252.241)
           INT ticket FW-A1-TC-AIRFLOW-PING-001
           PROD IdP: moria.a1.group → 10.89.169.241 (not moria-t / 10.89.252.241)

Symptom (expected before rule):
  TCP 443 connects; TLS Client Hello sent; handshake fails SSL_ERROR_SYSCALL (0 bytes read).

Verify after change (from prod Airflow pod or prod-01-tccli01):
  getent hosts moria.a1.group
  curl -sk -w "%{http_code}\n" -o /dev/null \
    https://moria.a1.group/.well-known/openid-configuration
  Expected: 200
```
