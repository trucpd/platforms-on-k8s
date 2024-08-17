# Capítulo 7:: Preocupaciones Compartidas de Aplicaciones

---
_🌍 Disponible en: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md) | [Español](README-es.md)

> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

En este tutorial paso a paso, veremos como usar [Dapr](https://dapr.io) para proporcionar API a nivel de Aplicación para resolver los problemas que la mayoría de las Aplicaciones Distribuidas tienen que resolver.

Luego miraremos [OpenFeature](https://openfeature.dev), un proyecto que está hecho para estandarizar las _feature flags_ y así los equipos de Desarrollo puedan seguir lanzando nuevas funcionalidades y los stakeholders puedan decidir cuándo habilitar/desactivar estas características para sus clientes.

Debido a que ambos proyectos están enfocados en proveer a los desarrolladores con nuevas API y herramientas que utilizar dentro del código de sus servicios, desplegaremos una nueva versión de la aplicación (`v2.0.0`).
Encontrarás todos los cambios requeridos para esta nueva versión en la rama `v2.0.0` de este repositorio.
[Puedes comparar las diferencias entre las ramas aquí](https://github.com/salaboy/platforms-on-k8s/compare/v2.0.0).

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
Esta versión de la aplicación también añade feature flags OpenFeature utilizando `flagd`.

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
esta suscripción reenvía los eventos a la ruta `/api/new-events/` para que las aplicaciones Dapr listadas
en la sección `Scopes`.
Solo la aplicación Frontend necesita exponer el endpoint para recibir eventos,
en este caso el sidecar de Dapr (`daprd`)
espera los mensajes entrantes en el componente PubSub llamado `conference-conference-pubsub` y reenvía todos los mensajes al endpoint de la Aplicación.

Esta versión de la aplicación elimina dependencias de la aplicación tales como el cliente de Kafka de todos los servicios y el cliente de Redis del servicio de Agenda.


![services without deps](imgs/conference-app-dapr-no-deps.png)

Además de eliminar las dependencias y hacer los contenedores más pequeños,
al consumir la API de los componentes Dapr, permitimos que el equipo de plataformas defina cómo se configuran esos componentes y sobre qué infraestructura.
Configurar la misma aplicación para usar servicios administrados de Google Cloud Platform como [Google PubSub](https://cloud.google.com/pubsub) o la [MemoryStore databases](https://cloud.google.com/memorystore) no requiere cambios en el código de la aplicación o agregar nuevas dependencias, solo se necesitan nuevas configuraciones de componentes de Dapr.

![in gcp](imgs/conference-app-dapr-and-gcp.png)

Finalmente, ya que todo se trata de permitir que los desarrolladores utilicen las API a nivel de aplicación, miremos cómo se ve desde la perspectiva del servicio.
Como los servicios están escritos en Go, he decidido añadir la SDK de Dapr para Go (que es opcional).

Cuando el servicio de Agenda quiere almacenar o leer datos del componente Statestore, puede utilizar el cliente Dapr para realizar estar operaciones, por ejemplo:

```golang
agendaItemsStateItem, err := s.APIClient.GetState(ctx, STATESTORE_NAME, fmt.Sprintf("%s-%s", TENANT_ID, KEY), nil)
```

La referencia `APIClient` es una [instancia del cliente Dapr, que se ha iniciado aquí](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L397)

Todo lo que la aplicación necesita saber es el nombre del componente Statestore (`STATESTORE_NAME`) y la clave (`KEY`) para localizar el dato que quiere recuperar.

Cuando la aplicación necesita [almacenar un estado en la Statestore, se utiliza asi](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L197C2-L199C3):

```golang
if err := s.APIClient.SaveState(ctx, STATESTORE_NAME, fmt.Sprintf("%s-%s", TENANT_ID, KEY), jsonData, nil); err != nil {
		...
}
```  

Finalmente,
si la aplicación necesita [publicar un nuevo evento en el componente PubSub](https://github.com/salaboy/platforms-on-k8s/blob/v2.0.0/conference-application/agenda-service/agenda-service.go#L225), se utiliza asi:

```golang
if err := s.APIClient.PublishEvent(ctx, PUBSUB_NAME, PUBSUB_TOPIC, eventJson); err != nil {
			...
}

```

Como hemos visto, Dapr provee API de nivel de aplicación para que las usen los desarrolladores de aplicaciones, sin importar el lenguaje que utilicen.
Estas API abstraen la complejidad de configurar y administrar componentes de infraestructura de la aplicación, permitiendo que los equipos tengan más flexibilidad para enfocarse en configurar cómo las aplicaciones interactúan con ellas sin que se requieran cambios en el código de la aplicación.

Ahora, hablemos de feature flags.
Este tópico involucra no solo a los desarrolladores, sino que permite a los administradores de producto y roles más cercanos a la empresa decidir cuándo y qué características deben ser expuestas.

## Feature Flags para todos

El proyecto [OpenFeature](https://openfeature.dev/) tiene como objetivo estandarizar cómo consumir Feature Flags desde aplicaciones escritas en diferentes lenguajes.

En este breve tutorial,
miraremos cómo la aplicación de conferencias `v2.0.0` utiliza Open Feature y,
más específicamente,
el proveedor `flagd` para permitir el uso de feature flags en todos los servicios.
Para este ejemplo, manteniendo la simplicidad, he utilizado el proveedor `flagd` que nos permite definir nuestra configuración de feature flag dentro de un `ConfigMap`de Kubernetes.

![openfeature](imgs/conference-app-openfeature.png)

De la misma manera que las API de Dapr, la idea aquí es tener una experiencia consistente, sin importar cuál proveedor hemos elegido.
Si el equipo de plataformas decide cambiar el proveedor, por ejemplo, LaunchDarkly o Split, no hará falta cambiar cómo se obtienen o evalúan los feature flags.
El equipo de plataformas será capaz de cambiar proveedores a cualquiera que piensen que es el mejor.
 
La versión `v2.0.0` crea un ConfigMap llamado `flag-configuration`que contiene la configuración de feature flags que utilizarán los servicios.

Puedes obtener el fichero json con la configuración de los feature flags incluido en el ConfigMap ejecutando:

```shell
kubectl get cm flag-configuration -o go-template='{{index .data "flag-config.json"}}'
```

Deberías ver algo similar a la siguiente salida:

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

Existen 3 feature flags definidas para este ejemplo:

- `debugEnabled` es una marca booleana que nos permite habilitar o deshabilitar la sección Debug en la sección Back Office de la aplicación. Esto reemplaza la necesidad de una variable de entorno que utilizábamos en `v1.0.0`. Podemos habilitar o deshabilitar la sección Debug sin reiniciar el contenedor de la aplicación frontend.
- `callForProposalsEnabled` Es una marca booleana que nos permite deshabilitar la sección **Call for Proposals** de la aplicación. Como las conferencias tienen un período de tiempo para que los potenciales participantes envíen propuestas, cuando este período finaliza, podemos deshabilitar esta sección. Tener que lanzar una versión específica simplemente para habilitar o deshabilitar esa sección sería muy complicado de manejar, por lo tanto, tener una feature flag para ello tiene mucho sentido. Podemos hacer este cambio sin tener que reiniciar el contenedor de la aplicación frontend.
- `eventsEnabled` es un objeto feature flag, lo que significa que contiene una estructura y permite a los equipos de plataformas definir configuraciones complejas. En este caso, hemos definido distintos perfiles de flags para configurar cual servicio puede emitir eventos (sección Eventos en la sección Back Office de la aplicación). Por defecto todos los servicios emiten eventos, pero cambiando el valor de `defaultVariant` a `none` podemos deshabilitar los eventos para todos los servicios, sin necesidad de reiniciar ningún contenedor.

Puedes modificar el ConfigMap para habilitar la feature debug siguiendo estos pasos.
Primero,
obtén el contenido del fichero `flag-config.json` localizado dentro del ConfigMap
y guárdalo localmente.

```shell
kubectl get cm flag-configuration -o go-template='{{index .data "flag-config.json"}}' > flag-config.json
```

Modifica el contenido de este fichero, por ejemplo, activa la flag debug:

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
Luego, modifica el `ConfigMap` existente:

```shell
kubectl create cm flag-configuration --from-file=flag-config.json=flag-config.json --dry-run=client -o yaml | kubectl patch cm flag-configuration --type merge --patch-file /dev/stdin
```

Después de unos 20 segundos, deberías ver la sección Debug en la sección Back Office de la aplicación.

![debug feature flag](imgs/feature-flag-debug-tab.png)

Puedes ver que las feature flags ahora se muestran en esta sección

Ahora envía una nueva propuesta y apruébala.
Verás que se mostrarán los eventos en la sección `Events`.

![events for approved proposal](imgs/feature-flag-events-for-proposal.png)

Si repites el proceso previo y cambias el valor de `eventsEnabled` a `defaultVariant` a `none`,
todos los servicios dejarán de emitir eventos.
Envía una nueva propuesta desde la interfaz de usuario de la aplicación y apruébala,
luego revisa la sección `Events` para verificar que no se emitieron eventos.
Recuerda que cuando cambias el `ConfigMap` `flag-configuration`,
`flagd` necesita esperar alrededor de 10 segundos para refresacar el contenido del ConfigMap.
Si tienes la sección Debug habilitada puedes refrescar esa pantalla hasta que veas que el valor ha cambiado. 

**Notemos que esta feature flag es consumida por todos los servicios que evalúan la flag antes de enviar un evento.**

Finalmente, si cambias la feature flag `callForProposalsEnabled` a `defaultVariant` a `off`, la sección **Call for Proposals** de la aplicación desaparecerá.

![no call for proposals feature flag](imgs/feature-flag-no-c4p.png)

Mientras seguimos utilizando un `ConfigMap` para almacenar la configuración de feature flags, hemos logrado algunas mejoras importantes que permiten a los equipos ir más rápido.
Los desarrolladores pueden continuar lanzando nuevas características a sus servicios de aplicación que luego los administradores de producto (o stakeholders) pueden deciidir cuando habilitarlas o deshabilitarlas.
Los equipos de plataformas pueden definir donde se almacenarán las feature flags (en un servicio de almacenamiento o en el disco local).
Al usar una especificación estándar, impulsada por una comunidad compuesta de proveedores de feature flags permite a nuestros equipos de desarrollo de aplicaciones hacer uso de las feature flags sin definir todos los aspectos técnicos requeridos para implementar estos mecanismos internamente. 

En este ejemplo, no hemos utilizado características más evanzadas para evaluar feature flags como [evaluación basada en el contexto](https://openfeature.dev/docs/reference/concepts/evaluation-context#providing-evaluation-context), que puede usar, por ejemplo, la geolocalización de un usuario para proveer valores diferentes para las mismas feature flags, o [tarjetas de evaluación](https://openfeature.dev/docs/reference/concepts/evaluation-context#targeting-key). Depende del lector el profundizar en las capacidades de OpenFeature así como qué otros [proveedores de feature flags](https://openfeature.dev/docs/reference/concepts/provider) están disponibles.


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

En este tutorial, hemos explorado las API a nivel de aplicación utilizando Dapr y las feature flags utilizando OpenFeature.
Las API a nivel de aplicación, como las expuestas por los Componentes de Dapr, pueden ser aprovechadas por los equipos de desarrollo de aplicaciones, ya que la mayoría de las aplicaciones estarán interesadas en almacenar y leer estado, emitir y consumir eventos, y políticas de resiliencia para comunicaciones de servicio a servicio.
Las feature flags también pueden ayudar a acelerar el proceso de desarrollo y lanzamiento al permitir que los desarrolladores sigan lanzando funciones mientras que otros stakeholders deciden cuándo habilitar o deshabilitar estas funciones.

¿Quieres mejorar este tutorial?
Crea un Issue, envíame un mensaje en [Twitter](https://twitter.com/salaboy) o envía un [Pull Request](https://github.com/salaboy/platforms-on-k8s/pulls).

