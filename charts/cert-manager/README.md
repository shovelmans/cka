# cert-manager - CA propia y TLS automático (lab CKA)

Despliegue de `cert-manager` vía Helm (patrón dependencia) y configuración de una CA autofirmada propia del cluster, usada para emitir certificados TLS gestionados automáticamente en todos los Ingress del lab.

## Por qué cert-manager

Antes de esto, el TLS del Dashboard se generó a mano con `openssl` (certificado estático, sin renovación). cert-manager automatiza todo el ciclo de vida: emisión, renovación y rotación de certificados, sin intervención manual — el estándar real en cualquier cluster de producción.

## 1. Controller: chart propio con dependencia al oficial

```
charts/cert-manager/
├── Chart.yaml
├── Chart.lock
├── values.yaml
└── charts/
    └── cert-manager-v1.21.1.tgz
```

### Chart.yaml

```yaml
apiVersion: v2
name: cert-manager
description: Wrapper chart para cert-manager (lab CKA)
type: application
version: 0.1.0
appVersion: "v1.21.1"
dependencies:
  - name: cert-manager
    version: "v1.21.1"
    repository: https://charts.jetstack.io
```

### values.yaml

```yaml
cert-manager:
  crds:
    enabled: true
  resources:
    requests:
      cpu: 25m
      memory: 32Mi
    limits:
      cpu: 100m
      memory: 128Mi
```

> A diferencia de `kube-prometheus-stack`, el chart de cert-manager incluye sus propios CRDs vía el flag `crds.enabled: true` — no requiere un paso de instalación de CRDs aparte.

### Resolver dependencia

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm dependency update charts/cert-manager
```

### Application de ArgoCD (`apps/cert-manager-app.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: charts/cert-manager
    helm:
      releaseName: cert-manager
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

> `ServerSideApply=true` desde el principio — lección aprendida de la incidencia con los CRDs de `kube-prometheus-stack` (límite de 262144 bytes en la anotación `last-applied-configuration` con `kubectl apply` client-side). Con esta opción desde el inicio, los 6 CRDs de cert-manager (`certificates`, `certificaterequests`, `issuers`, `clusterissuers`, `orders`, `challenges`) se instalaron sin ningún problema.

## 2. CA autofirmada propia

Estructura en tres piezas, encadenadas (`manifests/cert-manager-ca/ca-issuer.yaml`):

```yaml
# 1. Issuer "semilla" que se autofirma, solo para arrancar la cadena
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-bootstrap
  namespace: cert-manager
spec:
  selfSigned: {}
---
# 2. Certificado marcado como CA (isCA: true), firmado por el Issuer bootstrap
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lab-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: lab.local CA
  secretName: lab-ca-secret
  privateKey:
    algorithm: RSA
    size: 2048
  issuerRef:
    name: selfsigned-bootstrap
    kind: Issuer
---
# 3. El emisor real que se usa en cada app, basado en la CA generada en el paso 2
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lab-ca-issuer
spec:
  ca:
    secretName: lab-ca-secret
```

Gestionado por su propia `Application` de manifiestos planos (`apps/cert-manager-ca-app.yaml`), mismo patrón que `argocd-ingress` — no hay un chart natural donde colgarlo.

## 3. Certificate por app

Cada app tiene un `Certificate` propio en su carpeta `templates/` (o `manifests/` para ArgoCD), y su `Ingress` actualizado con sección `tls:` apuntando al Secret que genera ese Certificate.

Patrón repetido (ejemplo Grafana):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: grafana-tls
  namespace: monitoring
spec:
  secretName: grafana-tls-secret
  dnsNames:
    - grafana.lab.local
  issuerRef:
    name: lab-ca-issuer
    kind: ClusterIssuer
```

```yaml
# Ingress correspondiente, con tls: añadido
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - grafana.lab.local
      secretName: grafana-tls-secret
  rules:
    - host: grafana.lab.local
      ...
```

Replicado para: Grafana, Prometheus, Alertmanager (en `charts/monitoring/templates/`), ArgoCD (en `manifests/argocd-ingress/`), Dashboard (en `charts/kubernetes-dashboard/templates/`, sustituyendo el `Secret` manual de `openssl` original) y Harbor (en `charts/harbor/templates/`, preparado aunque la app esté apagada — el Certificate no se emite hasta que el namespace exista).

## 4. Verificación

```bash
kubectl get clusterissuer
kubectl get certificate -A
```

Todos los `Certificate` deben mostrar `READY: True`. Si alguno no aparece tras el sync, forzar refresh de la Application correspondiente:

```bash
kubectl patch application <nombre-app> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

## 5. Acceso HTTPS

Todas las apps pasan a servir por el puerto TLS del ingress-nginx controller (**31443**), en vez de HTTP (31080):

- `https://grafana.lab.local:31443`
- `https://prometheus.lab.local:31443`
- `https://alertmanager.lab.local:31443`
- `https://argocd.lab.local:31443`
- `https://dashboard.lab.local:31443`
- `https://harbor.lab.local:31443` (cuando Harbor esté activo)

> El navegador mostrará aviso de "No es seguro" en todas — es esperado, porque `lab-ca-issuer` es una CA autofirmada propia que el sistema operativo no conoce por defecto. Es aceptable seguir en cada acceso, o instalar la CA como confiable:

```bash
kubectl get secret lab-ca-secret -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d > lab-ca.crt
```

En Windows: importar `lab-ca.crt` en `certmgr.msc` → "Entidades de certificación raíz de confianza" → Importar. Reiniciar el navegador tras importar. `lab-ca.crt` está en `.gitignore` — nunca se sube al repo.

## 6. Flujo de despliegue (GitOps)

Igual que el resto del repo: `develop → PR → main`. `cert-manager`, `cert-manager-ca`, `argocd-ingress`, `harbor` tienen auto-sync activo; `monitoring` requiere sync manual (`syncPolicy.automated.enabled: false` a propósito).

```bash
git checkout develop
# ... cambios ...
git add -A
git commit -m "mensaje descriptivo"
git push origin develop
gh pr create --base main --head develop --title "..." --body "..."
gh pr merge --merge <numero-pr>
```