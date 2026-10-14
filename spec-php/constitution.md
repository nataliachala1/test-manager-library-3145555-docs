# Constitución — Manager Library (PHP/Laravel)

Principios innegociables del proyecto. No son estilo ni preferencia: cada artículo trae **cómo se verifica**, porque un principio que
nadie puede comprobar es una intención. Un cambio que viole un artículo se rechaza sin discutir el resto.

---

## Artículo I — El dominio no depende de nada

`app/Domain/` no importa infraestructura ni frameworks: solo PHP estándar. El dominio no depende de Laravel, Eloquent, controladores HTTP, validadores ni servicios externos.

**Por qué:** es lo único que permite probar las invariantes sin base de datos, sin servidor y sin contenedor, y cambiar de ORM sin tocar reglas.
**Verificación:** una prueba de arquitectura impide que `app/Domain/` dependa de `app/Infrastructure/`, `app/Http/` o clases de Laravel.

## Artículo II — El núcleo declara sus puertos; el mundo los implementa

La capa de aplicación solo conoce el dominio. Todo lo que necesita del exterior (repositorios, reloj, hash de claves, almacenamiento de
portadas, generación de tokens) lo expresa como un **puerto que ella misma define**. Ningún tipo de framework o de ORM aparece en la firma de un puerto.

**Verificación:** una prueba de arquitectura impide que `app/Application/` dependa de `app/Infrastructure/` o `app/Http/`. Las pruebas de aplicación usan fakes.

## Artículo III — Un único punto de composición

La composición se concentra en `app/Bootstrap/Composition/ServiceBindings.php`, donde los puertos se vinculan con sus adaptadores. Fuera de ese punto no se instancia infraestructura en los casos de uso.
**Verificación:** una prueba de arquitectura detecta dependencias directas de infraestructura desde dominio o aplicación.

## Artículo IV — SOLID no es decorativo

| Principio | Aplicación concreta | Verificación |
|---|---|---|
| **SRP** | Un servicio de aplicación = un caso de uso. `LendBooksService` presta y no hace nada más | Un servicio con dos razones para cambiar se parte |
| **OCP** | Cambiar el almacenamiento de portadas (disco → S3) no toca dominio ni aplicación | El adaptador nuevo solo añade una línea en la composición |
| **LSP** | Los fakes de test sustituyen al adaptador real sin cambiar expectativas | El mismo test pasa contra el fake y el real |
| **ISP** | Puertos pequeños: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | Ningún implementador deja métodos sin usar |
| **DIP** | El puerto lo define quien lo consume | Los puertos viven en el paquete de aplicación |

## Artículo V — El esquema es propiedad de las migraciones

El repositorio `infra` levanta un motor **vacío**. Todo el DDL (tablas, índices, restricciones, extensiones y semilla) vive en las migraciones
de Laravel. Ver [ADR-001](adr/adr-001-propiedad-del-esquema.md).
**Verificación:** `infra` no contiene DDL de aplicación; las tablas se crean mediante `php artisan migrate` ejecutado por la aplicación.

## Artículo VI — Las invariantes que la base puede imponer, la base las impone

Una regla que solo vive en código protege a la aplicación, no a los datos. Toda invariante expresable en el motor (`CHECK`, `UNIQUE`, FK)
se declara ahí. Lo que no está aplicado se marca **pendiente** en [`data-model.md`](data-model.md); no se promete.
**Verificación:** las consultas de `data-model.md` §10 coinciden con lo que declara el documento.

## Artículo VII — Lo derivado se calcula, no se almacena

El estado de un préstamo (`open`, `overdue`, `closed`) y los días de retraso se calculan. No hay columna para ellos.
**Verificación:** `data-model.md` no lista ninguna columna `status` ni `days_late`.

## Artículo VIII — Los valores congelados no siguen al catálogo

La línea de préstamo guarda título, ISBN y categoría del instante del préstamo. Ningún reporte los lee del catálogo vivo.
**Verificación:** una prueba que renombra un libro después de prestarlo y comprueba que el reporte conserva el título viejo ([ADR-004](adr/adr-004-reporte-agregado-y-congelado.md)).

## Artículo IX — Ningún secreto entra al repositorio ni a un log

Ni contraseñas, ni hashes precalculados, ni la clave del token. Todo secreto viene del entorno. El `.env` real está en `.gitignore`; solo se versiona `.env.example`.
**Verificación:** `gitleaks` o un `grep` de patrones en CI; ninguna respuesta HTTP incluye `password` ni `password_hash`.

## Artículo X — La base aplica el modelo documentado

`data-model.md` define el esquema esperado; las migraciones son su implementación ejecutable. Si una base desplegada difiere de `data-model.md`, esa diferencia es un defecto de migración o de despliegue que se investiga y corrige en el esquema, sin alterar el requisito documentado para ajustarlo a una desviación accidental. Si un requisito aprobado contradice el modelo, la inconsistencia se resuelve en la documentación y se registra la decisión antes de implementar; no se presume que la base o el documento puedan cambiar unilateralmente el comportamiento funcional.

## Artículo XI — Idioma

Código, columnas y rutas en inglés. Documentación y mensajes al usuario en español.

## Artículo XII — Nada se da por hecho sin evidencia

Una tarea de [`tasks.md`](tasks.md) está terminada cuando su **evidencia** existe y se puede ejecutar, no cuando «ya está programada».
