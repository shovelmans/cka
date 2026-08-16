# Ingress - ingress-nginx + Ingress por app (lab CKA)

Despliegue del controller `ingress-nginx` vía Helm (patrón dependencia) y migración de todos los servicios expuestos por NodePort a acceso unificado vía Ingress con hostname, gestionado por ArgoCD.

## Por qué Ingress en vez de NodePort

Con NodePort cada servicio necesita recordar un puerto distinto (`30300`, `30900`, `30002`...). Con Ingress, todo entra por un único puerto (`31080` HTTP / `31443` HTTPS) y se enruta según el hostname de la petición (cabecera `Host`) — el mismo modelo que las Routes de OpenShift, pero con el objeto estándar de Kubernetes.

## 1. Controller: chart propio con dependencia al oficial

Mismo patrón que Harbor y Monitoring: chart wrapper con `dependencies:` al chart oficial `kubernetes.github.io/ingress-nginx`.

```
charts/ingress-nginx/
├── Chart.yaml
├── Chart.lock
├── values.yaml
└── charts/
    └── ingress-nginx-4.15.1.tgz
```

### Chart.yaml

```yaml
apiVersion: v2
name: ingress-nginx
description: Wrapper chart para ingress-nginx (lab CKA)
type: application
version: 0.1.0
appVersion: "1.15.1"
dependencies:
  - name: ingress-nginx
    version: "4.15.1"
    repository: https://kubernetes.github.io/ingress-nginx
```

### values.yaml

```yaml
ingress-nginx:
  controller:
    service:
      type: NodePort
      nodePorts:
        http: 31080
        https: 31443
    resources:
      requests:
        cpu: 50m
        memory: 90Mi
      limits:
        cpu: 200m
        memory: 180Mi
```

> ⚠️ Puertos `31080`/`31443` elegidos deliberadamente distintos de `30080`/`30443` (ya usados por ArgoCD) para evitar conflicto de NodePorts.

### Resolver dependencia

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm dependency update charts/ingress-nginx
```

### Application de ArgoCD (`apps/ingress-nginx-app.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: charts/ingress-nginx
    helm:
      releaseName: ingress-nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

## 2. Regla general: nunca hardcodear la IP en el hostname

Primer intento erróneo: usar `grafana.<IP-del-nodo>.nip.io` como hostname — funciona, pero si el cluster se recrea con otra IP, el `Ingress` en git queda obsoleto y hay que tocar el repo cada vez.

**Solución adoptada:** hostnames fijos tipo `<app>.lab.local`, resueltos vía `/etc/hosts` local (`C:\Windows\System32\drivers\etc\hosts` en Windows) — nunca en git. Si el cluster cambia de IP, solo se actualiza esa línea local, el `Ingress` en el repo nunca cambia.

```
159.223.3.171 grafana.lab.local
159.223.3.171 harbor.lab.local
159.223.3.171 dashboard.lab.local
159.223.3.171 alertmanager.lab.local
159.223.3.171 prometheus.lab.local
159.223.3.171 argocd.lab.local
```

Editar en Windows: Bloc de notas **ejecutado como administrador** → Archivo → Abrir → `C:\Windows\System32\drivers\etc\hosts` (filtro "Todos los archivos") → añadir líneas → guardar. O por PowerShell como administrador: `notepad "C:\Windows\System32\drivers\etc\hosts" -Verb RunAs`.

## 3. Ingress por app

### Grafana (`charts/monitoring/templates/grafana-ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monitoring-grafana
                port:
                  number: 80
```

### Prometheus (`charts/monitoring/templates/prometheus-ingress.yaml`)
Igual que Grafana, `service: monitoring-kube-prometheus-prometheus`, `port: 9090`, `host: prometheus.lab.local`.

### Alertmanager (`charts/monitoring/templates/alertmanager-ingress.yaml`)
Igual, `service: monitoring-kube-prometheus-alertmanager`, `port: 9093`, `host: alertmanager.lab.local`.

### Kubernetes Dashboard (`charts/kubernetes-dashboard/templates/ingress.yaml`)

El Dashboard sirve internamente HTTPS (certificados autogenerados con `--auto-generate-certificates`), y además **rechaza el login por Token si la conexión del navegador no es segura** (HTTPS o localhost) — protección propia del proyecto. Requiere terminación TLS también en el Ingress, no basta con `backend-protocol: HTTPS`.

Generar certificado autofirmado y Secret TLS:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/dashboard-tls.key -out /tmp/dashboard-tls.crt \
  -subj "/CN=dashboard.lab.local"

kubectl create secret tls dashboard-tls-secret \
  -n kubernetes-dashboard \
  --cert=/tmp/dashboard-tls.crt --key=/tmp/dashboard-tls.key \
  --dry-run=client -o yaml > charts/kubernetes-dashboard/templates/tls-secret.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - dashboard.lab.local
      secretName: dashboard-tls-secret
  rules:
    - host: dashboard.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 443
```

Acceso: `https://dashboard.lab.local:31443` (puerto TLS del controller) — aceptar aviso de certificado autofirmado.

### Harbor (`charts/harbor/templates/harbor-ingress.yaml`)
`service: harbor`, `port: 80`, `host: harbor.lab.local`. (Chart de dependencia sin `templates/` previa — se creó la carpeta al añadir este primer recurso propio.)

### ArgoCD (`manifests/argocd-ingress/argocd-ingress.yaml`)

ArgoCD **no** vive dentro de un chart gestionado por ArgoCD (`argocd/` solo contiene el `values.yaml` del `helm install` inicial), así que su Ingress necesita su propia mini-Application de manifiestos planos:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-ingress
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: manifests/argocd-ingress
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

`server.insecure: true` en `argocd/values.yaml` hace que sirva HTTP plano, así que su Ingress no necesita TLS especial (a diferencia del Dashboard).

## 4. Accesos finales

| App | URL | Usuario | Contraseña |
|---|---|---|---|
| Grafana | `http://grafana.lab.local:31080` | `admin` | `admin12345` |
| ArgoCD | `http://argocd.lab.local:31080` | `admin` | `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" \| base64 -d` |
| Kubernetes Dashboard | `https://dashboard.lab.local:31443` | — | Token: `kubectl -n kubernetes-dashboard create token kubernetes-dashboard` (expira 1h) |
| Prometheus | `http://prometheus.lab.local:31080` | — | sin auth |
| Alertmanager | `http://alertmanager.lab.local:31080` | — | sin auth |
| Harbor | `http://harbor.lab.local:31080` | `admin` | `Harbor12345` |

## 5. Flujo de despliegue (GitOps)

Igual que el resto del repo — cambios en `charts/*/templates/*ingress*.yaml` o `manifests/argocd-ingress/` siguen `develop → PR → main`. `ingress-nginx`, `kubernetes-dashboard` y `harbor` tienen auto-sync activo; `monitoring` requiere sync manual (`syncPolicy.automated.enabled: false` a propósito).

```bash
git checkout develop
# ... cambios ...
git add -A
git commit -m "mensaje descriptivo"
git push origin develop
gh pr create --base main --head develop --title "..." --body "..."
gh pr merge --merge <numero-pr>
```