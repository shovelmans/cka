# Harbor - Chart Helm propio con dependencia (lab CKA)

Chart Helm propio, tipo *umbrella* (sin plantillas manuales, delegando en el chart oficial como `dependency`), para desplegar Harbor gestionado vía ArgoCD con el patrón App of Apps.

## Por qué dependencia y no plantillas propias

Harbor es una plataforma multi-componente (core, portal, registry, jobservice, database/PostgreSQL, redis, nginx). Reescribir manualmente cada Deployment/Service/Secret con su cableado interno (igual que se hizo con `apache-demo`) no es viable ni recomendable: alto riesgo de errores sutiles y ninguna ventaja real frente al chart oficial, ya validado y mantenido por el proyecto.

**Regla práctica adoptada en este repo:** plantillas propias para apps de 1 componente (`apache-demo`); dependencia al chart oficial para apps multi-componente con estado (`harbor`).

## Estructura del chart

```
charts/harbor/
├── Chart.yaml       # declara la dependencia al chart oficial
├── Chart.lock        # versión resuelta y fijada de la dependencia
├── values.yaml        # overrides propios
└── charts/
    └── harbor-1.19.2.tgz   # dependencia empaquetada, generada por `helm dependency update`
```

## 1. Chart.yaml

```yaml
apiVersion: v2
name: harbor
description: Wrapper chart para Harbor (lab CKA)
type: application
version: 0.1.0
appVersion: "2.15.2"
dependencies:
  - name: harbor
    version: "1.19.2"
    repository: https://helm.goharbor.io
```

## 2. values.yaml

Los overrides van anidados bajo la clave `harbor:` (el nombre de la dependencia).

```yaml
harbor:
  expose:
    type: nodePort
    nodePort:
      ports:
        http:
          port: 80
          nodePort: 30002
    tls:
      enabled: false

  externalURL: http://142.93.131.69:30002

  harborAdminPassword: "Harbor12345"

  persistence:
    enabled: false

  trivy:
    enabled: false
  notary:
    enabled: false
```

> ⚠️ `externalURL` fija la IP pública del nodo. Si el cluster se destruye y se recrea (nuevo droplet), la IP cambia y hay que actualizar este valor a mano (o usar una Reserved IP de DigitalOcean para evitarlo).

## 3. Resolver la dependencia

Paso obligatorio antes de subir el chart al repo — ArgoCD no ejecuta `helm dependency update` por defecto:

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
helm dependency update charts/harbor
```

Genera `Chart.lock` y descarga el `.tgz` en `charts/harbor/charts/`. **Ambos ficheros se versionan en git** — sin el `.tgz`, ArgoCD no tiene la dependencia resuelta al hacer el sync.

## 4. Validación local

```bash
helm template harbor charts/harbor --namespace harbor
helm lint charts/harbor
```

`[WARNING] templates/: directory does not exist` es esperado en un chart tipo dependencia pura, sin plantillas propias — no bloquea el despliegue.

## 5. Application de ArgoCD (`apps/harbor-app.yaml`)

Descubierta automáticamente por `root-app` (patrón App of Apps) al hacer merge a `main`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: harbor
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shovelmans/cka.git
    targetRevision: main
    path: charts/harbor
    helm:
      releaseName: harbor
  destination:
    server: https://kubernetes.default.svc
    namespace: harbor
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

## 6. Componentes desplegados

| Pod | Función |
|---|---|
| `harbor-core` | API principal |
| `harbor-portal` | UI web |
| `harbor-registry` | almacenamiento OCI de imágenes |
| `harbor-jobservice` | tareas async (replicación, GC) |
| `harbor-database` | PostgreSQL |
| `harbor-redis` | caché/colas |
| `harbor-nginx` | proxy de entrada |

`trivy` (escaneo de vulnerabilidades) y `notary` quedaron deshabilitados para simplificar el lab.

## 7. Comportamiento observado en el arranque

`harbor-jobservice` reinició 3 veces con `Running` pero `0/1 Ready` en los primeros ~90 segundos, sin errores en los logs (`API server is serving at 8080` aparecía correctamente). Causa: el `readinessProbe` (`delay=20s`, `period=10s`, `failure=3`) fallaba mientras `core` y `database` terminaban de inicializarse. Se estabilizó solo, sin intervención manual — comportamiento normal de arranque en cadena entre componentes de Harbor, no una incidencia real.

## 8. Acceso

```bash
kubectl get svc -n harbor
```

`http://<IP-del-nodo>:30002`
Usuario: `admin`
Contraseña: la definida en `harborAdminPassword` (cambiar tras el primer login).

## 9. Flujo de despliegue (GitOps)

```bash
git checkout develop
# ... cambios en charts/harbor/ o apps/harbor-app.yaml ...
# si se toca la dependencia: helm dependency update charts/harbor (y versionar el .tgz + Chart.lock)
git add -A
git commit -m "mensaje descriptivo"
git push origin develop
gh pr create --base main --head develop --title "..." --body "..."
gh pr merge --merge <numero-pr>
```

`root-app` detecta el cambio en `main` y sincroniza solo (`selfHeal: true`, `prune: true`).