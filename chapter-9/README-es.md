# Chapter 9 :: Midiendo tus plataformas

---
_🌍 Disponible en_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md)| [Español](README-es.md)

> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

Este capítulo cubre dos tutoriales diferentes, sobre cómo usar las métricas de DORA para medir el rendimiento de tu iniciativa de plataforma.

- [Métricas de DORA y CloudEvents](dora-cloudevents/README.md)
- [Keptn Lifecycle Toolkit](keptn/README.md)

## Resumen

Los tutoriales cubiertos en este capítulo tienen el propósito de mostrar dos formas completamente distintas,
pero complementarias para observar y monitorizar nuestras aplicaciones.
Mientras que el primer tutorial se enfoca en CloudEvents y CDEvents para mostrar cómo los equipos de plataformas pueden mirar los distintos orígenes para calcular las métricas de DORA, el segundo tutorial se enfoca en el toolkit de Ketpn, que provee la métrica de Frecuencia de Despliegue por defecto al extender el programador de Kubernetes y recolectando información acerca de nuestras aplicaciones.

Los equipos de plataformas deberían evaluar herramientas como las presentadas aquí, no solo para calcular métricas, sino también para justificar sus inversiones en la plataforma.
Si las decisiones e iniciativas de plataforma mejoran la frecuencia de despliegue, el tiempo de respuesta a los cambios y al mismo tiempo reducen el tiempo de recuperación de fallos, estás construyendo la plataforma correcta.
Si notas que los equipos no están desplegando tan frecuentemente como esperabas, los cambios tardan más en llegar a los clientes, es posible que tengas que reevaluar tus decisiones.

## Limpieza

Si quieres deshacerte del Clúster de KinD creado para este tutorial, puedes ejecutar:

```shell
kind delete clusters dev
```
