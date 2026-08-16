# Disaster Recovery - Recrear el entorno completo (lab CKA)

Guía paso a paso para reconstruir todo el entorno (ArgoCD + Harbor + Kubernetes Dashboard + Monitoring/Grafana/Alertmanager con alertas a Telegram) tras destruir y recrear el cluster kubeadm de lab desde cero.

Filosofía: casi todo vive en git (`charts/`, `apps/`) y se recupera solo vía GitOps. Solo unos pocos pasos manuales son necesarios porque dependen de infraestructura nueva (IPs, credenciales generadas).

## 0. Requisitos previos

- Cluster kubeadm nuevo ya creado y `Ready` (`kubectl get nodes`)
- Acceso SSH/consola al droplet para poder ver la IP pública del master y worker

## 1. Configurar acceso al cluster nuevo (kubeconfig)

El `kubeconfig` **nunca se sube a git** (contiene credenciales de admin del cluster). Está protegido en `.gitignore`.

```bash
touch /workspaces/cka/kubeconfig.yaml
```

Abre `/workspaces/cka/kubeconfig.yaml` en el editor y pega el contenido completo del kubeconfig del cluster nuevo (se obtiene del propio proceso de creación del cluster, o de `/etc/kubernetes/admin.conf` en el nodo master vía SSH). Guarda.

```bash
export KUBECONFIG=/workspaces/cka/kubeconfig.yaml
kubectl get nodes -o wide
```

Anota las IPs que salgan aquí (`INTERNAL-IP` de master y worker) — se usan en los pasos siguientes.

## 2. Sincronizar el repo

```bash
git -C /workspaces/cka checkout main
git -C /workspaces/cka pull origin main
```

## 3. Instalar ArgoCD vía Helm

Se instala directamente sobre `main` (solo lectura del `values.yaml` existente, no hay nada nuevo que commitear).

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

helm install argocd argo/argo-cd \
  --namespace argocd \
  -f /workspaces/cka/argocd/values.yaml

kubectl get pods -n argocd -w
```

Espera a que todos los pods estén `Running` (Ctrl+C para salir del `-w`).

## 4. Aplicar la App of Apps

Única `Application` que se aplica a mano — el resto (`harbor`, `kubernetes-dashboard`, `monitoring`) las descubre `root-app` automáticamente desde `apps/`.

```bash
kubectl apply -f /workspaces/cka/apps/root-app.yaml

sleep 15
kubectl get applications -n argocd
```

## 5. Acceso a ArgoCD

> ⚠️ Con `server.insecure: true` en `argocd/values.yaml`, el servidor sirve HTTP plano — usar el puerto **80/NodePort HTTP**, no el 443/HTTPS. Acceder por `https://` al puerto TLS da `Connection reset by peer` porque el contenedor nunca abre TLS ahí.

```bash
kubectl get svc argocd-server -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Accede a `http://<IP-del-nodo>:<nodePort-80>` (revisar el puerto exacto en la salida del `get svc`; normalmente `30080` en este lab). Usuario `admin`, contraseña la que devuelva el segundo comando.

## 6. Actualizar `externalURL` de Harbor con la IP nueva

El `externalURL` en `charts/harbor/values.yaml` tiene la IP del droplet anterior — hay que actualizarla siguiendo el flujo normal `develop → PR → main`.

```bash
git -C /workspaces/cka checkout develop
git -C /workspaces/cka pull origin develop

sed -i 's|http://<IP-ANTIGUA>:30002|http://<IP-NUEVA>:30002|' /workspaces/cka/charts/harbor/values.yaml
cat /workspaces/cka/charts/harbor/values.yaml
```

Sube con el flujo estándar:

```bash
MSG="fix: actualizar externalURL harbor tras recrear cluster (nueva IP <IP-NUEVA>)" && git -C /workspaces/cka add -A && git -C /workspaces/cka commit -m "$MSG" && git -C /workspaces/cka push origin develop && gh pr create --base main --head develop --title "$MSG" --body "$MSG" && gh pr merge --merge $(gh pr view --json number -q .number) && git -C /workspaces/cka checkout main && git -C /workspaces/cka pull origin main
```

`harbor` tiene auto-sync activo, así que ArgoCD lo recoge solo. Si quieres acelerarlo:

```bash
kubectl patch application harbor -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
sleep 15
kubectl get application harbor -n argocd
kubectl get pods -n harbor
```

Es normal ver brevemente `harbor-jobservice` en `CrashLoopBackOff` o pods duplicados mientras el nuevo `harbor-core`/`harbor-registry` arrancan — se estabiliza solo en 1-2 minutos.

## 7. Sincronizar `monitoring` manualmente

`monitoring` tiene `syncPolicy.automated.enabled: false` a propósito — no se sincroniza solo ni con `root-app`, hay que forzarlo:

```bash
kubectl patch application monitoring -n argocd --type merge -p '{"operation":{"sync":{}}}'
sleep 20
kubectl get application monitoring -n argocd
kubectl get pods -n monitoring
```

## 8. Verificación final y accesos

```bash
kubectl get applications -n argocd
kubectl get pods -A | grep -v "kube-system\|kube-flannel\|kube-node-lease\|kube-public"
```

Todas las `Applications` deberían estar `Synced`/`Healthy`. Accesos (sustituir `<IP>` por la IP del nodo):

| Servicio | URL | Notas |
|---|---|---|
| ArgoCD | `http://<IP>:30080` | usuario `admin`, password del Secret |
| Harbor | `http://<IP>:30002` | usuario `admin`, password en `values.yaml` |
| Grafana | `http://<IP>:30300` | usuario `admin` / `admin12345` |
| Prometheus | `http://<IP>:30900` | sin auth |
| Alertmanager | `http://<IP>:30903` | sin auth |
| Kubernetes Dashboard | `kubectl get svc -n kubernetes-dashboard` para ver el puerto | requiere token: `kubectl -n kubernetes-dashboard create token kubernetes-dashboard` |

Las alertas de Telegram (`HarborPodNotReady`) siguen operativas sin tocar nada — la config vive en `charts/monitoring/templates/` y se recupera junto con `monitoring`.

## 9. Nota de seguridad — `kubeconfig.yaml`

`kubeconfig.yaml` está en `.gitignore`. **Nunca debe commitearse** — contiene el certificado y clave privada de administrador del cluster completo. Si alguna vez aparece en `git status` como fichero para trackear, algo ha ido mal con el `.gitignore`: revisar antes de hacer `git add -A`.

Si por error llegara a subirse a un commit real (no solo working tree), no basta con borrarlo del último commit — hay que:
1. Eliminarlo del historial completo (`git filter-repo` o similar)
2. Rotar las credenciales del cluster (regenerar certificados de admin), ya que el certificado quedaría comprometido