# Manager Library — Spec PHP/Laravel

Esta carpeta es la referencia objetivo para implementar **Manager Library** con PHP/Laravel y React.

## Reglas de esta versión

- El nombre del producto es **Manager Library**.
- El alcance es exactamente el definido en [`spec.md`](spec.md); no se agregan funcionalidades ni se reabren exclusiones.
- Las historias, criterios de aceptación, reglas de negocio y contrato HTTP conservan el producto definido en las especificaciones de referencia.
- Las decisiones específicas de PHP/Laravel viven aquí, sin trasladar detalles de Python o .NET al código objetivo.
- No se inicia código ejecutable hasta cerrar esta documentación.
- Las decisiones que no bloquean el alcance se registran en [`deuda-tecnica.md`](deuda-tecnica.md).

## Orden de lectura

1. [`constitution.md`](constitution.md): principios obligatorios.
2. [`spec.md`](spec.md): alcance, actores, reglas y fuente original de las diez HU.
3. [`hu/`](hu/): fichas trazables de [HU-01](hu/HU-01.md), [HU-02](hu/HU-02.md), [HU-03](hu/HU-03.md), [HU-04](hu/HU-04.md), [HU-05](hu/HU-05.md), [HU-06](hu/HU-06.md), [HU-07](hu/HU-07.md), [HU-08](hu/HU-08.md), [HU-09](hu/HU-09.md) y [HU-10](hu/HU-10.md).
4. [`architecture.md`](architecture.md) y [`arquitectura-panoramica.md`](arquitectura-panoramica.md): estructura y responsabilidades.
5. [`data-model.md`](data-model.md): tablas, relaciones e invariantes.
6. [`api-contract.md`](api-contract.md): contrato HTTP consumido por React.
7. [`plan.md`](plan.md): decisiones técnicas y fases de arranque.
8. [`tasks.md`](tasks.md): dependencias y evidencias de implementación.
9. [`adr/`](adr/): decisiones arquitectónicas.
10. [`deuda-tecnica.md`](deuda-tecnica.md): asuntos pendientes y bloqueos.

## Validación documental y autorización de desarrollo

La validación se hace en este orden: alcance y criterios de `spec.md`; cobertura de cada HU en `hu/`; consistencia de contratos, modelo, arquitectura y ADR; trazabilidad en `tasks.md`; verificación de enlaces y evidencias pendientes. Una HU queda documentalmente validada cuando sus criterios originales están presentes y sus referencias API, entidad, tarea, repositorio y prueba/evidencia están identificadas o marcadas explícitamente como pendientes.

No puede comenzar el código ejecutable hasta que las diez HU y los documentos relacionados hayan sido revisados y aprobados expresamente por el propietario. La entrega de documentación por sí sola no constituye esa aprobación.

## Fuente de verdad

Cuando exista una discrepancia, se identifica su fuente y se corrige solo si el requisito aprobado y la solución son inequívocos. Si requiere decisión funcional o arquitectónica, queda pendiente y bloquea la HU afectada. `spec.md` conserva los requisitos funcionales aprobados; `data-model.md` y `api-contract.md` son las referencias detalladas para sus respectivos ámbitos, subordinadas a esos requisitos. Las decisiones específicas de Laravel no alteran el contrato que consumen `api`, `app`, `infra`, `page` y `tool`.

La guía general del repositorio raíz todavía describe solo `spec-python/` y `spec-.net/`; actualizar esa guía está fuera del alcance autorizado de esta tarea. Se registra en [`deuda-tecnica.md`](deuda-tecnica.md).
