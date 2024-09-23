# Estrategias de Lanzamiento con Argo Rollouts

—
_🌍 Disponible
en_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md) | [Español](README-es.md)
> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

En este tutorial, exploraremos los mecanismos integrados de Argo Rollouts para implementar estrategias de lanzamiento.
También examinaremos el panel de control de Argo Rollouts, que permite a los equipos promover nuevas versiones sin
necesidad de usar la terminal (`kubectl`).

## Instalación

Necesitas un clúster de Kubernetes para instalar Argo Rollouts [Argo Rollouts](https://argoproj.github.io/rollouts/).
Puedes crear uno usando Kubernetes KinD, como hicimos en
el Capítulo
2 [Capítulo 2](https://github.com/salaboy/platforms-on-k8s/blob/main/chapter-2/README-es.md#creating-a-local-cluster-with-kubernetes-kind)

Una vez que tengas el clúster, podemos instalar Argo Rollouts ejecutando:

```shell
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

o siguiendo la documentación oficial de Argo
Rollouts [official documentation that you can find here](https://argoproj.github.io/argo-rollouts/installation/#controller-installation).

También necesitas instalar el [Argo Rollouts
`kubectl` plugin](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)

Una vez que tengas el complemento, puedes iniciar una versión local del panel de control de Argo Rollouts ejecutando el
siguiente comando en una nueva terminal:

```shell
kubectl argo rollouts dashboard
```

Luego, puedes acceder al panel dirigiendo tu navegador
a [http://localhost:3100/rollouts](http://localhost:3100/rollouts)

![argo rollouts dashboard empty](../imgs/argo-rollouts-dashboard-empty.png)

## Lanzamientos Canary

Vamos a crear un recurso de Argo Rollout para implementar un lanzamiento Canary en el Servicio de Notificaciones de la
Aplicación de Conferencias. Puedes encontrarla en [full definition here](canary-release/rollout.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: notifications-service-canary
spec:
  replicas: 3
  strategy:
    canary:
      steps:
        - setWeight: 25
        - pause: { }
        - setWeight: 75
        - pause: { duration: 10 }
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: notifications-service
  template:
    metadata:
      labels:
        app: notifications-service
    spec:
      containers:
        - name: notifications-service
          image: salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
          env:
            - name: KAFKA_URL
              value: kafka.default.svc.cluster.local
            ...
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          resources:
            limits:
              cpu: "1"
              memory: 256Mi
            requests:
              cpu: "0.1"
              memory: 256Mi

```

The `Rollout` resource replaces our Kubernetes `Deployment` resource. This means we still need to create a Kubernetes
Service and an Ingress Resource to route traffic to our Notification Service instance. Notice that we are defining three
replicas for the Notification Service.

The previous `Rollout` defines a canary release with two steps:

El recurso `Rollout` reemplaza nuestro recurso de `Deployment` de Kubernetes. Esto significa que aún necesitamos crear
un
Servicio de Kubernetes y un recurso de Ingress para dirigir el tráfico a nuestra instancia del Servicio de
Notificaciones. Ten en cuenta que estamos definiendo tres réplicas para el Servicio de Notificaciones.

El Rollout anterior define un lanzamiento canary con dos pasos:

```yaml
strategy:
  canary:
    steps:
      - setWeight: 25
      - pause: { }
      - setWeight: 75
      - pause: { duration: 10 }
```

Primero, establecerá la división de tráfico al 25 por ciento y esperará que el equipo pruebe la nueva versión (el paso
de pausa). Luego, después de que señalemos manualmente que queremos continuar, el rollout pasará al 75 por ciento hacia
la nueva versión, para finalmente pausar durante 10 segundos y luego moverse al 100 por ciento.

Antes de aplicar los recursos de Rollout, Servicio e Ingress ubicados en el directorio `canary-release/`, instalemos
Kafka
para que el Servicio de Notificaciones pueda conectarse.

```shell
helm install kafka oci://registry-1.docker.io/bitnamicharts/kafka --version 22.1.5 --set "provisioning.topics[0].name=events-topic" --set "provisioning.topics[0].partitions=1" --set "persistence.size=1Gi" 

```

Ahora que Kafka está en funcionamiento, apliquemos todos los recursos en el directorio `canary-releases/`:

```shell
kubectl apply -f canary-release/
```

Usando el plugin de Argo Rollouts, puedes observar el despliegue desde la terminal:

```shell
kubectl argo rollouts get rollout notifications-service-canary --watch
```

Deberías ver algo como esto:

```shell
Name:            notifications-service-canary
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          4/4
  SetWeight:     100
  ActualWeight:  100
Images:          salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0 (stable)
Replicas:
  Desired:       3
  Current:       3
  Updated:       3
  Ready:         3
  Available:     3

NAME                                                      KIND        STATUS     AGE  INFO
⟳ notifications-service-canary                            Rollout     ✔ Healthy  80s  
└──# revision:1                                                                       
   └──⧉ notifications-service-canary-7f6b88b5fb           ReplicaSet  ✔ Healthy  80s  stable
      ├──□ notifications-service-canary-7f6b88b5fb-d86s2  Pod         ✔ Running  80s  ready:1/1
      ├──□ notifications-service-canary-7f6b88b5fb-dss5c  Pod         ✔ Running  80s  ready:1/1
      └──□ notifications-service-canary-7f6b88b5fb-tw8fj  Pod         ✔ Running  80s  ready:1/1
```

Como puede ver, debido a que acabamos de crear los lanzamientos, se crean tres réplicas y todo el tráfico se enruta a
esta `revision:1` inicial, y el estado se establece en `Healthy`.

Vamos a actualizar la versión del Servicio de Notificaciones a `v1.1.0` ejecutando:

```shell
kubectl argo rollouts set image notifications-service-canary \
  notifications-service=salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0
```

Ahora puedes ver que se ha creado la segunda revisión (`revision:2`):

```shell
Name:            notifications-service-canary
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/4
  SetWeight:     25
  ActualWeight:  25
Images:          salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0 (stable)
                 salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0 (canary)
Replicas:
  Desired:       3
  Current:       4
  Updated:       1
  Ready:         4
  Available:     4

NAME                                                      KIND        STATUS     AGE    INFO
⟳ notifications-service-canary                            Rollout     ॥ Paused   4m29s  
├──# revision:2                                                                         
│  └──⧉ notifications-service-canary-68fd6b4ff9           ReplicaSet  ✔ Healthy  14s    canary
│     └──□ notifications-service-canary-68fd6b4ff9-jrjxh  Pod         ✔ Running  14s    ready:1/1
└──# revision:1                                                                         
   └──⧉ notifications-service-canary-7f6b88b5fb           ReplicaSet  ✔ Healthy  4m29s  stable
      ├──□ notifications-service-canary-7f6b88b5fb-d86s2  Pod         ✔ Running  4m29s  ready:1/1
      ├──□ notifications-service-canary-7f6b88b5fb-dss5c  Pod         ✔ Running  4m29s  ready:1/1
      └──□ notifications-service-canary-7f6b88b5fb-tw8fj  Pod         ✔ Running  4m29s  ready:1/1
```

Ahora el Rollout se detiene en el paso 1, donde solo el 25 por ciento del tráfico se dirige a `revision:2` y el estado
está establecido en Pausa.

Siéntete libre de acceder al endpoint `service/info` para ver qué versión está respondiendo a tus solicitudes:

```shell
curl localhost/service/info
```

Aproximadamente, una de cada cuatro solicitudes debería ser respondida por la versión `v1.1.0`:

```shell
> curl localhost/service/info | jq

{
    "name":"NOTIFICATIONS",
    "version":"1.0.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
    "podName":"notifications-service-canary-7f6b88b5fb-tw8fj",
    "podNamespace":"default",
    "podNodeName":"dev-worker2",
    "podIp":"10.244.3.3",
    "podServiceAccount":"default"
}

> curl localhost/service/info | jq

{
    "name":"NOTIFICATIONS",
    "version":"1.0.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
    "podName":"notifications-service-canary-7f6b88b5fb-tw8fj",
    "podNamespace":"default",
    "podNodeName":"dev-worker2",
    "podIp":"10.244.3.3",
    "podServiceAccount":"default"
}

> curl localhost/service/info | jq

{
    "name":"NOTIFICATIONS-IMPROVED",
    "version":"1.1.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/v1.1.0/conference-application/notifications-service",
    "podName":"notifications-service-canary-68fd6b4ff9-jrjxh",
    "podNamespace":"default",
    "podNodeName":"dev-worker",
    "podIp":"10.244.2.4",
    "podServiceAccount":"default"
}

> curl localhost/service/info | jq

{
    "name":"NOTIFICATIONS",
    "version":"1.0.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
    "podName":"notifications-service-canary-7f6b88b5fb-tw8fj",
    "podNamespace":"default",
    "podNodeName":"dev-worker2",
    "podIp":"10.244.3.3",
    "podServiceAccount":"default"
}
```

También revisa el panel de Argo Rollouts ahora; debería mostrar el lanzamiento Canary:

![canary release in dashboard](../imgs/argo-rollouts-dashboard-canary-1.png)

Puedes avanzar con el canary utilizando el comando de promoción o el botón de promoción en el panel. El comando se ve
así:

```shell
kubectl argo rollouts promote notifications-service-canary
```

Esto debería mover a canary al 75% del tráfico y después de 10 segundos más debería estar al 100%, ya que el último
paso de la fase dura solo 10 segundos. Deberías ver en la terminal:

```shell
Name:            notifications-service-canary
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          4/4
  SetWeight:     100
  ActualWeight:  100
Images:          salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0 (stable)
Replicas:
  Desired:       3
  Current:       3
  Updated:       3
  Ready:         3
  Available:     3

NAME                                                      KIND        STATUS        AGE  INFO
⟳ notifications-service-canary                            Rollout     ✔ Healthy     16m  
├──# revision:2                                                                          
│  └──⧉ notifications-service-canary-68fd6b4ff9           ReplicaSet  ✔ Healthy     11m  stable
│     ├──□ notifications-service-canary-68fd6b4ff9-jrjxh  Pod         ✔ Running     11m  ready:1/1
│     ├──□ notifications-service-canary-68fd6b4ff9-q4zgj  Pod         ✔ Running     51s  ready:1/1
│     └──□ notifications-service-canary-68fd6b4ff9-fctjv  Pod         ✔ Running     46s  ready:1/1
└──# revision:1                                                                          
   └──⧉ notifications-service-canary-7f6b88b5fb           ReplicaSet  • ScaledDown  16m  

```

Y en el Dashboard:
![canary promoted](../imgs/argo-rollouts-dashboard-canary-2.png)

Ahora todas las solicitudes deberían ser respondidas por `v1.1.0`:

```shell

> curl localhost/service/info

{
    "name":"NOTIFICATIONS-IMPROVED",
    "version":"1.1.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/v1.1.0/conference-application/notifications-service",
    "podName":"notifications-service-canary-68fd6b4ff9-jrjxh",
    "podNamespace":"default",
    "podNodeName":"dev-worker",
    "podIp":"10.244.2.4",
    "podServiceAccount":"default"
}

> curl localhost/service/info

{
    "name":"NOTIFICATIONS-IMPROVED",
    "version":"1.1.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/v1.1.0/conference-application/notifications-service",
    "podName":"notifications-service-canary-68fd6b4ff9-jrjxh",
    "podNamespace":"default",
    "podNodeName":"dev-worker",
    "podIp":"10.244.2.4",
    "podServiceAccount":"default"
}

> curl localhost/service/info

{
    "name":"NOTIFICATIONS-IMPROVED",
    "version":"1.1.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/v1.1.0/conference-application/notifications-service",
    "podName":"notifications-service-canary-68fd6b4ff9-jrjxh",
    "podNamespace":"default",
    "podNodeName":"dev-worker",
    "podIp":"10.244.2.4",
    "podServiceAccount":"default"
}

```

Antes de avanzar con las implementaciones Blue/Green, limpiemos la implementación canary ejecutando lo siguiente:

```shell
kubectl delete -f canary-release/
```

## Despliegues Blue/Green

Con las implementaciones Blue/Green queremos tener dos versiones de nuestro servicio ejecutándose al mismo tiempo. La
versión Blue (activa) a la que accederán todos los usuarios y la versión Green (vista previa) que los equipos internos
pueden usar para probar nuevas funciones y cambios.

Las implementaciones de Argo brindan una estrategia Blue/Green lista para usar:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: notifications-service-bluegreen
spec:
  replicas: 2
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: notifications-service
  template:
    metadata:
      labels:
        app: notifications-service
    spec:
      containers:
        - name: notifications-service
          image: salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
          env:
            - name: KAFKA_URL
              value: kafka.default.svc.cluster.local
            ..
  strategy:
    blueGreen:
      activeService: notifications-service-blue
      previewService: notifications-service-green
      autoPromotionEnabled: false
```

Una vez más, estamos utilizando nuestro Servicio de Notificaciones para probar el mecanismo de despliegue gradual (
Rollout). Aquí hemos definido un despliegue Blue/Green para el Servicio de Notificaciones que apunta a dos Servicios de
Kubernetes existentes: `notifications-service-blue` y `notifications-service-green`. Tenga en cuenta que la bandera
`autoPromotionEnabled` está configurada como `false`, esto detiene la promoción automática cuando la nueva versión está
lista.

Verifique que ya tenga Kafka ejecutándose desde la sección anterior (Lanzamientos Canary) y aplique todos los recursos
ubicados dentro del directorio `blue-green/`:

```shell
kubectl apply -f blue-green/
```

Esto crea el recurso `Rollout`, dos Servicios de Kubernetes y dos recursos de Ingress, uno para el Servicio Blue que
redirige el tráfico desde `/` y otro para el Servicio Green que redirige el tráfico desde `/preview/`.

Puedes monitorear el Rollout en la terminal ejecutando:

```shell
kubectl argo rollouts get rollout notifications-service-bluegreen --watch
```

Debes ver algo como esto:

```
Name:            notifications-service-bluegreen
Namespace:       default
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0 (stable, active)
Replicas:
  Desired:       2
  Current:       2
  Updated:       2
  Ready:         2
  Available:     2

NAME                                                         KIND        STATUS     AGE    INFO
⟳ notifications-service-bluegreen                            Rollout     ✔ Healthy  3m16s  
└──# revision:1                                                                            
   └──⧉ notifications-service-bluegreen-56bb777689           ReplicaSet  ✔ Healthy  2m56s  stable,active
      ├──□ notifications-service-bluegreen-56bb777689-j5ntk  Pod         ✔ Running  2m56s  ready:1/1
      └──□ notifications-service-bluegreen-56bb777689-qzg9l  Pod         ✔ Running  2m56s  ready:1/1

```

Tenemos dos réplicas de nuestro Servicio de Notificaciones funcionando. Si hacemos curl `localhost/service/info`
deberíamos obtener la información del Servicio de Notificaciones `v1.0.0`:

```shell
> curl localhost/service/info | jq

{
    "name":"NOTIFICATIONS",
    "version":"1.0.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
    "podName":"notifications-service-canary-7f6b88b5fb-tw8fj",
    "podNamespace":"default",
    "podNodeName":"dev-worker2",
    "podIp":"10.244.3.3",
    "podServiceAccount":"default"
}
```

Y el panel de control de Argo Rollouts debería mostrarnos nuestro despliegue Blue/Green:

![blue green 1](../imgs/argo-rollouts-dashboard-bluegree-1.png)

Al igual que hicimos con el lanzamiento Canary, podemos actualizar nuestra configuración de Rollout, en este caso
configurando la imagen para la versión `v1.1.0`.

```shell
kubectl argo rollouts set image notifications-service-bluegreen \
  notifications-service=salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0
```

Ahora deberías ver en la terminal ambas versiones del Servicio de Notificaciones ejecutándose en paralelo:

```shell
Name:            notifications-service-bluegreen
Namespace:       default
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0 (stable, active)
                 salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0 (preview)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                                         KIND        STATUS     AGE    INFO
⟳ notifications-service-bluegreen                            Rollout     ॥ Paused   8m54s  
├──# revision:2                                                                            
│  └──⧉ notifications-service-bluegreen-645d484596           ReplicaSet  ✔ Healthy  16s    preview
│     ├──□ notifications-service-bluegreen-645d484596-ffhsm  Pod         ✔ Running  16s    ready:1/1
│     └──□ notifications-service-bluegreen-645d484596-g2zr4  Pod         ✔ Running  16s    ready:1/1
└──# revision:1                                                                            
   └──⧉ notifications-service-bluegreen-56bb777689           ReplicaSet  ✔ Healthy  8m34s  stable,active
      ├──□ notifications-service-bluegreen-56bb777689-j5ntk  Pod         ✔ Running  8m34s  ready:1/1
      └──□ notifications-service-bluegreen-56bb777689-qzg9l  Pod         ✔ Running  8m34s  ready:1/1
```

Tanto `v1.0.0` como `v1.1.0` están ejecutándose y en buen estado, pero el estado del despliegue Blue/Green está en
pausa, ya que mantendrá ambas versiones ejecutándose hasta que el equipo responsable de validar la versión `preview` /
`green` esté listo para el prime time.

Revisa el panel de control de Argo Rollouts, debería mostrar ambas versiones ejecutándose también:

![blue green 2](../imgs/argo-rollouts-dashboard-bluegree-2.png)

En este punto, puedes enviar solicitudes a ambos servicios utilizando las rutas de Ingress que definimos. Puedes hacer
curl `localhost/service/info` para acceder al servicio Blue (servicio estable) y curl `localhost/preview/service/info`
para acceder al servicio Green (servicio de previsualización).

```shell
> curl localhost/service/info

{
    "name":"NOTIFICATIONS",
    "version":"1.0.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/main/conference-application/notifications-service",
    "podName":"notifications-service-canary-7f6b88b5fb-tw8fj",
    "podNamespace":"default",
    "podNodeName":"dev-worker2",
    "podIp":"10.244.3.3",
    "podServiceAccount":"default"
}
```

Y ahora verifiquemos el Servicio Verde:

```shell
> curl localhost/green/service/info

{
    "name":"NOTIFICATIONS-IMPROVED",
    "version":"1.1.0",
    "source":"https://github.com/salaboy/platforms-on-k8s/tree/v1.1.0/conference-application/notifications-service",
    "podName":"notifications-service-canary-68fd6b4ff9-jrjxh",
    "podNamespace":"default",
    "podNodeName":"dev-worker",
    "podIp":"10.244.2.4",
    "podServiceAccount":"default"
}
```

Si estamos satisfechos con los resultados, podemos promover nuestro Servicio Green para que sea nuestro nuevo servicio
estable, lo hacemos presionando el botón Promover en el panel de control de Argo Rollouts o ejecutando el siguiente
comando:

```shell
kubectl argo rollouts promote notifications-service-bluegreen
```

Debes ver en la terminal:

```shell
Name:            notifications-service-bluegreen
Namespace:       default
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.0.0
                 salaboy/notifications-service-0e27884e01429ab7e350cb5dff61b525:v1.1.0 (stable, active)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                                         KIND        STATUS     AGE    INFO
⟳ notifications-service-bluegreen                            Rollout     ✔ Healthy  2m44s  
├──# revision:2                                                                            
│  └──⧉ notifications-service-bluegreen-645d484596           ReplicaSet  ✔ Healthy  2m27s  stable,active
│     ├──□ notifications-service-bluegreen-645d484596-fnbg7  Pod         ✔ Running  2m27s  ready:1/1
│     └──□ notifications-service-bluegreen-645d484596-ntcbf  Pod         ✔ Running  2m27s  ready:1/1
└──# revision:1                                                                            
   └──⧉ notifications-service-bluegreen-56bb777689           ReplicaSet  ✔ Healthy  2m44s  delay:9s
      ├──□ notifications-service-bluegreen-56bb777689-k6qxk  Pod         ✔ Running  2m44s  ready:1/1
      └──□ notifications-service-bluegreen-56bb777689-vzsw7  Pod         ✔ Running  2m44s  ready:1/1
```

Ahora el servicio estable es `revision:2`. Verás que Argo Rollouts mantendrá `revision:1` activo por un tiempo, solo en
caso
de que queramos revertir, pero después de unos segundos, se reducirá.

Verifica en el panel de control que nuestro despliegue está en `revision:2` también:

![rollout promoted](../imgs/argo-rollouts-dashboard-bluegree-3.png)

¡Si llegaste hasta aquí, implementaste Lanzamientos Canary y Despliegues Blue/Green usando Argo Rollouts!

## Limpieza

Si deseas eliminar el clúster KinD creado para este tutorial, puedes ejecutar:

```shell
kind delete clusters dev
```

