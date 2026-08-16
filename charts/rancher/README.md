# Rancher Server - Chart Helm propio con dependencia (lab CKA)

Despliegue de Rancher Server vía Helm (patrón dependencia) gestionado por ArgoCD, con TLS emitido por la CA propia del cluster (cert-manager).

## Por qué dependencia

Mismo criterio que Harbor/Monitoring/cert-manager: aplicación multi-componente y compleja del propio fabricante, se usa el chart oficial `rancher-stable/rancher` como dependencia.

## Estructura del chart

```
charts/rancher/
├── Chart.yaml
├── Chart.lock
├── values.yaml
├── charts/
│   └── rancher-2.14.3.tgz
└── templates/
    ├── rancher-tls.yaml
    └── bootstrap-secret-override.yaml
```

## 1. Chart.yaml

```yaml
apiVersion: v2
name: rancher
description: Wrapper chart para Rancher Server (lab CKA)
type: application
version: 0.1.0
appVersion: "v2.14.3"
dependencies:
  - name: rancher
    version: "2.14.3"
    repository: https://releases.rancher.com/server-charts/stable
```

## 2. values.yaml

```yaml
rancher:
  hostname: rancher.lab.local
  replicas: 1
  ingressClassName: nginx
  bootstrapPassword: ""
  ingress:
    ingressClassName: nginx
    tls:
      source: secret
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
```

> `bootstrapPassword: ""` a propósito — ver troubleshooting abajo.

## 3. TLS con la CA propia del cluster

Rancher espera un Secret con nombre fijo `tls-rancher-ingress` en su propio namespace (`cattle-system`), gestionado vía `Certificate` de cert-manager igual que el resto de apps:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-rancher-ingress
  namespace: cattle-system
spec:
  secretName: tls-rancher-ingress
  dnsNames:
    - rancher.lab.local
  issuerRef:
    name: lab-ca-issuer
    kind: ClusterIssuer
```

## 4. Resolver la dependencia

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm dependency update charts/rancher
```

## 5. Application de ArgoCD (`apps/rancher-app.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rancher
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: charts/rancher
    helm:
      releaseName: rancher
  destination:
    server: https://kubernetes.default.svc
    namespace: cattle-system
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

## 6. Troubleshooting real: bootstrap-secret y hooks de Helm en ArgoCD

### Síntoma

Tras el primer sync, el pod `rancher-xxx` quedaba en `CreateContainerConfigError`:

```
Warning  Failed  kubelet  spec.containers{rancher}: Error: secret "bootstrap-secret" not found
```

### Investigación

El chart oficial de Rancher genera el Secret `bootstrap-secret` (con la contraseña de `bootstrapPassword`) mediante una plantilla marcada con **hooks de Helm**:

```yaml
annotations:
  "helm.sh/hook": pre-install,pre-upgrade
  "helm.sh/hook-weight": "-5"
  "helm.sh/resource-policy": keep
```

`kubectl get application rancher -n argocd -o jsonpath='{.status.operationState.syncResult.resources}'` confirmó que el hook **sí se ejecutó y creó el Secret con éxito** (`"hookPhase": "Succeeded"`, `"message": "bootstrap-secret created"`) — pero el Secret **no existía después** en el cluster (`kubectl get secret` → `NotFound`).

Causa: ArgoCD traduce los hooks `pre-install`/`pre-upgrade` de Helm a sus propios `PreSync` hooks. Aunque el chart incluye `"helm.sh/resource-policy": keep` (pensado para que Helm CLI no borre el recurso tras el hook), ArgoCD no respeta esa anotación de la misma forma — el recurso se trató como transitorio y se limpió tras completarse el hook.

### Intentos fallidos

1. **Crear un Secret propio adicional** (mismo nombre, sin marcarlo como hook) → generó `RepeatedResourceWarning: Resource /Secret/cattle-system/bootstrap-secret appeared 2 times` — ArgoCD detectó dos definiciones del mismo recurso (una del chart vía hook, otra nuestra) y no las reconcilió de forma predecible; el hook seguía ganando y borrando el Secret tras su ejecución.

2. **Forzar sync repetidamente** (`kubectl patch application ... sync`) → la `Application` quedaba reportando `Synced` sin haber aplicado realmente el cambio; solo un **hard refresh explícito** (`argocd.argoproj.io/refresh: hard`) forzó la reconciliación real.

### Solución

Dos cambios combinados:

1. **Vaciar `bootstrapPassword` en `values.yaml`** (`bootstrapPassword: ""`) — la condición interna del chart (`{{- if $bootstrapPassword }}`) deja de renderizar el Secret vía hook por completo. Verificado con:
   ```bash
   helm template rancher charts/rancher --namespace cattle-system | grep -c "helm.sh/hook.*pre-install"
   # → 0
   ```

2. **Crear el Secret nosotros mismos como recurso normal** (sin anotaciones de hook de Helm), usando el `sync-wave` nativo de ArgoCD para garantizar que se aplique antes que el Deployment:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: bootstrap-secret
     namespace: cattle-system
     annotations:
       argocd.argoproj.io/sync-wave: "-10"
   type: Opaque
   stringData:
     bootstrapPassword: "Rancher12345"
   ```

Con ambos cambios, solo existe una definición del Secret, gestionada de forma predecible por ArgoCD (sin el ciclo de vida especial de los hooks de Helm), y persiste correctamente en el cluster.

### Lección para el examen

Los hooks de Helm (`pre-install`, `post-install`, etc.) no siempre se comportan igual bajo ArgoCD que bajo `helm install` directo — cuando un recurso crítico (Secret, ConfigMap) depende de un hook y el pod que lo consume falla con "not found" a pesar de que el hook reporta éxito, sospechar de una política de limpieza de hooks divergente entre el gestor GitOps y Helm CLI. La solución robusta es sacar ese recurso del ciclo de hooks y gestionarlo como un recurso normal con orden explícito (`sync-wave`).

## 7. Acceso

```bash
kubectl get pods -n cattle-system
kubectl get ingress -n cattle-system
```

`https://rancher.lab.local:31443`
Usuario: `admin`
Contraseña de bootstrap: `Rancher12345`

Primer acceso pide confirmar la Server URL (autodetectada como `https://rancher.lab.local:31443`) y aceptar los términos antes de completar el login.

## 8. Recursos

Rancher es el componente más pesado del lab hasta ahora (`requests: 250m CPU / 512Mi`, `limits: 500m CPU / 1Gi`). Si el droplet se queda corto de recursos con varias apps activas a la vez (Harbor + Monitoring + Rancher), vigilar pods en `Pending` con `kubectl describe pod` para confirmar falta de CPU/memoria en los nodos.

## 9. Flujo de despliegue (GitOps)

```bash
git checkout develop
# ... cambios ...
git add -A
git commit -m "mensaje descriptivo"
git push origin develop
gh pr create --base main --head develop --title "..." --body "..."
gh pr merge --merge <numero-pr>
```

`rancher` tiene auto-sync activo. Si un cambio no se aplica pese a `Synced`, forzar hard refresh:

```bash
kubectl patch application rancher -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```