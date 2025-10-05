# Instalacion de RK2 con Cilium

Para instalar **RKE2** con **Cilium** y un Ingress controller, el proceso se divide en dos fases principales: prepararacion del nodo e instalacion de componentes.

## Nodo Mestro

### Fase 1: Preparación del Nodo

1.  **Crea el archivo de configuración de RKE2**: Antes de instalar RKE2, necesitas decirle que no use sus componentes predeterminados. Para ello, crea un directorio y un archivo de configuración.

    ```sh
    sudo mkdir -p /etc/rancher/rke2/
    sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
    cni: "none"
    disable:
    - rke2-ingress-nginx
    disable-kube-proxy: true
    EOF
    ```

      * `cni: "none"`: Le dice a RKE2 que no instale ningún CNI por defecto.
      * `disable: - rke2-ingress-nginx`: Desactiva el controlador NGINX Ingress que viene incluido con RKE2.
      * `disable-kube-proxy: true`: Deshabilita el `kube-proxy` de Kubernetes, ya que Cilium se encargará de esta función.

2.  **Instala RKE2**: Descarga y ejecuta el script de instalación.

    ```sh
    curl -sfL https://get.rke2.io | sudo sh -
    ```

    Una vez que el script finalice, se instalarán los servicios necesarios.

3.  **Inicia el servicio de RKE2**: Inicia el servicio y habilítalo para que se inicie automáticamente en cada reinicio.

    ```sh
    sudo systemctl enable --now rke2-server.service
    ```

    En este punto, el clúster estará en funcionamiento, pero los nodos estarán en estado `NotReady` porque aún no tienen un CNI.

-----

### Fase 2: Instalación de Cilium e Ingress

1.  **Copia el `kubeconfig`**: Para poder interactuar con el clúster, necesitas acceder al archivo de configuración.

    ```sh
    mkdir -p ~/.kube
    sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
    sudo chown $(id -u):$(id -g) ~/.kube/config
    ```

2.  **Instala Cilium con Helm**: Cilium es la mejor opción para la gestión de la red. Utilizar la guia oficial te permite una instalación y configuración sencillas.

      * Primero, agrega el CLI de Cilium.
        ```sh
        CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
        CLI_ARCH=amd64
        if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
        curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
        sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
        sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
        rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
        ```
      * Luego, instala Cilium
        ```sh
        cilium install
        ```

      * Para validar que Cilium se ha instalado correctamente, puede ejecutar
        ```sh
        $ cilium status --wait
           /¯¯\
        /¯¯\__/¯¯\    Cilium:         OK
        \__/¯¯\__/    Operator:       OK
        /¯¯\__/¯¯\    Hubble:         disabled
        \__/¯¯\__/    ClusterMesh:    disabled
           \__/
        
        DaemonSet         cilium             Desired: 2, Ready: 2/2, Available: 2/2
        Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
        Containers:       cilium-operator    Running: 2
                          cilium             Running: 2
        Image versions    cilium             quay.io/cilium/cilium:v1.9.5: 2
                          cilium-operator    quay.io/cilium/operator-generic:v1.9.5: 2
        ```
      

3.  **Instala el Ingress Controller (NGINX)**: Aunque desactivaste el de RKE2, necesitas instalar uno para gestionar el tráfico externo.

      * Agrega el repositorio de Helm de NGINX Ingress Controller.
        ```sh
        helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
        helm repo update
        ```
      * Instala el controlador.
        ```sh
        helm install ingress-nginx ingress-nginx/ingress-nginx \
        --namespace ingress-nginx \
        --create-namespace
        ```
      * Verifica que el pod de NGINX Ingress se esté ejecutando.
        ```sh
        kubectl get pods -n ingress-nginx
        ```

## Nodo esclavo

Para agregar dos nodos (esclavo) más a tu clúster **RKE2** existente, debes generar un token de unión y luego usarlo para unir los nuevos nodos como agentes.

### Paso 1: Obtener el token del servidor

En el nodo donde instalaste RKE2 como servidor (el primer nodo), necesitas obtener el token de unión. Este token se encuentra en el archivo `/var/lib/rancher/rke2/server/node-token`.

Ejecuta el siguiente comando en el nodo servidor para mostrar el token:

```sh
sudo cat /var/lib/rancher/rke2/server/node-token
```

El token será una larga cadena de caracteres alfanuméricos.

### Paso 2: Configurar y unir los nuevos nodos

En cada uno de los dos nuevos nodos, debes crear un archivo de configuración para RKE2. Este archivo le dirá al nodo que se una al clúster como un agente y también le indicará que debe usar Cilium como CNI, al igual que el servidor.

