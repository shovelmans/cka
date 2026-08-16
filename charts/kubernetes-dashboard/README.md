# Kubernetes Dashboard - Chart Helm propio (lab CKA)

Chart Helm propio (sin dependencias externas, plantillas escritas a mano) para desplegar la versión clásica del Kubernetes Dashboard (`kubernetesui/dashboard:v2.7.0`), gestionado vía ArgoCD con el patrón App of Apps.

## Por qué chart propio y no dependencia

El chart oficial moderno (`kubernetes-dashboard/kubernetes-dashboard` 7.x, repo `https://kubernetes-retired.github.io/dashboard/`) despliega un stack multi-componente (Kong, Redis, metrics-scraper...). Para entender el patrón Helm de principio a fin, se optó por construir un chart propio con la versión clásica mono-Deployment, replicando el mismo enfoque usado en `charts/apache-demo`.

## Estructura del chart

```
charts/kubernetes-dashboard/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── serviceaccount.yaml   # ServiceAccount + ClusterRoleBinding a cluster-admin
    ├── deployment.yaml
    ├── service.yaml
    └── secrets.yaml          # Secrets vacíos requeridos por el Dashboard (ver troubleshooting)
```

## 1. Chart.yaml

```yaml
apiVersion: v2
name: kubernetes-dashboard
description: Chart propio de Kubernetes Dashboard (lab CKA), sin dependencias externas
type: application
version: 0.1.0
appVersion: "2.7.0"
```

## 2. values.yaml

```yaml
replicaCount: 1

image:
  repository: kubernetesui/dashboard
  tag: "v2.7.0"

service:
  type: NodePort
  port: 443

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

## 3. Validación local antes de subir

```bash
helm template kubernetes-dashboard charts/kubernetes-dashboard
helm lint charts/kubernetes-dashboard
```

## 4. Application de ArgoCD (`apps/dashboard-app.yaml`)

Descubierta automáticamente por `root-app` (patrón App of Apps) al hacer merge a `main`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubernetes-dashboard
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: charts/kubernetes-dashboard
    helm:
      releaseName: kubernetes-dashboard
  destination:
    server: https://kubernetes.default.svc
    namespace: kubernetes-dashboard
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

## 5. Troubleshooting real: CrashLoopBackOff por Secret CSRF

Al primer despliegue el pod entraba en `CrashLoopBackOff` con:

```
panic: secrets "kubernetes-dashboard-csrf" not found
```

El binario del Dashboard v2.7.0 con `--auto-generate-certificates` intenta **leer** el secret `kubernetes-dashboard-csrf` antes de crearlo; si no existe, hace panic en vez de crearlo automáticamente. El RBAC (`cluster-admin` vía ClusterRoleBinding) ya permitía crearlo — el problema no era de permisos, sino de que el Secret no existía de antemano.

**Solución:** pre-crear los tres Secrets vacíos que el Dashboard espera, como plantilla propia del chart (`templates/secrets.yaml`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-dashboard-csrf
type: Opaque
---
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-dashboard-key-holder
type: Opaque
---
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-dashboard-certs
type: Opaque
```

El Dashboard los rellena en caliente en el arranque. Tras el `merge` a `main`, como ArgoCD no reinicia el Deployment solo por un cambio en Secrets ya existentes, fue necesario un restart manual:

```bash
kubectl rollout restart deployment/kubernetes-dashboard -n kubernetes-dashboard
```

## 6. Acceso

```bash
kubectl get svc -n kubernetes-dashboard
kubectl get nodes -o wide
```

Accede a `https://<IP-del-nodo>:<nodePort>` (certificado autofirmado, aceptar aviso del navegador).

Generar token de login (expira en 1h por defecto):

```bash
kubectl -n kubernetes-dashboard create token kubernetes-dashboard
```

Pega el token en la opción **Token** de la pantalla de login.

> ⚠️ El `ServiceAccount` tiene un `ClusterRoleBinding` a `cluster-admin`: acceso total al cluster sin restricciones RBAC. Válido para lab, nunca replicar así en un cluster real — en producción se usa un RBAC mínimo y acceso vía Bearer Token restringido.

## 7. Flujo de despliegue (GitOps)

Como el resto de apps del repo, ningún comando `kubectl apply` ni `helm install` manual sobre el workload: todo pasa por el flujo estándar del repo.

```bash
git checkout develop
# ... cambios en charts/kubernetes-dashboard/ o apps/dashboard-app.yaml ...
git add -A
git commit -m "mensaje descriptivo"
git push origin develop
gh pr create --base main --head develop --title "..." --body "..."
gh pr merge --merge <numero-pr>
```

`root-app` detecta el cambio en `main` y sincroniza solo (`selfHeal: true`, `prune: true`).