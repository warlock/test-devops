### Test DevOps

---

### Parte 1

He creado un cluster de kubernetes en Google Cloud.

Descargando llaves para "kubectl":

```sh
gcloud container clusters get-credentials {NOMBRE_DEL_CLUSTER} --zone {ZONA} --project {NOMBRE_DEL_PROYECTO}
```

Instalación de Homebrew para gestionar software en Mac

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

Instalacion de Helm para simplificar el despliegue.

```sh
brew install helm
```

#### Creando un StorageClass "custom"

```sh
kubectl patch storageclass gitlab-st -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### Almacenamiento optimo para hacer pruebas:

Documento gitlab-storage.yml

```yaml
gitlab:
  gitaly:
    persistence:
      storageClass: gitlab-st
      size: 10Gi
postgresql:
  persistence:
    storageClass: gitlab-st
    size: 10Gi
minio:
  persistence:
    storageClass: gitlab-st
    size: 10Gi
redis:
  master:
    persistence:
      storageClass: gitlab-st
      size: 5Gi
```

#### Creando un namespace

kubectl create namespace gitlab

#### Creando nodo de GitLab

```sh
helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm upgrade --namespace gitlab \
  --install gitlab gitlab/gitlab \
  --timeout 600s \
  --set global.edition=ce \
  --set global.hosts.domain=git.girona.dev \
  --set certmanager-issuer.email=josep@girona.dev \
  --set gitlab-runner.runners.privileged=true \
  -f gitlab-storage.yml
```
