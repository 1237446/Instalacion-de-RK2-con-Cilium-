# Nextcloud-con-cluster-en-kubernetes
Instalacion de Nextcloud en kubernetes usando clusters de postgresql y redis para alta disponibilidad

## Postgresql
Postgresql es la base de datos que usaremos para almacenar la informacion de Nextcloud ademas de los metadatos que se generen. para la generacion del cluster:

### CloudNativePG
Es un operador de código abierto para Kubernetes que admite la creacion clústeres PostgreSQL basados ​​en replicación de transmisión asincrónica y sincrónica para administrar múltiples réplicas en espera activa dentro del mismo clúster de Kubernetes, con las siguientes especificaciones:

- Un servidor principal, con múltiples réplicas opcionales en espera activa para alta disponibilidad 
- Servicios disponibles para aplicaciones:
  *  **rw:** las aplicaciones se conectan solo a la instancia principal del clúster
  *  **ro:** las aplicaciones se conectan solo a réplicas en espera activa para cargas de trabajo de solo lectura (opcional)
  *  **r:** las aplicaciones se conectan a cualquiera de las instancias para cargas de trabajo de solo lectura (opcional)

El siguiente diagrama proporciona una vista simplista de la arquitectura compartida recomendada para un clúster PostgreSQL

![guia](/pictures/k8s-pg-architecture.png)

#### Cargas de trabajo de lectura y escritura
Nextcloud usara la instancia maestra de PostgreSQL para las acciones de escritura/lectura de la base de datos, como se muestra en el siguiente diagrama:

![guia](pictures/architecture-rw.png)

> [!NOTE]
>Las aplicaciones usaran el servicio de sufijo **-rw**.

> [!TIP]
> En caso de indisponibilidad temporal o permanente del servidor principal, para fines de alta disponibilidad CloudNativePG activará una conmutación por error, apuntando el -rw servicio a otra instancia del clúster.

#### Cargas de trabajo de solo lectura
Nextcloud usara las instancias replicas de PostgreSQL para las acciones de lectura de la base de datos, como se muestra en el siguiente diagrama:

![guia](pictures/architecture-read-only.png)

> [!NOTE]
> Las aplicaciones usaran el servicio de sufijo **-ro**.

### Instalacion del cluster PostgreSQL
Para la instalacion usaremos el manifiesto YAML, de la documentacion

```
kubectl apply --server-side -f \
https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.26/releases/cnpg-1.26.0.yaml
```

Comprobamos la instalacion
```
kubectl get pods -n cnpg-system
```
```
NAME                                      READY   STATUS    RESTARTS      AGE
cnpg-controller-manager-6848689f4-l2ztv   1/1     Running   0             4m
```

Usamos el manifiesto YAML para la creacion del cluster
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secrets
type: kubernetes.io/basic-auth
data:
  username: bmV4dGNsb3Vk
  password: cm9vdHBhc3N3b3Jk

---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgresql-node
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.5
  primaryUpdateStrategy: unsupervised

  bootstrap:
    initdb:
      dataChecksums: true
      database: nextcloud
      owner: nextcloud
      secret:
        name: cluster-secrets

  resources:
    requests:
      memory: "512Mi"
      cpu: "1"
    limits:
      memory: "1Gi"
      cpu: "2"

  storage:
    size: 10Gi
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: standard
      volumeMode: Filesystem

  walStorage:
    size: 5Gi

  managed:
    services:
      ## disable the default services
      disabledDefaultServices: ["r"]
```
```
kubectl apply -f cluster-postgresql.yaml
```

Crearemos las copias de seguridad periódicas
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: postgresql-node-backup
  labels:
    app.kubernetes.io/component: database-backup
spec:
  schedule: "0 0 23/12 * * *"
  backupOwnerReference: none
  immediate: false
```

