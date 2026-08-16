# Monitoring (kube-prometheus-stack) - Chart Helm propio con dependencia (lab CKA)

Chart Helm propio, tipo *umbrella* (dependencia al chart oficial), para desplegar el stack completo de monitorización — Prometheus, Grafana, Alertmanager, Prometheus Operator, kube-state-metrics y node-exporter — gestionado vía ArgoCD con el patrón App of Apps.

## Por qué dependencia y no plantillas propias

Mismo criterio que `harbor`: stack multi-componente con Operator + CRDs propios (`Prometheus`, `Alertmanager`, `ServiceMonitor`...). Se usa el chart oficial `kube-prometheus-stack` del repo Prometheus Community como dependencia, controlando solo `values.yaml`.

## Estructura del chart

```
charts/monitoring/
├── Chart.yaml
├── Chart.lock
├── values.yaml
└── charts/
    └── kube-prometheus-stack-88.3.0.tgz
```

## 1. Chart.yaml

```yaml
apiVersion: v2
name: monitoring
description: Wrapper chart para kube-prometheus-stack (Prometheus + Grafana + Alertmanager) - lab CKA
type: application
version: 0.1.0
appVersion: "v0.93.0"
dependencies:
  - name: kube-prometheus-stack
    version: "88.3.0"
    repository: https://prometheus-community.github.io/helm-charts
```

## 2. values.yaml

Overrides anidados bajo `kube-prometheus-stack:`. NodePort para los tres componentes con UI/API propia:

```yaml
kube-prometheus-stack:
  grafana:
    service:
      type: NodePort
      nodePort: 30300
    adminPassword: "admin12345"

  prometheus:
    service:
      type: NodePort
      nodePort: 30900
    prometheusSpec:
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 300m
          memory: 512Mi

  alertmanager:
    service:
      type: NodePort
      nodePort: 30903
    alertmanagerSpec:
      resources:
        requests:
          cpu: 25m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
```

## 3. Resolver la dependencia

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm dependency update charts/monitoring
```

Genera `Chart.lock` + `.tgz`, ambos versionados en git — imprescindible, ArgoCD no ejecuta `helm dependency update` por sí solo.

## 4. Validación local

```bash
helm template monitoring charts/monitoring --namespace monitoring
helm lint charts/monitoring
```

## 5. Application de ArgoCD (`apps/monitoring-app.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: charts/monitoring
    helm:
      releaseName: monitoring
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## 6. Troubleshooting real (2 incidencias encadenadas)

### Incidencia 1 — CRDs demasiado grandes para `kubectl apply` client-side

**Síntoma:** `kubectl get prometheus -n monitoring` → `error: the server doesn't have a resource type "prometheus"`. Solo 4 de los 10 CRDs del stack se instalaron (`podmonitors`, `probes`, `prometheusrules`, `servicemonitors`); faltaban los más grandes: `prometheuses`, `alertmanagers`, `alertmanagerconfigs`, `prometheusagents`, `scrapeconfigs`, `thanosrulers`.

**Causa raíz:** ArgoCD usa por defecto `kubectl apply` (client-side), que almacena el estado previo del recurso en la anotación `kubectl.kubernetes.io/last-applied-configuration`. Esta anotación tiene un límite de 262144 bytes. Los CRDs grandes de Prometheus Operator superan ese límite → el apply falla con `metadata.annotations: Too long: must have at most 262144 bytes`.

**Solución:** añadir `ServerSideApply=true` a `syncOptions` — mueve el tracking del estado al servidor (`managedFields`) en vez de la anotación, evitando el límite.

**Nota real observada:** tras añadir la opción sobre una `Application` que ya venía fallando en bucle de reintentos, el auto-sync no recogió el cambio de inmediato (permaneció `OutOfSync` varios refresh). Fue necesario borrar el objeto `Application` (`kubectl delete application monitoring -n argocd`, sin tocar el namespace) y dejar que `root-app` la recreara desde cero desde `apps/monitoring-app.yaml` — con `ServerSideApply=true` presente desde el primer intento, sincronizó correctamente.

### Incidencia 2 — Prometheus Operator arrancó antes de que existieran sus propios CRDs

**Síntoma:** tras resolverse la incidencia 1, los CRDs `Prometheus`/`Alertmanager` existían y la `Application` estaba `Synced`, pero no se creaban los StatefulSets reales. `kubectl describe prometheus ... -n monitoring` mostraba `Events: <none>`.

**Causa raíz** (localizada en los logs del operator, filtrando el ruido de `TLS handshake error` de los probes):
```
resource "prometheuses" (group: "monitoring.coreos.com/v1") not installed in the cluster
```
El pod del operator arrancó (09:31) *antes* de que el CRD `prometheuses` se creara (09:37, tras el fix de la incidencia 1). El operator cachea la lista de recursos disponibles en su arranque y no vuelve a comprobarla después — se quedó con una vista congelada sin el CRD, aunque este apareciera más tarde.

**Solución:**
```bash
kubectl rollout restart deployment/monitoring-kube-prometheus-operator -n monitoring
```
Al reiniciar, el operator releyó la lista de recursos disponibles (ya con los CRDs completos), reconoció `Prometheus`/`Alertmanager` y creó los StatefulSets correctamente.

## 7. Componentes desplegados

| Pod | Función |
|---|---|
| `monitoring-grafana` | UI de dashboards |
| `prometheus-monitoring-kube-prometheus-prometheus-0` | motor de métricas (StatefulSet) |
| `alertmanager-monitoring-kube-prometheus-alertmanager-0` | gestión de alertas (StatefulSet) |
| `monitoring-kube-prometheus-operator` | Prometheus Operator (reconcilia los CRDs) |
| `monitoring-kube-state-metrics` | expone métricas del estado de los objetos K8s |
| `monitoring-prometheus-node-exporter` | DaemonSet, métricas de cada nodo |

## 8. Acceso

```bash
kubectl get svc -n monitoring
```

| Componente | URL | Credenciales |
|---|---|---|
| Grafana | `http://142.93.131.69:30300` | `admin` / `admin12345` |
| Prometheus | `http://142.93.131.69:30900` | — |
| Alertmanager | `http://142.93.131.69:30903` | — |

Datasource de Prometheus verificado en Grafana (`Connections > Data sources > Prometheus`): *"Successfully queried the Prometheus API"*.

## 9. Flujo de despliegue (GitOps)

```bash
git checkout develop
# ... cambios en charts/monitoring/ o apps/monitoring-app.yaml ...
# si se toca la dependencia: helm dependency update charts/monitoring (versionar el .tgz + Chart.lock)
git add -A
git commit -m "mensaje descriptivo"
git push origin develop
gh pr create --base main --head develop --title "..." --body "..."
gh pr merge --merge <numero-pr>
```

`root-app` detecta el cambio en `main` y sincroniza solo (`selfHeal: true`, `prune: true`).