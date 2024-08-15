# Capítulo 7 :: Shared Applications Concerns

---
_🌍 Available in_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md) | [Español](README-es.md)

> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

En este tutorial paso a paso, veremos como usar [Dapr](https://dapr.io) para proporcionar APIs a nivel de Aplicación para resolver los problemas que la mayoría de las Aplicaciones Distribuidas tienen que resolver.

Luego miraremos [OpenFeature](https://openfeature.dev), un proyecto que está hecho para estandarizar las banderas de características y así los equipos de Desarrollo puedan seguir lanzando nuevas funcionalidades y los stakeholders puedan decidir cuándo habilitar/desactivar estas características para sus clientes.

Debido a que ambos proyectos están enfocados en proveer a los desarrolladores con nuevas API y herramientas que utilizar dentro del código de sus servicios, desplegaremos una nueva versión de la aplicación (`v2.0.0).
encontrarás todos los cambios requeridos para esta nueva versión en la rama `v2.0.0` de este repositorio.
Puedes comparar las diferencias entre las ramas aquí](https://github.com/salaboy/platforms-on-k8s/compare/v2.0.0).


# Instalación

Necesitas un Clúster de Kubernetes para instalar [Dapr](https://dapr.io) y `flagd`, un proveedor de [OpenFeature](https://openfeature.dev/).
Puedes crear uno utilizando Kubernetes KinD como vimos en el [Capítulo 2](../chapter-2/README-es.md#creando-un-clúster-local-con-kubernetes-kind)

Luego puedes instalar Dapr en el clúster ejecutando:
```shell
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update
helm upgrade --install dapr dapr/dapr \
--version=1.11.0 \
--namespace dapr-system \
--create-namespace \
--wait

```

Una vez que Dapr está instalado, podemos instalar nuestra versión `v2.0.0` de la aplicación habilitada con Dapr y FeatureFlag.

# Ejecutando v2.0.0
Ahora puedes instalar la versión `v2.0.0` de la aplicación ejecutando:

```shell
helm install conference oci://docker.io/salaboy/conference-app --version v2.0.0
```

Esta versión del chart de Helm instala la misma infraestructura de la aplicación que la versión
`v1.0.0` (PostgreSQL, Redis, y Kafka).
Los servicios que interactúan con Redis y Kafka ahora usan las API de Dapr.
Esta versión de la aplicación también añade banderas de características OpenFeature utilizando `flagd`.

# API a nivel de aplicación con Dapr

En la versión `v2.0.0`, si haces un listado de pods de la Aplicación, verás que cada servicio (agenda, c4p, frontend, y notificaciones) tiene un Dapr Sidecar (`daprd`) corriendo junto con el contenedor de servicio (READY 2/2):
```shell
> kubectl get pods
NAME                                                           READY   STATUS    RESTARTS      AGE
conference-agenda-service-deployment-5dd4bf67b-qkctd           2/2     Running   7 (7s ago)    74s
conference-c4p-service-deployment-57b5985757-tdqg4             2/2     Running   6 (19s ago)   74s
conference-frontend-deployment-69d9b479b7-th44h                2/2     Running   2 (68s ago)   74s
conference-kafka-0                                             1/1     Running   0             74s
conference-notifications-service-deployment-7b6cbf965d-2pdkh   2/2     Running   6 (42s ago)   74s
conference-postgresql-0                                        1/1     Running   0             74s
conference-redis-master-0                                      1/1     Running   0             74s
flagd-6bbdc5d999-c42wk                                         1/1     Running   0             74s
```

Notemos que el contenedor `flagd` también se está ejecutando.
Cubriremos esto en la siguiente sección.

Desde la perspectiva de Dapr, la aplicación se ve así: 

![conference-app-with-dapr](imgs/conference-app-with-dapr.png)

El sidecar de Dapr exponer las API de los componentes Dapr para que los servicios puedan interactuar con las API de Statestore (Redis) y PubSub (Kafka).

Puedes listar los componentes Dapr ejecutando:

```shell
> kubectl get components
NAME                                   AGE
conference-agenda-service-statestore   30m
conference-conference-pubsub           30m
```

Puedes describir cada componente para ver sus configuraciones:
```shell
> kubectl describe component conference-agenda-service-statestore
Name:         conference-agenda-service-statestore
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: conference
              meta.helm.sh/release-namespace: default
API Version:  dapr.io/v1alpha1
Auth:
  Secret Store:  kubernetes
Kind:            Component
Metadata:
  Creation Timestamp:  2023-07-28T08:26:55Z
  Generation:          1
  Resource Version:    4076
  UID:                 b4674825-d298-4ee3-8244-a13cdef8d530
Spec:
  Metadata:
    Name:   keyPrefix
    Value:  name
    Name:   redisHost
    Value:  conference-redis-master.default.svc.cluster.local:6379
    Name:   redisPassword
    Secret Key Ref:
      Key:   redis-password
      Name:  conference-redis
  Type:      state.redis
  Version:   v1
Events:      <none>

```

Puedes observar que el componente Statestore se conecta con la instancia de Redis expuesta por este nombre de servicio`conference-redis-master.default.svc.cluster.local` usando el secreto `conference-redis` para obtener la contraseña para conectarse.

De forma similar, el componente Dapr PubSub conecta con Kafka:
```shell
kubectl describe component conference-conference-pubsub 
Name:         conference-conference-pubsub
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: conference
              meta.helm.sh/release-namespace: default
API Version:  dapr.io/v1alpha1
Kind:         Component
Metadata:
  Creation Timestamp:  2023-07-28T08:26:55Z
  Generation:          1
  Resource Version:    4086
  UID:                 e145bc49-18ff-4390-ad15-dcd9a4275479
Spec:
  Metadata:
    Name:   brokers
    Value:  conference-kafka.default.svc.cluster.local:9092
    Name:   authType
    Value:  none
  Type:     pubsub.kafka
  Version:  v1
Events:     <none>
```

La pieza final del rompecabezas que permite al servicio Frontend recibir eventos que se envían al componente Pubsub es la siguiente suscripción de Dapr: 

```shell
> kubectl get subscription
NAME                               AGE
conference-frontend-subscritpion   39m
```

También puedes describir este recurso para mirar sus configuraciones:
```shell
> kubectl describe subscription conference-frontend-subscritpion
Name:         conference-frontend-subscritpion
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: conference
              meta.helm.sh/release-namespace: default
API Version:  dapr.io/v2alpha1
Kind:         Subscription
Metadata:
  Creation Timestamp:  2023-07-28T08:26:55Z
  Generation:          1
  Resource Version:    4102
  UID:                 9f748cb0-125a-4848-bd39-f84e37e41282
Scopes:
  frontend
Spec:
  Bulk Subscribe:
    Enabled:   false
  Pubsubname:  conference-conference-pubsub
  Routes:
    Default:  /api/new-events/
  Topic:      events-topic
Events:       <none>
```

Como puedes ver,
esta suscripción reenvía los eventos a la ruta `/api/new-events/` para que las aplicaciones Daps listadas
en la sección `Scopes`, solo la aplicación `frontend`.
Solo la aplicación Frontend necesita exponer el endpoint para recibir eventos,
en este caso el sidecar de Dapr (`daprd`)
espera los mensajes entrantes en el componente PubSub llamado `conference-conference-pubsub` y reenvía todos los mensajes al endpoint de la Aplicación.

Esta versión de la aplicación eliminar dependencias de la aplicación tales como el cliente de Kafka de todos los servicios y el cliente de Redis del servicio de Agenda.


![services without deps](imgs/conference-app-dapr-no-deps.png)

Ademas de eliminar las dependencias y hacer los contenedores más pequeños,
al consumir la API de los componentes Dapr, permitimos que el equipo de plataforma defina cómo se configurar esos componentes y sobre qué infraestructura.
Configurar la misma aplicación para usar servicios administrados de Google Cloud Platform como [Google PubSub](https://cloud.google.com/pubsub) o la [MemoryStore databases](https://cloud.google.com/memorystore) no requiere cambios en el código de la aplicación o agregar nuevas dependencias, solo se necesitan nuevas configuraciones de componentes de Dapr.

![in gcp](imgs/conference-app-dapr-and-gcp.png)

Finalmente, ya que todo se trata de permitir que los desarrolladores utilicen las API a nivel de aplicación, miremos cómo se ve desde la perspectiva del servicio.
Como los servicios están escritos en Go, he decidido añadir la SDK de Dapr para Go (que es opcional).

Cuando el servicio de Agenda quiere almacenar o leer datos del componente Statestore, puede utilizar el cliente Dapr para realizar estar operaciones, por ejemplo 

Finally, because this is all about enabling developers with Application Level APIs, let's look at how this looks from the application's service perspective. Because the services are written in Go, I've decided to add the Dapr Go SDK (which is optional). 

When the Agenda Service wants to store or read data from the Dapr Statestore component, it can use the Dapr Client to perform these operations, for example [reading values from the Statestore looks like this](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L136C2-L136C116): 

```golang
agendaItemsStateItem, err := s.APIClient.GetState(ctx, STATESTORE_NAME, fmt.Sprintf("%s-%s", TENANT_ID, KEY), nil)
```

The `APIClient` reference is just a [Dapr Client instance, which was initialized here](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L397)

All the application needs to know is the Statestore name (`STATESTORE_NAME`) and the key (`KEY`) to locate the data that wants to be retrieved.

When the application wants to [store state into the Statestore it looks like this](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L197C2-L199C3):

```golang
if err := s.APIClient.SaveState(ctx, STATESTORE_NAME, fmt.Sprintf("%s-%s", TENANT_ID, KEY), jsonData, nil); err != nil {
		...
}
```  

Finally, if the application code wants to [publish a new event to the PubSub component](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L225), it will look like this: 

```golang
if err := s.APIClient.PublishEvent(ctx, PUBSUB_NAME, PUBSUB_TOPIC, eventJson); err != nil {
			...
}

```

As we have seen, Dapr provides Application-Level APIs for application developers to use regarding the programming language that they are using. These APIs abstract away the complexities of setting up and managing application infrastructure components, enabling platform teams to have more flexibility to focus on fine-tuning how applications will interact with them without pushing teams to change the application source code. 

Next, let's talk about feature flags. This topic involves not only developers but also enables product managers and roles closer to the business to decide when certain features should be exposed.


## Feature Flags for everyone

The [OpenFeature](https://openfeature.dev/) project aims to standardize how to consume Feature Flags from applications written in different languages. 

In this short tutorial, we will look at how the Conference Application `v2.0.0` uses Open Feature and, more specifically, the `flagd` provider to enable feature flags across all application services. For this example, to keep it simple, I've used the `flagd` provider that allows us to define our feature flag configurations inside a Kubernetes `ConfigMap`.

![openfeature](imgs/conference-app-openfeature.png)

In the same way as Dapr APIs, the idea here is to have a consistent experience no matter which provider we have selected. If the platform team wants to switch providers, for example, LaunchDarkly or Split, there will be no need to change how features are fetched or evaluated. The platform team will be able to swap providers to whichever they think is best. 

`v2.0.0` created a ConfigMap called `flag-configuration` containing the Feature Flag that the application's services will use. 

You can get the flag configuration json file included in the ConfigMap by running: 

```shell
kubectl get cm flag-configuration -o go-template='{{index .data "flag-config.json"}}'
```

You should see the following output: 

```json
{
  "flags": {
    "debugEnabled": {
      "state": "ENABLED",
      "variants": {
        "on": true,
        "off": false
      },
      "defaultVariant": "off"
    },
    "callForProposalsEnabled": {
      "state": "ENABLED",
      "variants": {
        "on": true,
        "off": false
      },
      "defaultVariant": "on"  
    },
    "eventsEnabled": {
      "state": "ENABLED",
      "variants": {
        "all": {
          "agenda-service": true,
          "notifications-service": true,
          "c4p-service": true
        },
        "decisions-only": {
          "agenda-service": false,
          "notifications-service": false,
          "c4p-service": true
        },
        "none": {
          "agenda-service": false,
          "notifications-service": false,
          "c4p-service": false
        }
      },
      "defaultVariant": "all"
    }
  }
}
```

There are three feature flags defined for this example: 
- `debugEnabled` is a boolean flag that allows us to turn on and off the Debug tab in the back office of the application. This replaces the need for an environment variable that we used in `v1.0.0`. We can turn the debug section on and off without restarting the application frontend container.
- `callForProposalsEnabled` This boolean flag allows us to disable the **Call for Proposals** section of the application. As conferences have a window to allow potential speakers to submit proposals, when that period is over, this section can be hidden away. Having to release a specific version to just turn off that section would be too complicated to manage, hence having a feature flag for this makes a lot of sense. We can make this change without the need to restart the application frontend container.
- `eventsEnabled` is an object feature flag, this means that it contains a structure and allows teams to define complex configurations. In this case, I've defined different profiles of flags to configure which service can emit events (Events tab in the application back office). By default all services emit events, but by changing the `defaultVariant` value to `none` we can disable events for all services, without the need to restart any container.


You can patch the ConfigMap to turn on the debug feature by following these steps. First, fetch the content of the `flag-config.json` file located inside the ConfigMap and store it locally.
```shell
kubectl get cm flag-configuration -o go-template='{{index .data "flag-config.json"}}' > flag-config.json
```

Modify the content of this file, for example turn on the debug flag: 

```json
{
  "flags": {
    "debugEnabled": {
      "state": "ENABLED",
      "variants": {
        "on": true,
        "off": false
      },
    **"defaultVariant": "on"**
    },
    ...
```
Then patch the existing `ConfigMap`: 

```shell
kubectl create cm flag-configuration --from-file=flag-config.json=flag-config.json --dry-run=client -o yaml | kubectl patch cm flag-configuration --type merge --patch-file /dev/stdin
```

After 20 seconds or so, you should see the Debug Tab in the application's back office section

![debug feature flag](imgs/feature-flag-debug-tab.png)

You can see that feature flags are now also displayed in this tab. 

Now submit a new proposal and approve it. You will see that in the `Events` tab, Events will be displayed. 

![events for approved proposal](imgs/feature-flag-events-for-proposal.png)


If you repeat the previous process and change the `eventsEnabled` feature flag to `"defaultVariant": "none"`, all services will stop emitting events. Submit a new proposal from the application user interface and approve it, then check the `Events` tab to validate that no event has been emitted. Remember that when changing the `flag-configuration` ConfigMap, `flagd` needs to wait around 10 seconds to refresh the content of the ConfigMap. If you have the Debug tab enabled you can refresh that screen until you see that the value has changed. 

**Notice that this feature flag is being consumed by all services that evaluate the flag before sending any event. **

Finally, if you change the `callForProposalsEnabled` feature flag `"defaultVariant": "off"`, the Call for Proposal menu option will disappear from the application frontend. 

![no call for proposals feature flag](imgs/feature-flag-no-c4p.png)


While we are still using a `ConfigMap` to store the feature flags configurations we have achieved some important improvements that enable teams to go faster. Developers can keep releasing new features to their application services that then product managers (or stakeholders) can decide when to enable/disable. Platform Teams can define where the feature flags will be stored (a managed service or local storage). By using a standard specification driven by a community composed of Feature Flag vendors enables our application development teams to make use of feature flags without defining all the technical aspects required to implement these mechanisms in-house. 

In this example, we haven't used more advanced features to evaluate feature flags like [context-based evaluations](https://openfeature.dev/docs/reference/concepts/evaluation-context#providing-evaluation-context), which can use for example, the geo-location of the user to provide different values for the same feature flag, or [targetting keys](https://openfeature.dev/docs/reference/concepts/evaluation-context#targeting-key). It is up to the reader to go deeper into OpenFeature capabilities as well as which other [Open Feature flag providers](https://openfeature.dev/docs/reference/concepts/provider) are available.  

## Limpieza

Si quieres eliminar el Cluster KinD creado para este tutorial, ejecuta:

```shell
kind delete clusters dev
```


## Próximos Pasos

Un siguiente paso natural sería ejecutar la versión `v2.0.0` en la infraestructura provisionada por Crossplane, como hicimos en el [Capítulo 5](../chapter-5/README-es.md).
Esto puede ser gestionado por nuestro esqueleto de plataforma, que se encargará de configurar el gráfico de Helm de la Aplicación de Conferencias para conectarse a la infraestructura provisionada, conectando los recursos de Kubernetes entre sí.
Si te interesa este tema, he escrito un artículo sobre por qué herramientas como Crossplane y Dapr están destinadas a trabajar juntas: [https://blog.crossplane.io/crossplane-and-dapr/](https://blog.crossplane.io/crossplane-and-dapr/).

Otra extensión simple, pero muy útil del código de la aplicación sería asegurarse de que el Servicio de Convocatoria de Propuestas lea la bandera de características `callForProposalsEnabled` y devuelva errores significativos cuando esta función esté deshabilitada.
La implementación actual solo elimina la entrada del menú "Convocatoria de Propuestas", lo que significa que si envías una solicitud `curl` a las API, la funcionalidad aún debería funcionar.

## Resumen y Contribuir

En este tutorial, hemos explorado las API a nivel de aplicación utilizando Dapr y las banderas de características utilizando OpenFeature.
Las API a nivel de aplicación, como las expuestas por los Componentes de Dapr, pueden ser aprovechadas por los equipos de desarrollo de aplicaciones, ya que la mayoría de las aplicaciones estarán interesadas en almacenar y leer estado, emitir y consumir eventos, y políticas de resiliencia para comunicaciones de servicio a servicio.
Las banderas de características también pueden ayudar a acelerar el proceso de desarrollo y lanzamiento al permitir que los desarrolladores sigan lanzando funciones mientras que otros stakeholders deciden cuándo habilitar o deshabilitar estas funciones.

¿Quieres mejorar este tutorial?
Crea un Issue, envíame un mensaje en [Twitter](https://twitter.com/salaboy) o envía un [Pull Request](https://github.com/salaboy/platforms-on-k8s/pulls).

