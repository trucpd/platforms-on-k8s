# Environment Pipelines

---
_🌍 Available in_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md)
> **Nota:** Presentado por la fantástica comunidad de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!
---

En este breve tutorial, configuraremos el Pipeline de nuestro entorno de ensayo utilizando [ArgoCD](https://argo-cd.readthedocs.io/en/stable/). Configuraremos el ambiente para que contenga una instancia de la `Aplicación para Conferencias`.

Definiremos la configuración del ambiente de ensayo utilizando un repositorio de Git. El [directorio `argo-cd/staging`](argo-cd/staging/) contiene la definición de un Helm chart que puede ser sincronizado con multiples clusters de Kubernetes.

## Prerequisites and installation

- Necesitamos un Clúster de Kubernetes. En este tutorial, utilizaremos Kubernetes [KinD](https://kind.sigs.k8s.io/)
- Instala ArgoCD en tu Clúster, [sigue estas instrucciones](https://argo-cd.readthedocs.io/en/stable/getting_started/), opcionalmente puedes instalar el CLI de `argocd`
- You can fork/copy [this repository](http://github.com/salaboy/platforms-on-k8s/) como si quisieras cambiar la configuración de la aplicación, necesitarás tener accesos de escritura al repositorio.. Utilizaremos el directorio `chapter-4/argo-cd/staging/`

[Crea un Cluster KinD, tal y como hicimos en el capítulo 2](../chapter-2/README-es.md#creando-un-cl%C3%BAster-local-con-kubernetes-kind).

Una vez que el clúster esté en funcionamiento con el controlador nginx-ingress, vamos a instalar ArgoCD en el clúster: 

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Deberías ver algo como lo siguiente: 

```shell
> kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
namespace/argocd created
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
serviceaccount/argocd-dex-server created
serviceaccount/argocd-notifications-controller created
serviceaccount/argocd-redis created
serviceaccount/argocd-repo-server created
serviceaccount/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-applicationset-controller created
role.rbac.authorization.k8s.io/argocd-dex-server created
role.rbac.authorization.k8s.io/argocd-notifications-controller created
role.rbac.authorization.k8s.io/argocd-server created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-redis created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
configmap/argocd-cm created
configmap/argocd-cmd-params-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-notifications-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-notifications-secret created
secret/argocd-secret created
service/argocd-applicationset-controller created
service/argocd-dex-server created
service/argocd-metrics created
service/argocd-notifications-controller-metrics created
service/argocd-redis created
service/argocd-repo-server created
service/argocd-server created
service/argocd-server-metrics created
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
statefulset.apps/argocd-application-controller created
networkpolicy.networking.k8s.io/argocd-application-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-applicationset-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-dex-server-network-policy created
networkpolicy.networking.k8s.io/argocd-notifications-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-redis-network-policy created
networkpolicy.networking.k8s.io/argocd-repo-server-network-policy created
networkpolicy.networking.k8s.io/argocd-server-network-policy created
```

Puedes acceder la interfaz de usuario de ArgoCD utilizando `port-forward`, en una **nueva terminal** ejecuta:

```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Nota**: Debes esperar que los pods de ArgoCD hayan iniciado. La primera vez que hagas esto, tomará más tiempo porque necesita obtener las imagenes de los contenedores desde Internet.

Puedes acceder a la interfaz de usuario, apuntando tu navegador a [http://localhost:8080](http://localhost:8080)

<img src="imgs/argocd-warning.png" width="600">

**Nota**: Por defecto la instalación funciona utilizando HTTP y no HTTPS, Por lo tanto, necesitas aceptar la advertencia (Presiona el botón "Advanced" en Chrome) y procede (**Proceed to localhost (unsafe)**). 

<img src="imgs/argocd-proceed.png" width="600">

Eso debería llevarte a la página de inicio de sesión:

<img src="imgs/argocd-login.png" width="600">

El usuario es `admin`, y para obtener la contraseña para el Dashboard de ArgoCD ejecutamos: 

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Una vez dentro, deberías ver la pantalla de inicio de sesión, vacía: 

<img src="imgs/argocd-dashboard.png" width="600">

Ahora, vamos a configurar nuestro ambiente de ensayo.


# Configurando nuestra aplicación para el ambiente de ensayo

Para este tutorial, utilizaremos un solo namespace para representar nuestro Ambiente de Ensayo. Con ArgoCD no hay límites, y nuestro ambiente de ensayo podría ser un Clúster de Kubernetes totalmente diferente. 

Primero, creemos el namespace para nuestro Ambiente de Ensayo:

```shell
kubectl create ns staging
```

Deberías ver algo como lo siguiente:

```shell
> kubectl create ns staging
namespace/staging created
```

Nota: Alternativamente, puedes usar la opción para crear de manera automática un namespace en la creación de la aplicación ArgoCD.

Una vez tenemos ArgoCD instalado, podemos acceder a la interfaz de usuario para configurar el proyecto.

<img src="imgs/argocd-dashboard.png" width="600">

Presiona el botón **"+ New App"** y utiliza los siguientes detalles para configurar el proyecto:

<img src="imgs/argocd-app-creation.png" width="600">

Estas son los parámetros de entrada utilizados para crear la aplicación: 

- Application Name: "staging-environment"
- Project: "default"
- Sync Policy: "Automatic"
- Source Repository: [https://github.com/salaboy/platforms-on-k8s](https://github.com/salaboy/platforms-on-k8s) (here you can point to your fork)
- Revision: "HEAD"
- Path: "chapter-4/argo-cd/staging/"
- Cluster: "https://kubernetes.default.svc" 
- Namespace: "staging"

<img src="imgs/argocd-app-creation2.png" width="600">

Los demás valores permanecerán con sus valores predeterminados y presione **Create** en la parte superior 

Una vez la aplicación sea creada, automáticamente sincronizará los cambios, dado que seleccionamos el modo **Automatic** mode.

<img src="imgs/argocd-syncing.png" width="600">

Puedes expandir la aplicación haciendo click en ella para ver la vista completa de todos los recursos que están siendo creados:

<img src="imgs/app-detail.png" width="600">

Si estás ejecutando en un ambiente local, siempre puedes acceder a la aplicación usando `port-forward`, en una **nueva terminal** ejecuta:

```shell
kubectl port-forward svc/frontend -n staging 8081:80
```

Espera que los pods de la aplicación estén en funcionamiento y puedes acceder a la aplicación apuntanto tu navegador a [http://localhost:8081](http://localhost:8081).

<img src="imgs/app-home.png" width="600">


Como siempre, puedes monitorear el estado de los pods y servicios utilizando `kubectl`. Para verificar si los pods de la aplicación están listos, puedes ejecutar: 

```shell
kubectl get pods -n staging
```

Deberías ver algo como lo siguiente:

```shell
> kubectl get pods -n staging
NAME                                                              READY   STATUS    RESTARTS        AGE
stating-environment-agenda-service-deployment-6c9cbb9695-xj99z    1/1     Running   5 (6m ago)      8m4s
stating-environment-c4p-service-deployment-69d485ffd8-q96z4       1/1     Running   5 (5m52s ago)   8m4s
stating-environment-frontend-deployment-cd76bdc8c-58vzr           1/1     Running   5 (6m3s ago)    8m4s
stating-environment-kafka-0                                       1/1     Running   0               8m4s
stating-environment-notifications-service-deployment-5c9b5bzb5p   1/1     Running   5 (6m13s ago)   8m4s
stating-environment-postgresql-0                                  1/1     Running   0               8m4s
stating-environment-redis-master-0                                1/1     Running   0               8m4s
```

**Nota**: Unos cuantos reinicios está bien OK (columna RESTARTS), dado que algunos servicios deben esperar que la infraestructura correspondiente esté en funcionamiento (Redis, PostgreSQL, Kafka) para ellos también entrar en funcionamiento sin ningún problema.

## Cambiando la configuración de la Aplicación en el ambiente de ensayo

Para actualizar la versión de las configuraciones de tus servicios, puedes actualizar los archivos ubicados en el archivo [Chart.yaml](argo-cd/staging/Chart.yaml) o [values.yaml](argo-cd/staging/values.yaml) ubicado dentro del directorio [staging](staging/).

Para este ejemplo, puedes cambiar la configuración de la aplicación, actulizando los detalles y parámetros de la aplicación de ArgoCD. 

Si bien no harás esto con tus aplicaciones, simplemente estamos simulando un cambio en el repositorio de GitHub donde se encuentra definido nuestro Ambiente de Ensayo.

<img src="imgs/argocd-change-parameters.png" width="600">

Continúa editando los detalles y parámetros de la aplicación, selecciona `values-debug-enabled.yaml` para el archivo de valores que queremos utilizar con esta aplicación. Este archivo establece el indicador (flag) de depuración en el servicio de la interfaz de usuario y nos simula cambiar el archivo `values.yaml`que utilizamos para la primera instalación.

<img src="imgs/argocd-new-values.png" width="600">

Debido a que hemos estado utilizando port-forwarding, es posible que necesites ejecutar este comando nuevamente:

```shell
kubectl port-forward svc/frontend -n staging 8081:80
```

Esto se debe a que el Servicio de la Interfaz de Usuario será reemplazado por la nueva versión configurada, por lo tanto el redireccionamiento de puerto (port-forwarding) necesita reiniciarse para que apunte al nuevo pod del servicio.

Una vez la Interfaz de Usuario esté en funcionamiento, deberías ver la pestaña de Debug en la sección de Back Office:
![](imgs/app-debug.png)

## Limpiando los cambios

Si  quieres deshacerte del Clúster KinD que hemos creado para este tutorial, puedes ejecutar: 

```shell
kind delete clusters dev
```

## Siguientes Pasos

ArgoCD es solo un proyecto para implementar GitOps, ¿puedes replicar este tuorial utilizando FluxCD? ¿Cuál prefieres? ¿Tu organización está actualmente utilizando una herramienta de GitOps? ¿Qué se necesitaría para implementar el esqueleto móvil de la Aplicación de Conferencia en un clúster de Kubernetes utilizando esa herramienta?

¿Puedes crear otro ambiente, digamos un `production-environment` y describir el flujo que se necesitaría seguir un nuevo despliegue del `notifications-service` desde el Ambiente de Ensayo al Ambiente de Producción? ¿Dónde almacenarías la configuración del Ambiente de Producción? 

## Resumen y Contribución

En este tutorial, creamos nuestro **Ambiente de Ensayo** utilizando una aplicación ArgoCD. Esto nos permitió sincronizar la configuración ubicada dentro de un repositorio de GitHub con nuestro clúster de Kubernetes en ejecución en KinD. Si realizas cambios en el contenido del repositorio de GitHub y actualizas la aplicación de ArgoCD, este notará que nuestro ambiente no está sincronizado. Si utilizamos una estrategia de sincronización automatizada, ArgoCD ejecutará el paso de sincronización automáticamente cada vez que note que se han producido cambios en la configuración. Para obtener más información, consulta el [sitio web del proyecto](https://argo-cd.readthedocs.io/en/stable/) o [mi blog](https://www.salaboy.com). 

Do you want to improve this tutorial? Create an issue, drop me a message on [Twitter](https://twitter.com/salaboy), or send a Pull Request.
