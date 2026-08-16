# ArgoCD - Despliegue en cluster de lab (CKA)

Despliegue de ArgoCD vía Helm sobre el cluster kubeadm de lab (`k8s-master-1786866631` + `k8s-worker-1786866631`, v1.31.14), expuesto por NodePort.

## Requisitos previos

- Cluster kubeadm accesible con `kubectl`
- `helm` instalado
- Trabajo realizado sobre la rama `develop` (merge a `main` vía MR al finalizar)

## 1. Añadir el repo de Helm de Argo

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-cd --versions | head -5
```

## 2. Values del chart (`argocd/values.yaml`)

```yaml
## values.yaml - ArgoCD (lab CKA)
global:
  domain: argocd.local

configs:
  params:
    server.insecure: true   # sin TLS propio, exposición vía NodePort en lab

server:
  replicas: 1
  service:
    type: NodePort

controller:
  replicas: 1

repoServer:
  replicas: 1

applicationSet:
  enabled: true

redis:
  enabled: true

dex:
  enabled: false   # no necesitamos SSO externo en el lab
```

## 3. Namespace y verificación previa (dry-run)

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

helm install argocd argo/argo-cd \
  --namespace argocd \
  -f argocd/values.yaml \
  --dry-run --debug
```

## 4. Instalación real

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  -f argocd/values.yaml

kubectl get pods -n argocd -w
```

## 5. Acceso a la UI

```bash
# Puertos NodePort asignados
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'; echo
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}'; echo

# IP de un nodo del cluster
kubectl get nodes -o wide

# Contraseña inicial de admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Accede a `https://<IP-del-nodo>:<nodePort-https>` (o `http://<IP-del-nodo>:<nodePort-http>`) con usuario `admin` y la contraseña obtenida. El certificado es autofirmado: acepta el aviso del navegador.

## 6. Limpieza del secret inicial (opcional, recomendado)

> ⚠️ Destructivo: borra el secret con la contraseña inicial en texto plano. No afecta a la sesión activa de `admin`, solo elimina ese secret concreto.

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```