# Estrategias de Lanzamiento con Knative Serving

—
_🌍 Disponible
en_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md) | [Español](README-es.md)
> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

Este tutorial creará un clúster de Kubernetes e instalará `Knative Serving` para implementar diferentes estrategias de
lanzamiento. Utilizaremos la división de tráfico basada en porcentajes de Knative Serving y el enrutamiento basado en
etiquetas y encabezados.

## Creando un Clúster de Kubernetes con Knative Serving

Necesitas un clúster de Kubernetes para instalar [Knative Serving](https://knative.dev). Puedes crear uno utilizando
Kubernetes KinD, pero en lugar de usar las configuraciones proporcionadas en
el [Capítulo 2](../../chapter-2/README-es.md), usaremos el siguiente
comando:

```shell
cat <<EOF | kind create cluster --name dev --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31080 # expose port 31380 of the node to port 80 on the host, later to be use by kourier or contour ingress
    listenAddress: 127.0.0.1
    hostPort: 80
EOF
```

Cuando utilizas Knative Serving, no es necesario instalar un Ingress Controller, ya que Knative Serving requiere una
pila de red más avanzada para habilitar funciones como el enrutamiento y la división de tráfico.
Instalaremos [Kourier](https://github.com/knative-extensions/net-kourier)
para esto, aunque también puedes instalar un service mesh completo como [Istio](https://istio.io/).

Once you have a Cluster, let's start by
installing [Knative Serving](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/). You can
follow the official documentation or copy the installation steps listed here, as the examples had been tested with this
version of Knative.

Una vez que tengas un clúster, comencemos
instalando [Knative Serving](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/). Puedes
seguir la documentación oficial o copiar
los pasos de instalación que se enumeran aquí, ya que los ejemplos han sido probados con esta versión de Knative.

Instalar las Definiciones de Recursos Personalizados de Knative Serving:

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.2/serving-crds.yaml
```

Luego, instala el Plano de Control de Knative Serving:

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.2/serving-core.yaml
```

Instala una pila de red compatible con Knative Serving. Aquí puedes instalar Istio, un service mesh completo, o opciones
más simples como Kourier, que utilizaremos aquí para limitar la utilización de recursos en tu clúster:

```shell
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.10.0/kourier.yaml
```

Configura Kourier como la pila de red seleccionada:

```shell
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

Finalmente, configura la resolución de DNS para tu clúster, de modo que pueda exponer URLs accesibles para nuestros
servicios:

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.2/serving-default-domain.yaml
```

**Solo para Knative en KinD**

Para que el DNS mágico de Knative funcione en KinD, necesitas parchear el siguiente ConfigMap:

```shell
kubectl patch configmap -n knative-serving config-domain -p "{\"data\": {\"127.0.0.1.sslip.io\": \"\"}}"
```

Y si instalaste la capa de red `kourier`, necesitas crear un ingreso:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: kourier-ingress
  namespace: kourier-system
  labels:
    networking.knative.dev/ingress-provider: kourier
spec:
  type: NodePort
  selector:
    app: 3scale-kourier-gateway
  ports:
    - name: http2
      nodePort: 31080
      port: 80
      targetPort: 8080
EOF
```

Se necesita un paso más para ejecutar los ejemplos cubiertos en la siguiente sección, y este es habilitar el
`tag-header-based-routing` y el acceso
a [Kubernetes Downward API](https://knative.dev/docs/serving/configuration/feature-flags/#kubernetes-downward-api) para
obtener información
sobre los pods que se están ejecutando en el clúster. Podemos ajustar la instalación de Knative Serving parcheando el
ConfigMap config-features (https://knative.dev/docs/serving/configuration/feature-flags/#feature-and-extension-flags):

```shell
kubectl patch cm config-features -n knative-serving -p '{"data":{"tag-header-based-routing":"Enabled", "kubernetes.podspec-fieldref": "Enabled"}}'
```

Después de todas estas configuraciones, deberíamos estar listos para usar Knative Serving en nuestro clúster KinD.

# Estrategias de Lanzamiento con Knative Serving

Antes de pasar a implementar diferentes estrategias de lanzamiento, necesitamos comprender los conceptos básicos de
Knative Serving. Para ello, definiremos un Servicio Knative para el Servicio de Notificaciones. Esto nos dará
experiencia práctica en el uso de Knative Serving y sus capacidades.

## Introducción rápida a los Servicios Knative

Knative Serving simplifica y amplía las capacidades ofrecidas por Kubernetes mediante el concepto de un Servicio
Knative. Un Servicio Knative utiliza la capa de red de Knative Serving para enrutar el tráfico a nuestras cargas de
trabajo sin obligarnos a definir recursos complejos de Kubernetes. Debido a que Knative Serving tiene acceso a la
información sobre cómo fluye el tráfico hacia nuestros servicios, puede entender la carga que están experimentando y
utilizar un escalador automático diseñado específicamente para aumentar o reducir nuestras instancias de servicio según
la demanda. Esto puede ser muy útil para los equipos de plataforma que buscan implementar un modelo de función como
servicio para sus cargas de trabajo, ya que Knative Serving puede reducir a cero los servicios que no están recibiendo
tráfico.

Los Servicios Knative también exponen una configuración simplificada que se asemeja a un modelo de Contenedores como
Servicio, como Google Cloud Run, Azure Container Apps y AWS App Runner, en el que al definir qué contenedor queremos
ejecutar, la plataforma se encargará del resto (sin configuraciones complejas para redes, enrutamiento de tráfico,
etc.).

Dado que el Servicio de Notificaciones utiliza Kafka para emitir eventos, necesitamos instalar Kafka usando Helm:

```shell
helm install kafka oci://registry-1.docker.io/bitnamicharts/kafka --version 22.1.5 --set "provisioning.topics[0].name=events-topic" --set "provisioning.topics[0].partitions=1" --set "persistence.size=1Gi" 
```

Verifica que Kafka esté en funcionamiento antes de continuar, ya que suele tardar un poco en obtener la imagen del
contenedor de Kafka y arrancarlo.

Si tienes un clúster de Kubernetes con Knative Serving instalado, puedes aplicar el siguiente recurso de Servicio
Knative para ejecutar una instancia de nuestro Servicio de Notificaciones:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: notifications-service
spec:
  template:
    spec:
      containers:
        - image: salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
          env:
            - name: KAFKA_URL
              value: kafka.default.svc.cluster.local
          ...<MORE ENVIRONMENT VARIABLES HERE>...  
```

Puedes aplicar este recurso ejecutando:

```shell
kubectl apply -f knative/notifications-service.yaml
```

Knative Serving creará una instancia de nuestro contenedor y configurará todas las configuraciones de red para
proporcionarnos una URL para acceder al servicio.

Puedes listar todos los Servicios Knative ejecutando el siguiente comando:

```shell
> kubectl get ksvc 
NAME                    URL                                                       LATESTCREATED                 LATESTREADY                   READY   REASON
notifications-service   http://notifications-service.default.127.0.0.1.sslip.io   notifications-service-00001   notifications-service-00001   True    

```

You can now curl the `service/info` URL of the service to make sure that it is working, we are using [
`jq`](https://jqlang.github.io/jq/download/) a popular json utility to pretty-print the output:

Ahora puedes usar curl en la URL `service/info` del servicio para asegurarte de que esté funcionando. Estamos
utilizando [jq](https://jqlang.github.io/jq/download/), una popular utilidad para JSON, para formatear la salida:

```shell
curl http://notifications-service.default.127.0.0.1.sslip.io/service/info | jq 
```

Deberías ver la siguiente salida:

```json
{
  "name": "NOTIFICATIONS",
  "podIp": "10.244.0.16",
  "podName": "notifications-service-00001-deployment-7ff76b4c77-qkk69",
  "podNamespace": "default",
  "podNodeName": "dev-control-plane",
  "podServiceAccount": "default",
  "source": "https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
  "version": "1.0.0"
}
```

Verifica que haya un Pod ejecutando nuestro contenedor:

```shell
> kubectl get pods
NAME                                                      READY   STATUS    RESTARTS   AGE
kafka-0                                                   1/1     Running   0          7m54s
notifications-service-00001-deployment-798f8f79f5-jrbr8   2/2     Running   0          4s
```

Observa que hay dos contenedores en ejecución: uno es nuestro Servicio de Notificaciones y el otro es el proxy `queue`
de Knative Serving, que se utiliza para almacenar en búfer las solicitudes entrantes y obtener métricas de tráfico.

Después de 90 segundos (por defecto), si no estás enviando solicitudes al Servicio de Notificaciones, la instancia del
servicio se reducirá automáticamente. Cada vez que llegue una nueva solicitud entrante, Knative Serving aumentará
automáticamente la escala del servicio y almacenará la solicitud en búfer hasta que la instancia esté lista.

En resumen, con Knative Serving obtenemos dos beneficios de inmediato:

- Una forma simplificada de ejecutar nuestras cargas de trabajo sin crear múltiples recursos de Kubernetes. Este enfoque
  se asemeja a una oferta de Contenedores como Servicio, que como equipo de Plataforma podrías querer ofrecer a tus
  equipos.
- Autoescalado dinámico utilizando el Autoscaler de Knative, que puede utilizarse para reducir a cero tus aplicaciones
  cuando no estén en uso. Esto se asemeja a un enfoque de Funciones como Servicio, que como equipo de Plataforma podrías
  querer proporcionar a tus equipos.

## Ejecutar la aplicación Conference con Servicios Knative

En esta sección implementaremos diferentes estrategias de lanzamiento para nuestra aplicación Conference. Para ello,
desplegaremos todos los demás servicios de la aplicación, también usando Servicios Knative.

Antes de instalar los otros servicios, necesitamos configurar PostgreSQL y Redis, ya que hemos instalado Kafka
previamente. Antes de instalar PostgreSQL, necesitamos crear un ConfigMap que contenga la instrucción SQL y cree la
tabla Proposals, para que el Helm Chart pueda hacer referencia al ConfigMap y ejecutar la instrucción cuando se inicie
la instancia de la base de datos.

```shell
kubectl apply -f knative/c4p-sql-init.yaml
```

```shell
helm install postgresql oci://registry-1.docker.io/bitnamicharts/postgresql --version 12.5.7 --set "image.debug=true" --set "primary.initdb.user=postgres" --set "primary.initdb.password=postgres" --set "primary.initdb.scriptsConfigMap=c4p-init-sql" --set "global.postgresql.auth.postgresPassword=postgres" --set "primary.persistence.size=1Gi"
```

y Redis:

```shell
helm install redis oci://registry-1.docker.io/bitnamicharts/redis --version 17.11.3 --set "architecture=standalone" --set "master.persistence.size=1Gi"
```

Now we can install all the other services (frontend, c4p-service, and agenda-service) by running:

```shell
kubectl apply -f knative/
```

Verifica que todos los Servicios Knative estén en estado `READY`:

```shell
> kubectl get ksvc
NAME                    URL                                                       LATESTCREATED                 LATESTREADY                   READY   REASON
agenda-service          http://agenda-service.default.127.0.0.1.sslip.io          agenda-service-00001          agenda-service-00001          True    
c4p-service             http://c4p-service.default.127.0.0.1.sslip.io             c4p-service-00001             c4p-service-00001             True    
frontend                http://frontend.default.127.0.0.1.sslip.io                frontend-00001                frontend-00001                True    
notifications-service   http://notifications-service.default.127.0.0.1.sslip.io   notifications-service-00001   notifications-service-00001   True    

```

Accede al Frontend de la Aplicación Conference apuntando tu navegador a la siguiente URL.
URL [http://frontend.default.127.0.0.1.sslip.io](http://frontend.default.127.0.0.1.sslip.io)

En este punto, la aplicación debería funcionar como se espera, con una pequeña diferencia: servicios como Agenda
Service y el C4P Service se reducirán cuando no estén en uso. Si listas los pods después de 90 segundos de inactividad,
deberías ver lo siguiente:

```shell
> kubectl get pods 
NAME                                                     READY   STATUS    RESTARTS   AGE
frontend-00002-deployment-7fdfb7b8c5-cw67t               2/2     Running   0          60s
kafka-0                                                  1/1     Running   0          20m
notifications-service-00002-deployment-c5787bc49-flcc9   2/2     Running   0          60s
postgresql-0                                             1/1     Running   0          9m23s
redis-master-0                                           1/1     Running   0          8m50s
```

Debido a que los servicios Agenda y C4P están almacenando estado en almacenamiento persistente (Redis y PostgreSQL), los
datos no se pierden cuando la instancia del servicio se reduce. Sin embargo, los servicios de Notificaciones y Frontend
mantienen datos en memoria (notificaciones y eventos consumidos), por lo que hemos configurado nuestro servicio Knative
para mantener al menos una instancia activa en todo momento. Todo el estado en memoria que se mantiene de esta manera
impactará cómo la aplicación puede escalar, pero recuerda que esto es solo un esqueleto funcional.

Ahora que tenemos la aplicación en funcionamiento, echemos un vistazo a algunas estrategias de lanzamiento diferentes.

## Lanzamientos Canary

En esta sección, ejecutaremos un ejemplo simple que muestra cómo realizar lanzamientos canary utilizando Servicios
Knative. Comenzaremos simplemente analizando la división de tráfico basada en porcentajes.

Las funcionalidades de división de tráfico basada en porcentajes se proporcionan listas para usar en los Servicios
Knative. Actualizaremos el Servicio de Notificaciones que desplegamos antes en lugar de cambiar el Frontend, ya que
manejar múltiples solicitudes para obtener archivos CCS y JavaScript puede ser complicado al usar la división de tráfico
basada en porcentajes.

Para asegurarte de que el servicio siga en funcionamiento, puedes ejecutar el siguiente comando:

```shell
curl http://notifications-service.default.127.0.0.1.sslip.io/service/info | jq
```

Deberías ver la siguiente salida:

```json
{
  "name": "NOTIFICATIONS",
  "podIp": "10.244.0.16",
  "podName": "notifications-service-00001-deployment-7ff76b4c77-qkk69",
  "podNamespace": "default",
  "podNodeName": "dev-control-plane",
  "podServiceAccount": "default",
  "source": "https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
  "version": "1.0.0"
}

```

Puedes editar el Servicio Knative (`ksvc`) del Servicio de Notificaciones y crear una nueva revisión cambiando la imagen
del contenedor que está utilizando el servicio o modificando cualquier otro parámetro de configuración, como las
variables de entorno:

```shell
kubectl edit ksvc notifications-service
```

Y luego cambia de:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: notifications-service
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "1"
    spec:
      containers:
        - image: salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
      ...
```

A `v1.1.0`:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: notifications-service
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "1"
    spec:
      containers:
        - image: salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0
      ...
```

Antes de guardar este cambio, que creará una nueva revisión que podemos usar para dividir el tráfico, necesitamos
agregar los siguientes valores en la sección de tráfico:

```yaml
 traffic:
   - latestRevision: false
     percent: 50
     revisionName: notifications-service-00001
   - latestRevision: true
     percent: 50
```

Ahora, si comienzas a acceder al endpoint `service/info` nuevamente, verás que la mitad del tráfico se está dirigiendo a
la versión `v1.0.0` de nuestro servicio y la otra mitad a la versión `v1.1.0`.

```shell
> curl http://notifications-service.default.127.0.0.1.sslip.io/service/info | jq
{
    "name":"NOTIFICATIONS-IMPROVED",
    "version":"1.1.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/v1.1.0/conference-application/notifications-service",
    "podName":"notifications-service-00003-deployment-59fb5bff6c-2gfqt",
    "podNamespace":"default",
    "podNodeName":"dev-control-plane",
    "podIp":"10.244.0.34",
    "podServiceAccount":"default"
}

> curl http://notifications-service.default.127.0.0.1.sslip.io/service/info | jq
{
    "name":"NOTIFICATIONS",
    "version":"1.0.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
    "podName":"notifications-service-00001-deployment-7ff76b4c77-h6ts4",
    "podNamespace":"default",
    "podNodeName":"dev-control-plane",
    "podIp":"10.244.0.35",
    "podServiceAccount":"default"
}
```

Este mecanismo es muy útil cuando necesitas probar una nueva versión, pero no estás dispuesto a dirigir todo el tráfico
en vivo de inmediato a la nueva versión en caso de que surjan problemas.

Puedes modificar las reglas de tráfico para tener una división de porcentaje diferente si te sientes confiado en que la
versión más reciente es lo suficientemente estable como para recibir más tráfico.

```yaml
 traffic:
   - latestRevision: false
     percent: 10
     revisionName: notifications-service-00001
   - latestRevision: true
     percent: 90
```

En el momento en que una revisión (versión) no tenga ninguna regla de tráfico apuntando a ella, la instancia se
reducirá, ya que no se dirigirá tráfico hacia ella.

# Pruebas A/B y Despliegues Blue/Green

Con las pruebas A/B queremos ejecutar dos o más versiones de la misma aplicación/servicio al mismo tiempo para permitir
que diferentes grupos de usuarios prueben cambios y así decidir cuál versión funciona mejor para ellos.

Con Knative Serving tenemos dos opciones: enrutamientos `Header-based` y `Tag-based`. Ambos utilizan
los mismos mecanismos y configuraciones detrás de escena, pero veamos cómo se pueden usar estos mecanismos.

Con el enrutamiento `Tag/Header-based` tenemos más control sobre hacia dónde se enviarán las solicitudes, ya que
podemos usar un encabezado HTTP o una URL específica para instruir a los mecanismos de red de Knative a dirigir el
tráfico a versiones específicas del servicio.

Esto significa que, para este ejemplo, podemos cambiar el Frontend de nuestra aplicación, ya que todas las solicitudes
que incluyan un encabezado o una URL específica se dirigirán a la misma versión del servicio.

Asegúrate de acceder al Frontend de la aplicación apuntando tu navegador
a: [http://frontend.default.127.0.0.1.sslip.io](http://frontend.default.127.0.0.1.sslip.io)

![frontend v1.0.0](../imgs/frontend-v1.0.0.png)

Ahora vamos a modificar el Servicio Knative del Frontend para desplegar una nueva versión con la función de depuración
habilitada:

```shell
kubectl edit ksvc frontend
```

Actualiza el campo de imagen para que apunte a `v1.1.0` y agrega la variable de entorno FEATURE_DEBUG_ENABLED (recuerda
que estamos usando la primera versión de la aplicación que no utiliza OpenFeature).

```yaml
spec:
  containerConcurrency: 0
  containers:
    - env:
        - name: FEATURE_DEBUG_ENABLED
          value: "true"
    ...
    image: salaboy/frontend-go-1739aa83b5e69d4ccb8a5615830ae66c:v1.1.0
```

Antes de guardar el Servicio Knative, cambiemos las reglas de tráfico para que coincidan con las siguientes:

```yaml
traffic:
  - latestRevision: false
    percent: 100
    revisionName: frontend-00001
    tag: current
  - latestRevision: true
    percent: 0
    tag: version110
```

Ten en cuenta que no se dirigirá tráfico (porcentaje: 0) a `v1.1.0` a menos que se especifique la etiqueta en la URL del
servicio. Los usuarios ahora pueden acceder
a: [http://version110-frontend.default.127.0.0.1.sslip.io](http://version110-frontend.default.127.0.0.1.sslip.io) para
acceder a `v1.1.0`

![v1.1.0](../imgs/frontend-v1.1.0.png)

Ten en cuenta que `v1.1.0` tiene un tema de color diferente; cuando los veas uno al lado del otro, notarás la
diferencia.
También revisa las otras secciones de la aplicación.

Si por alguna razón no deseas o no puedes cambiar la URL del servicio, puedes utilizar encabezados HTTP para acceder a
`v1.1.0`. Usando un complemento de navegador
como [Chrome ModHeader](https://chrome.google.com/webstore/detail/modheader-modify-http-hea/idgpnmonknjnojddfkpgkljpfnnfcklj),
puedes modificar todas las solicitudes que el navegador está enviando al
agregar parámetros o encabezados.

Aquí estamos configurando el encabezado `Knative-Serving-Tag` con el valor `version110`, que es el nombre de la etiqueta
que
configuramos en las reglas de tráfico para nuestro Servicio Knative de frontend.

Ahora podemos acceder a la URL normal del Servicio Knative (sin cambios) para acceder a:
`v1.1.0`: [http://frontend.default.127.0.0.1.sslip.io](http://frontend.default.127.0.0.1.sslip.io)

![v1.1.0 with header](../imgs/frontend-v1.1.0-with-header.png)

El enrutamiento basado en etiquetas y encabezados nos permite implementar despliegues Blue/Green de la misma manera, ya
que el servicio `green` (el que queremos probar hasta que esté listo para su uso en producción) puede estar oculto
detrás
de una etiqueta con un 0% de tráfico asignado.

```yaml
traffic:
  - revisionName: <blue-revision-name>
    percent: 100 # All traffic is still being routed to this revision
  - revisionName: <gree-revision-name>
    percent: 0 # 0% of traffic routed to this version
    tag: green # A named route
```

Whenever we are ready to switch to our `green` service we can change the traffic rules:

```yaml
traffic:
  - revisionName: <blue-revision-name>
    percent: 0
    tag: blue
  - revisionName: <gree-revision-name>
    percent: 100

```

Para resumir, al utilizar las capacidades de división de tráfico y enrutamiento `header/tag-based` de los
Servicios Knative, hemos implementado lanzamientos Canary, patrones de pruebas A/B y despliegues Blue/Green. Consulta el
sitio web [Knative Website](https://knative.dev)  de Knative para obtener más información sobre el proyecto.

To recap, by using Knative Services traffic splitting and header/tag-based routing capabilities we have implemented
Canary Releases, A/B testing patterns, and Blue/Green deployments. Check the [Knative Website](https://knative.dev) for
more information about the project.

Limpieza
Si deseas eliminar el clúster KinD creado para este tutorial, puedes ejecutar:

```shell
kind delete clusters dev
```

