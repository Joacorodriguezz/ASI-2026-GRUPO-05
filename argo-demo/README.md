# Caso Práctico — Argo CD + GitOps

Aplicación nginx desplegada en un clúster local (Minikube) y sincronizada por Argo CD a partir de este repositorio. Git es la **fuente de la verdad**: cualquier cambio que se commitee acá se refleja solo en el clúster.

## Estructura

```
app/                  -> manifiestos que Argo CD sincroniza
  namespace.yaml
  configmap.yaml      -> página HTML ("Versión 1")
  deployment.yaml     -> nginx, 1 réplica
  service.yaml        -> NodePort 30080
argocd/
  application.yaml    -> definición de la Application de Argo CD
```

## Requisitos

Docker Desktop (o Docker Engine), Minikube y kubectl. En Windows se instalan fácil con:

```
winget install Docker.DockerDesktop
winget install Kubernetes.minikube
winget install Kubernetes.kubectl
```

En Mac: `brew install minikube kubectl` (más Docker Desktop). Recomendado: 4 GB de RAM libres para el clúster.

---

## Paso 1 — Levantar el clúster local

```
minikube start --driver=docker --memory=4096 --cpus=2
kubectl get nodes
```

Tiene que aparecer un nodo `minikube` en estado `Ready`.

## Paso 2 — Instalar Argo CD

```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Esperar a que todos los pods estén `Running` (tarda 1–3 minutos):

```
kubectl get pods -n argocd -w
```

(Ctrl+C para salir del modo watch.)

## Paso 3 — Acceder a la interfaz web

En una terminal aparte (queda abierta mientras se use la UI):

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Entrar a **https://localhost:8080** (aceptar la advertencia del certificado). Usuario: `admin`.

Contraseña inicial — Linux/Mac/Git Bash:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Windows PowerShell:

```
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```

📸 Captura: pantalla de login y dashboard vacío.

## Paso 4 — Preparar el repositorio

1. Crear un repo **público** en GitHub (ej. `argo-demo`).
2. Subir el contenido de esta carpeta a la rama `main`.
3. Editar `argocd/application.yaml` y poner la URL real del repo en `repoURL`.

📸 Captura: el repo en GitHub con la carpeta `app/`.

## Paso 5 — Crear la Application en Argo CD

Opción A (recomendada, por YAML):

```
kubectl apply -f argocd/application.yaml
```

Opción B (desde la UI, más visual para mostrar): **+ NEW APP** → Name `demo-nginx`, Project `default`, Sync Policy `Automatic` (tildar *Prune Resources* y *Self Heal*), Repository URL = el repo, Revision `main`, Path `app`, Cluster `https://kubernetes.default.svc`, Namespace `demo`, tildar *Auto-Create Namespace* → **CREATE**.

## Paso 6 — Verificar el despliegue

En la UI la app tiene que quedar **Synced** y **Healthy**. Por consola:

```
kubectl get all -n demo
```

Para abrir la app en el navegador:

```
minikube service demo-nginx -n demo
```

(o `kubectl port-forward svc/demo-nginx -n demo 8081:80` y entrar a http://localhost:8081). Tiene que verse la página "Versión 1".

📸 Captura: árbol de recursos en Argo CD y la página en el navegador.

## Paso 7 — Demostrar la reconciliación automática ⭐

**Demo A — escalar réplicas:** en GitHub editar `app/deployment.yaml`, cambiar `replicas: 1` por `replicas: 3` y hacer commit. En Argo CD se ve cómo aparecen 2 pods nuevos. Verificar con `kubectl get pods -n demo`.

**Demo B — cambiar el contenido:** editar `app/configmap.yaml`, cambiar "Versión 1" por "Versión 2" y commitear. Tras la sincronización, recargar la página (el volumen del ConfigMap puede tardar hasta ~1 minuto en actualizarse dentro del pod).

**Demo C — self-heal (Git manda, no kubectl):** intentar cambiar el clúster a mano:

```
kubectl scale deployment demo-nginx -n demo --replicas=5
```

Argo CD detecta que el clúster se desvió de Git y lo vuelve a dejar como dice el repo. Es la mejor forma de mostrar que Git es la fuente de la verdad.

> **Tip para la exposición:** Argo CD revisa el repo cada ~3 minutos. Para no esperar en vivo, después del commit apretar **REFRESH** en la app.

📸 Captura: antes/después de cada demo.

---

## Problemas comunes

| Síntoma | Solución |
|---|---|
| `minikube start` falla | Verificar que Docker Desktop esté abierto |
| Pods de argocd en `Pending` | Poca RAM: `minikube delete` y volver a arrancar con `--memory=4096` |
| App en `Unknown` / error de repo | Revisar que el repo sea público y que `repoURL` y `path: app` sean correctos |
| No abre localhost:8080 | El `port-forward` se cortó: volver a ejecutarlo |
| Cambios en Git no se ven | Apretar REFRESH; confirmar que el commit fue a `main` |

## Limpieza

```
kubectl delete -f argocd/application.yaml
minikube delete
```
