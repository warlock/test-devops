## DevOps

---

## Parte 1 / Parte 2 : Servicios

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

#### Almacenamiento óptimo para hacer pruebas:

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

#### Creando los "namespace" de GitLab y Jenkins

```sh
kubectl create namespace gitlab
kubectl create namespace jenkins
```

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

#### Instalación de Jenkins

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install jenkins "bitnami/jenkins" \
  --namespace jenkins \
  --set jenkinsUser=admin \
  --set jenkinsPassword=password \
  --set persistence.size="2Gi" \
  --set metrics.service.type=LoadBalancer
```

#### Permisos para que Jenkins pueda gestionar Kubernetes (Es mejor hacer un rol a medida)

```sh
kubectl create clusterrolebinding jenkins --clusterrole cluster-admin --serviceaccount=jenkins:default
```

#### Creando un LoadBalancer para el acceso remoto:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-svc
  namespace: jenkins
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/component: jenkins-master
    app.kubernetes.io/instance: jenkins
```

#### Pequeña web en Go con un healthcheck y con un test:

[https://github.com/warlock/jenkins-golang-play](https://github.com/warlock/jenkins-golang-play)

Esta es una pipeline de gitlab que ya hace el despliegue en kubernetes

[https://github.com/warlock/jenkins-golang-play/blob/master/.gitlab-ci.yml](https://github.com/warlock/jenkins-golang-play/blob/master/.gitlab-ci.yml)

En la pipeline de Jenkins aun no he echo el despliegue pero si funciona los test creando pod de kubernetes dinamicamente, el build y el push:

[https://github.com/warlock/jenkins-golang-play/blob/master/Jenkinsfile](https://github.com/warlock/jenkins-golang-play/blob/master/Jenkinsfile)

#### Script en Node.js para hacer pipeline en Jenkins a traves de la API:

[https://github.com/warlock/make-jenkins-pipeline](https://github.com/warlock/make-jenkins-pipeline)

El script es "createpipeline.js":

```sh
npm i
node createpipeline.js
```

#### Instalación y análisis de Fiddler

Fiddler es un proxy para filtrar y depurar conexiones entrantes y salientes de servicios HTTP. Es mucho más cómodo y organizado que otras herramientas como Wireshark. La interfaz muestra los datos del "request" y del "response" de forma más legible. También da la posibilidad de documentar API y probar " request". Está disponible para Windows, Linux y Mac.

## Parte 3 : Startup de videos

Dependiendo de los equipos de empresa los servicios/Lambda los crearía en Rust/Go para mejor velocidad/consumo de recursos o en Javascript/Typescript para tener un equipo que hable un solo lenguaje.

### Versión cómoda/rápida aprovechando al máximo los servicios de AWS(Aplicable a Azure/Google Cloud).

1. Sistemas de archivos:

   - Sistema de archivos para la entrega:
     Escogería AWS Elastic Block Storage y poco a poco refrescaría los archivos con menos éxito moviéndolos a S3 con distintos niveles de enfriamiento.

   - Sistema de archivos S3:

     - Procesamiento de videos.
     - Alojamiento para estáticos como el dashboard privado de la empresa
     - Alojamiento imágenes como pueden ser avatares o fotos de video.

2. Motor de búsqueda Elastic Search(AWS)
   Contraria el servicio de AWS. Ahí guardaría los "títulos/textos/usuarios" del video publicado para facilitar la búsqueda.

3. Base de datos DynamoDB:

   - Almacenaje de los datos variables de los videos como visitas, likes.
   - Almacenaje de la parte la lógica de negocio.

4. Lambda (Yo usaría serverless framework para organizar las lambda)
   En esta situación usaría prácticamente todas las combinaciones:

   - Colas:

     - Preparación del archivo adaptando la resolución/resoluciones en s3 para ser colocados posteriormente en el sistema de archivos.
     - Archivos de imagen como son avatares de distintos tamaños.
     - Envío de correos.

   - Tareas Cron:

     - Bajada o Subida de nivel de los sistemas de archivos según su uso.
     - Limpieza de cuentas viejas o cerradas de forma progresiva.

   - API Lambda con Cloudfront para respuestas más inmediatas y distribuidas.

   - Front de la web: Next.js con despliegue del framework Serverless(Lambda+Cloudfront+S3):
     Permite hacer estáticos de todo lo que no es dinámico, mantiene el SEO transformando las secciones que requieren de server side rendering en Lambdas autónomas. Posteriormente publica los lambda y assets en CloudFront.

5. Dashbird: Monitorización externa de servicios Lambda.

### Versión manual.

- Hadoop HDFS como sistema de archivos distribuido/escalable.
  Crería varios cluster dependiendo de la tipología de datos.

- Cluster Kubernetes(Evidentemente cada proceso con su ReplicaSet adaptado a la necesidad):

  - Ingress NGINX para Balancear la carga.
  - CronJob de kubernetes para gestionar las tareas: Limpieza de cuentas, mover archivos, compilación de Next.js estilo JAMSTACK de las zonas de la pagina menos dinámicas...
  - RabbitMQ para las colas de procesamiento.
  - Procesamiento de colas y CronJob en servicios GO/Rust para mejor aprovechamiento de recursos.
  - API en servicios GO/Rust para mejor aprovechamiento de recursos y para evitar al máximo los request perdidos.
  - ElaticSearch almacenando los datos en volúmenes rápidos (Dependiendo de la situación quizá lo separaría en otro cluster solo para el)
  - API/Tareas en Go/Rust para minimizar el consumo de hardware CPU/RAM. (Servicios de monitorización estilo Jaeger para visualizar el comportamiento de sistemas críticos).
  - Redis cache de web Next.js compilada previamente con JAMSTACK.
  - Next.js para recibir las peticiones mas dinámicas que requieren de SSR y SEO.

- Fuera del cluster
  - Kit de monitorización Grafana/Prometheus
  - NewRelic como proveedor externo
