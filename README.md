# Airflow-Customization

A1 Austria — Helm customizations for **Airflow Ingress** on AKS (shared nginx ingress + customer TLS).

Validated on **INT01**: `https://airflow.int.corp.amdocs.azr` → nginx ingress `10.57.234.10`  
Target **PROD**: `https://airflow.prod.corp.amdocs.azr` → nginx ingress `10.57.230.141`

## Repository layout

```
helm-charts/
  services/airflow-srv/          # Ingress template + ClusterIP when ingress enabled
  deployments/airflow/custom/    # BASE_URL, proxy fix, OAuth (Stage 4) overrides
docs/                            # Implementation notes and core-case description
manifests/                       # Standalone Ingress YAML (reference; Helm is source of truth)
```

## Deploy (INT)

```bash
kubectl create secret tls airflow-tls --cert=fullchain.pem --key=airflow.key \
  -n int01-k8s --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install airflow-srv ./helm-charts/services/airflow-srv \
  -n int01-k8s \
  -f ./helm-charts/services/airflow-srv/.values-airflow.yaml \
  -f ./helm-charts/services/airflow-srv/custom/override-values.yaml

helm upgrade --install airflow ./helm-charts/deployments/airflow \
  -n int01-k8s \
  -f ./helm-charts/deployments/airflow/custom/override-values.yaml
```

## Deploy (PROD — Stage 3 ingress only, no SSO)

```bash
helm upgrade --install airflow-srv ./helm-charts/services/airflow-srv \
  -n prod01-k8s \
  -f ./helm-charts/services/airflow-srv/.values-airflow.yaml \
  -f ./helm-charts/services/airflow-srv/custom/override-values-prod.yaml

helm upgrade --install airflow ./helm-charts/deployments/airflow \
  -n prod01-k8s \
  -f ./helm-charts/deployments/airflow/custom/override-values-prod.yaml \
  -f ./helm-charts/deployments/airflow/custom/override-values-stage3-ingress.yaml
```

## Prerequisites

- Platform: external-dns `--source=ingress`, ClusterRole `ingresses`, ingress SVC zone annotation
- TLS secret `airflow-tls` in release namespace (SAN = Ingress host)
- VDI DNS + firewall 443 → ingress IP (separate network requests)

See `docs/Airflow_IngresController_DNS_Implementation.txt` and `docs/Airflow_Core_Case_Helm_Ingress_Integration.md`.

## Key technical notes

- Airflow webserver listens on **HTTPS port 1031** — Ingress uses `backend-protocol: HTTPS`
- Ingress backend must use **`port.number: 1031`** (not port name; K8s 15-character limit)
- When `ingress.enabled: true`, `airflow-srv-tcp` renders as **ClusterIP** (retires dedicated LoadBalancer)
