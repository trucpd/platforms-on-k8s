# Keptn Lifecycle Toolkit, out of the box Deployment Frequency

---
_🌍 Disponible en_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md)| [Español](README-es.md)

> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

En este breve tutorial exploraremos Ketpn Lifecycle Toolkit para monitorizar, observar y reaccionar a eventos del ciclo de vida de nuestras aplicaciones nativas en la nube.

## Instalación

Necesitas un Clúster de Kubernetes para instalar [Keptn KLT](https://keptn.sh).
Puedes crear uno usando Kubernetes KinD como lo hicimos en el [Capítulo 2](https://github.com/salaboy/platforms-on-k8s/blob/main/chapter-2/README-es.md#creando-un-clúster-local-con-kubernetes-kind)

Luego podemos instalar el toolkit de Keptn lifecycle (KLT, en inglés).
Esto se puede hacer usualmente con el chart de Helm de, pero para este ejemplo también queremos instalar Prometheus, Jaeger y Grafana para tener dashboards.
Por esta razón, basado en el repositorio de KLT, utilizaremos un Makefile para instalar todas las herramientas que necesitamos para este ejemplo.

Ejecuta: 

```shell
make install
```

**Nota*: EL proceso de instalación tomará unos minutos para instalar todas las herramientas necesarias.

Finalmente, necesitamos hacerle saber a KLT cuál _namespace_ queremos monitorizar, y para ese necesitamos anotar los namespaces:

```shell
kubectl annotate ns default keptn.sh/lifecycle-toolkit="enabled"
```

## Keptn Lifecycle toolkit en acción

Keptn utiliza anotaciones estándar de Kubernetes para reconocer y monitorizar nuestras cargas de trabajo.
Los Despliegues de Kubernetes usados por la Aplicación de Conferencias están anotados con las siguientes anotaciones, por ejemplo el Servicio Agenda:
 

```shell
        app.kubernetes.io/name: agenda-service
        app.kubernetes.io/part-of: agenda-service
        app.kubernetes.io/version: {{ .Values.services.tag  }}
```

Estas anotaciones permiten a las herramientas entender un poco más nuestras cargas de trabajo, por ejemplo, en el caso que las herramientas sepanr que el nombre del servicio es `agenda-service`.
Podemos usar la anotación `app.kubernetes.io/part-of` para agregar múltiples servicios para que sean parte de la misma aplicación. Para este ejemplo, queremos que cada servicio sea una entidad separada para monitorizarla individualmente.

En este ejemplo usaremos también una `KeptnTask`, que nos permite realizar tareas de pre- y post-despliegue.
Puedes desplegar el siguiente ejemplo extremadamente sencillo usando `KeptnTaskDefinition`:

```yaml
apiVersion: lifecycle.keptn.sh/v1alpha3
kind: KeptnTaskDefinition
metadata:
  name: stdout-notification
spec:
  function:
    inline:
      code: |
        let context = Deno.env.get("CONTEXT");
        console.log("Tarea Keptn ejecutada con el contexto: \n");
        console.log(context);

```

Como puedes ver, esta tarea solo imprimie el contexto de su ejecuación, pero aquí es donde puedes construir integraciones con otros proyectos o llamar sistemas externos.
Si mirar los ejemplos de Keptn, encontrarás `KeptnTaskDefinition` para conectar con Slack, por ejemplo, o ejecutar pruebas de carga, o validar que los despligues funcionan de forma esperada luego de ser actualizados.
Estas tareas usan [Deno](https://deno.land/), un _runtime_ de JavaScript seguro y con soporte para Typescript por defecto, Python 3 o directamente una imagen de un contenedor.

Al ejecutar: 

```shell
kubectl apply -f keptntask.yaml
```

KeptnTaskDefinition les permite a los equipos de plataformas crear tareas reutilizables que se pueden ejecutar en los hooks de pre/post-despliegue de nuestra aplicación. Al añadir la siguiente anotación a nuestras cargas de trabajo (despliegues en este caso), Keptn ejecutará automáticamente la tarea `stdout-notification`, en este caso luego de realizar el despliegue (y luego de cualquier actualización).

```shell
  keptn.sh/post-deployment-tasks: stdout-notification
``` 

Vamos a desplegar la aplicación de Conferencias, y vamos a abrir los dashboards de Jaeger y Grafana.
Ejecuta en pestañas separadas: 
 

```shell
make port-forward-jaeger
```

Puedes abrir tu navegador en la URL `http://localhost:16686/`, y deberías ver:

![jaeger](../imgs/jaeger.png)

y luego en un terminal separado:

```shell
make port-forward-grafana
```

Puedes abrir tu navegador en la URL [http://localhost:3000/](http://localhost:3000).
Utiliza las credenciales `admin/admin` y deberías ver:

![grafana](../imgs/grafana.png)

Vamos a desplegar ahora la aplicación de Conferencias como lo hicimos en el capítulo 2:

```shell
helm install conference oci://registry-1.docker.io/salaboy/conference-app --version v1.0.0
```

Revisa los dashboards de Jaeger y Grafana, y por defecto Keptn Workloads rastreará la frecuencia de despliegue.

En Grafana, ve a `Dashboards` -> `Keptn Applications`.
Verás un cuadro desplegable que permite seleccionar los servicios de las diferentes aplicaciones.
Revisa el servicio de Notificaciones.
Debido a que solo hemos desplegado la primera versión del despliegue, no hay mucho que ver, pero el dashboard se volverá más interesante cuando lancemos nuevas versiones de nuestros servicios.

Por ejemplo, edita el despliegue del servicio de notificaciones y actualiza la anotación `app.kubernetes.io/version` con el valor `v1.1.0` y actualiza la etiqueta utilizada por la imagen del contendedor a `v1.1.0`.

```shell
kubectl edit deploy conference-notifications-service-deployment
```

Luego de realizar los cambios,
y la nueva versión está ejecutándose, revisa nuevamente los dashboards.
En Grafana, verás que estamos en el segundo despliegue exitoso,
que el promedio entre despliegues fue de 5,83 minutos en mi ambiente,
y que `v1.0.0` tomó 641 s mientras que `v1.1.0` tomó solo 40 s. Definitivamente, hay mucho que podemos mejorar ahí.
 

![grafana](../imgs/grafana-notificatons-service-v1.1.0.png)

Si miras las trazas en Jaeger,
verás que `lifecycle-operator`, uno de los components núcleo en Keptn está monitorizando nuestros recursos de despliegue y realizando operaciones del ciclo de vida, como por ejemplo llamar a tareas de pre y post-despliegue.

![jager](../imgs/jaeger-notifications-service-v1.1.0.png)

Estas tareas se ejecutan como Jobs de Kubernetes en el mismo namespace donde se ejecutan las cargas de trabajo.
Puedes mirar los logs de estas tareas siguiendo los registros de los jobs del pod.

```shell
kubectl get jobs
NAME                                   COMPLETIONS   DURATION   AGE
post-stdout-notification-25899-78387   1/1           3s         66m
post-stdout-notification-28367-11337   1/1           4s         61m
post-stdout-notification-54572-93558   1/1           4s         66m
post-stdout-notification-75100-85603   1/1           3s         66m
post-stdout-notification-77674-78421   1/1           3s         66m
post-stdout-notification-93609-30317   1/1           3s         23m
```

El Job con el id `post-stdout-notification-93609-30317` fue ejecutado luego que realicé la actualización en el despliegue del Servicio de Notificaciones.  

```shell
> kubectl logs -f post-stdout-notification-93609-30317-vvwp4
Keptn Task Executed with context: 

{"workloadName":"notifications-service-notifications-service","appName":"notifications-service","appVersion":"","workloadVersion":"v1.1.0","taskType":"post","objectType":"Workload"}

```

## Próximos pasos

Te recomiendo encarecidamente que te familiarices con las características y funcionalidades de Keptn Lifecycle toolkit, ya que solo hemos visto lo básico en este breve tutorial.
Revisa los conceptos de [KeptnApplication](https://lifecycle.keptn.sh/docs/concepts/apps/) para tener más control sobre cómo se despliegan tus servicios, porque Keptn te permite reglas específicas sobre cuáles servicios y qué versiones es permitido desplegar. 

Al agrupar múltiples servicios como parte de la misma aplicación de Kubernetes, usando la anotación `app.kubernetes.io/part-of`, puedes realizar acciones pre.
y post en un grupo de servicios, permitiéndote validar que no solo los servicios individuales funcionan como se espera, sino que todo el conjunto lo hace.



