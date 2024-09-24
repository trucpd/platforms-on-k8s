# Capítulo 8: Facilitando que los equipos experimenten

—
_🌍 Disponible
en_: [English](README.md) | [中文 (Chinese)](README-zh.md) | [日本語 (Japanese)](README-ja.md) | [Español](README-es.md)
> **Nota:** Presentado por la fantástica comunidad
> de [ 🌟 contribuidores](https://github.com/salaboy/platforms-on-k8s/graphs/contributors) cloud-native!

---

En estos tutoriales, instalarás Knative Serving y Argo Rollouts en un clúster de Kubernetes para implementar
el lanzamiento tipo Canary, pruebas A/B y despliegues Blue/Green. Las estrategias de lanzamiento discutidas aquí tienen
como objetivo brindar a los equipos más control al liberar nuevas versiones de sus servicios. Al aplicar diferentes
técnicas para lanzar software, los equipos pueden experimentar y probar sus nuevas versiones en un entorno controlado,
sin dirigir todo el tráfico en vivo a una nueva versión en un solo momento.

- [Estrategias de Lanzamiento usando Knative Serving](knative/README-es.md)
- [Estrategias de Lanzamiento usando Argo Rollouts](argo-rollouts/README-es.md)

## Limpieza

Si deseas eliminar el clúster KinD creado para este tutorial, puedes ejecutar:

```shell
kind delete clusters dev
```

## Próximos Pasos

- Revisa el proyecto [Knative Functions](https://knative.dev/docs/functions/) si estás interesado en construir una
  plataforma de Function-as-a-Service, ya que esta iniciativa está trabajando en herramientas para facilitar la vida de
  los desarrolladores de funciones.

- Después de probar Argo Rollouts, el siguiente paso es crear un ejemplo de extremo a extremo que muestre el flujo de
  Argo CD a Argo Rollouts. Esto requiere crear un repositorio que contenga tus definiciones de Rollouts. Consulta la
  sección  [FAQ section on the Argo Projects](https://argo-rollouts.readthedocs.io/en/latest/FAQ/) para más detalles
  sobre su integración.

- Experimenta con ejemplos más complejos utilizando `AnalysisTemplates` y `AnalysisRuns`, ya que esta función ayuda a
  los equipos a implementar nuevas versiones con mayor confianza.

- Dado que ambos proyectos pueden funcionar con un Service Mesh como [Istio](https://istio.io/), familiarízate con lo
  que Istio puede hacer por ti.

## Resumen y Contribuir

¿Quieres mejorar este tutorial? Crea un issue, envíame un mensaje en [Twitter](https://twitter.com/salaboy)  o envía un
Pull Request.