Verificamos el funcionamiento del cluster
```
kubectl get pods
```
```
NAME                               READY   STATUS    RESTARTS        AGE
postgresql-node-1                  1/1     Running   0               3m
postgresql-node-2                  1/1     Running   0               3m
postgresql-node-3                  1/1     Running   0               3m
```
```
kubectl get services
```
```
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
kubernetes                           ClusterIP   10.96.0.1        <none>        443/TCP     1d
postgresql-node-ro                   ClusterIP   10.97.85.79      <none>        5432/TCP    3m
postgresql-node-rw                   ClusterIP   10.111.110.188   <none>        5432/TCP    3m
```
> [!NOTE]
> Para mayor informacion del servicio, leer la documentacion [aqui](https://cloudnative-pg.io/documentation/1.26/).

## Redis
Redis es el serviodr de almacenamiento servira como almacenamiento cache de datos de navegacion dando mayor velocidad. para la generacion del cluster:

### OT Redis Operator
Es un operador que gestiona la configuración de Redis en modo independiente, clúster, replicación y centinela sobre Kubernetes. Permite crear una configuración de clúster de Redis con las mejores prácticas.

El operador de Redis admite las siguientes estrategias de implementación para Redis:

- **Clúster:** Es simplemente una estrategia de fragmentación de datos . Particiona automáticamente los datos entre múltiples nodos de Redis. Es una función avanzada de Redis que logra almacenamiento distribuido y evita un punto único de fallo.
- **Replicación:** Utiliza replicación asíncrona, lo que significa que el nodo líder no espera a que los nodos seguidores apliquen los cambios antes de enviar nuevas actualizaciones. En su lugar, los nodos seguidores se ponen al día con el nodo líder tan pronto como pueden.
- **Sentinel:** Es una herramienta que proporciona conmutación por error y monitorización automáticas para los nodos de Redis.

Para el caso actual, usaremos el metodo de Replicacion,El siguiente diagrama proporciona una vista simplista de la arquitectura compartida

![guia](pictures/replication-redis.png)

### Instalacion del Cluster Redis
Añadimos el repositorio en helm y lo instalamos
```
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm install redis-operator ot-helm/redis-operator --namespace ot-operators --create-namespace 
```
Comprobamos la instalacion
```
kubectl get pods -n ot-operators
```
```
NAME                             READY   STATUS    RESTARTS      AGE
redis-operator-bb784b6df-4pfxt   1/1     Running   0             4m
```
Usamos el manifiesto YAML para la creacion del cluster
```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisSentinel
metadata:
  name: redis-sentinel
spec:
  clusterSize: 3
  podSecurityContext:
    runAsUser: 1000
    fsGroup: 1000
  redisSentinelConfig:
    redisReplicationName: redis-replication
    redisReplicationPassword:
      secretKeyRef:
        name: password-nextcloud
        key: redis_password
  kubernetesConfig:
    image: quay.io/opstree/redis-sentinel:v7.0.15
    imagePullPolicy: IfNotPresent
    redisSecret:
      name: password-nextcloud
      key: redis_password
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
---
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisReplication
metadata:
  name: redis-replication
spec:
  clusterSize: 3
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.15
    imagePullPolicy: IfNotPresent
    redisSecret:
      name: password-nextcloud
      key: redis_password
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: nfs-client
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
  redisExporter:
    enabled: false
    image: quay.io/opstree/redis-exporter:v1.44.0
  podSecurityContext:
    runAsUser: 1000
    fsGroup: 1000
```
```
kubectl apply -f cluster-redis.yaml
```

Verificamos el funcionamiento del cluster
```
kubectl get pods
```
```
NAME                        READY   STATUS    RESTARTS   AGE
redis-replication-0         1/1     Running   0             15m
redis-replication-1         1/1     Running   0             15m
redis-replication-2         1/1     Running   0             15m
redis-sentinel-sentinel-0   1/1     Running   0             14m
redis-sentinel-sentinel-1   1/1     Running   0             14m
redis-sentinel-sentinel-2   1/1     Running   0             14m
```
```
kubectl get services
```
```
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
kubernetes                           ClusterIP      10.96.0.1        <none>         443/TCP        30m
redis-replication                    ClusterIP      10.111.8.17      <none>         6379/TCP       16m
redis-replication-additional         ClusterIP      10.104.207.9     <none>         6379/TCP       16m
redis-replication-headless           ClusterIP      None             <none>         6379/TCP       16m
redis-replication-master             ClusterIP      10.98.144.110    <none>         6379/TCP       16m
redis-replication-replica            ClusterIP      10.99.112.250    <none>         6379/TCP       16m
redis-sentinel-sentinel              ClusterIP      10.111.52.25     <none>         26379/TCP      16m
redis-sentinel-sentinel-additional   ClusterIP      10.97.14.223     <none>         26379/TCP      16m
redis-sentinel-sentinel-headless     ClusterIP      None             <none>         26379/TCP      16m
```
> [!NOTE]
> Para mayor informacion del servicio, leer la documentacion [aqui](https://ot-redis-operator.netlify.app/docs/overview/).

## Nextcloud
Es la aplicacion principal la cual es un software de código abierto para crear y utilizar servicios de alojamiento de archivos y colaboración en la nube

### Componentes PHP - Apache