1.  **Crea el directorio y el archivo de configuración**:

    ```sh
    sudo mkdir -p /etc/rancher/rke2/
    sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
    server: https://<DIRECCIÓN_IP_DEL_SERVIDOR>:9345
    token: <TU_TOKEN_OBTENIDO_ARRIBA>
    cni: "none"
    disable-kube-proxy: true
    EOF
    ```

      * **`<DIRECCIÓN_IP_DEL_SERVIDOR>`**: Reemplaza esto con la dirección IP del nodo donde instalaste RKE2 como servidor.
      * **`<TU_TOKEN_OBTENIDO_ARRIBA>`**: Reemplaza esto con el token que obtuviste en el paso anterior.
      * `cni: "none"` y `disable-kube-proxy: true`: Estas líneas son cruciales para asegurar que los nuevos nodos utilicen la misma configuración de red que el servidor principal.

2.  **Instala y ejecuta el agente de RKE2**:

      * En cada uno de los nuevos nodos, descarga y ejecuta el script de instalación, pero esta vez, se instalará como agente (`rke2-agent`).

    <!-- end list -->

    ```sh
    curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
    ```

      * Una vez que el script finalice, habilita y comienza el servicio del agente.

    <!-- end list -->

    ```sh
    sudo systemctl enable --now rke2-agent.service
    ```

    Después de un minuto o dos, los nodos se conectarán al servidor, Cilium se desplegará en ellos y se unirán al clúster.

### Paso 3: Verificar la unión de los nodos

Vuelve al nodo servidor y ejecuta el siguiente comando para ver el estado de los nodos del clúster:

```sh
kubectl get nodes
```

Deberías ver los tres nodos (el servidor y los dos nuevos agentes) en estado **`Ready`**.

## Añadir un Nodo Maestro (HA)

Para añadir un **nodo maestro** adicional a un clúster **RKE2** existente, debes instalar el servicio `rke2-server` en el nuevo nodo, apuntándolo al servidor inicial y utilizando el mismo token. Esto crea un clúster de **alta disponibilidad (HA)**.

### 1\. Preparación del Nuevo Servidor Maestro

De manera similar al primer nodo maestro, necesitas configurar el nuevo nodo maestro para que sepa cómo unirse al clúster y qué componentes deshabilitar.

#### Obtén la IP y el Token

  * **IP del Servidor Inicial:** Necesitas la dirección IP del primer nodo maestro que instalaste. Sustituiremos `<IP_DEL_SERVIDOR_INICIAL>` con esta IP.
  * **Token del Clúster:** Este es el token que usaste o generaste previamente, que se encuentra en `/var/lib/rancher/rke2/server/node-token` en el servidor inicial. Sustituiremos `<TU_TOKEN_OBTENIDO>` con este valor.

#### Crea el Archivo de Configuración

Crea el directorio y el archivo de configuración en el **nuevo servidor maestro**.

```bash
sudo mkdir -p /etc/rancher/rke2/
sudo tee /etc/rancher/rke2/config.yaml > /dev/null <<EOF
server: https://<IP_DEL_SERVIDOR_INICIAL>:9345
token: <TU_TOKEN_OBTENIDO>
cni: "none"
disable:
- rke2-ingress-nginx
disable-kube-proxy: true
EOF
```

  * **`server: https://<IP_DEL_SERVIDOR_INICIAL>:9345`**: Esta línea es **crucial**. Le dice al nuevo servidor que se una al plano de control existente a través de la IP del primer nodo maestro. El puerto por defecto es `9345`.
  * **`token: <TU_TOKEN_OBTENIDO>`**: Usa el token que obtuviste del servidor inicial.
  * Las opciones de `cni: "none"` y `disable-kube-proxy: true` son necesarias para mantener la coherencia con el primer servidor, que usa Cilium y deshabilitó el `kube-proxy`.

-----

### 2\. Instalación de RKE2 como nodo Maestro

En el **nuevo servidor maestro**, instala RKE2. A diferencia de los nodos agentes, no se especifica el tipo (`INSTALL_RKE2_TYPE="agent"`), por lo que se instalará como **master** por defecto.

```bash
curl -sfL https://get.rke2.io | sudo sh -
```

-----

### 3\. Inicia el Servicio de RKE2

Una vez completada la instalación, inicia y habilita el servicio del servidor RKE2 en el **nuevo nodo**.

```bash
sudo systemctl enable --now rke2-server.service
```

-----

### 4\. Verificación

El nuevo nodo se unirá al clúster como un nodo maestro. En el **nodo maestro inicial** o cualquier máquina con el `kubeconfig` configurado, verifica el estado de los nodos:

```bash
kubectl get nodes
```

Deberías ver el nuevo nodo con el rol **`control-plane`** o **`master`** (y quizás también `etcd`), y su estado eventualmente cambiará a **`Ready`** a medida que Cilium se despliegue en él y se sincronice con el plano de control. El plano de control de tu clúster RKE2 ahora será de **alta disponibilidad**.
