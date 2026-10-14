# Tareas — Manager Library (PHP/Laravel)

Cada tarea termina cuando su **evidencia** existe y se puede ejecutar (artículo XII). Estado inicial de todas: **pendiente**.

| # | Tarea | Depende de | Evidencia |
|---|---|---|---|
| T-01 | Esqueleto del proyecto y `GET /health` | — | `GET /health` responde `{"status":"ok"}`; el contenedor arranca con un comando |
| T-02 | Migración inicial: seis tablas, `book.version`, FK, índices únicos y las cinco categorías sembradas | T-01 | Las consultas de `data-model.md` §10 devuelven las seis tablas, columnas, tipos, nulabilidad, restricciones, relaciones e índices documentados; ninguna columna con `DEFAULT` |
| T-03 | Dominio: `Isbn`, `Quantity`, `DateRange` (objetos de valor) | T-01 | Pruebas unitarias: ISBN de 13 dígitos, cantidad > 0, fin ≥ inicio |
| T-04 | Dominio: `Book` (`Withdraw`, `Restock`, baja lógica) | T-03 | Pruebas: nunca < 0, nunca > total |
| T-05 | Dominio: `Loan` (`AddItem`, `ReturnItem`, estado calculado, 14 días, máx. 3) | T-04 | Pruebas con un `Clock` falso: `open`, `overdue`, `closed` |
| T-06 | Dominio: `Member` y `User` (normalización de nombre, rol válido) | T-03 | `"Ana "` queda `ana`; rol `admin` por API rechazado |
| T-07 | Puertos y casos de uso de **autenticación** (`LoginService`, `CreateLibrarianService`) con fakes | T-06 | HU-01 y HU-02 pasan contra fakes |
| T-08 | Puertos y casos de uso de **libros** (crear, buscar, editar, baja, portada) con fakes | T-04 | HU-03 y HU-04 pasan contra fakes; consultar categorías cubre la dependencia de lectura de HU-03 |
| T-09 | Caso de uso **prestar** (`LendBooksService`) con fakes | T-05, T-08, T-12 | HU-06 completa: límites, vencidos, sin ejemplares; revalidar límite por socio de forma atómica requiere resolución DT-011 |
| T-10 | Caso de uso **devolver** y **vencidos** | T-09 | HU-07 y HU-09; devolver dos veces → `ITEM_ALREADY_RETURNED`; registrar la concurrencia de devoluciones en ADR-002 |
| T-11 | Caso de uso **reporte de más prestados** y su puerto de lectura | T-09 | HU-10; el título viejo se conserva tras renombrar (ADR-004) |
| T-12 | Adaptador de persistencia: repositorios de libro, socio, usuario | T-02, T-08 | Pruebas de integración contra PostgreSQL real; incluye búsqueda y conteo de socios (HU-05) |
| T-13 | Adaptador de persistencia: repositorio de préstamos + testigo de versión | T-12, T-09 | Dos préstamos del último ejemplar: uno 201, otro 409 (RNF-2); consultas y estado calculado de HU-08; carrera por máximo de préstamos por socio queda bloqueada por DT-011 |
| T-14 | Adaptador de seguridad: hash de claves y token JWT | T-07 | La clave nunca aparece en logs ni respuestas (RNF-3) |
| T-15 | Adaptador de portadas (disco) | T-08 | Subir PNG de 2 MB ok; 3 MB → 422; sin portada → `cover_key` nulo |
| T-16 | Adaptador HTTP: rutas de [`api-contract.md`](api-contract.md), cuerpo de error único, autorización por rol | T-10, T-11, T-14, T-15 | Prueba de contrato recorre todas las rutas, solicitudes, respuestas, códigos de §2 y autenticación 401/403; cubre HU-01 a HU-10 |
| T-17 | Composición y arranque (admin inicial desde entorno) | T-16 | El motor inicia vacío de datos de demostración, la aplicación ejecuta migraciones y se puede iniciar sesión como admin; las cinco categorías fijas sí están sembradas |
| T-18 | Bajar a la base las invariantes **solo dominio**: `ck_book_total_positive`, `ck_user_role` | T-12 | Un `INSERT` por `psql` que las viole falla; `data-model.md` §4 se actualiza |
| T-19 | Pruebas de arquitectura (artículos I, II y III) | T-17 | Las reglas de dependencia de PHP pasan y fallan si se introduce una dependencia prohibida |
| T-20 | `infra`: `docker compose up` levanta motor, API y frontend | T-17 | Un comando; `infra` no crea tablas y la aplicación ejecuta migraciones antes de servir tráfico |
| T-21 | `tool`: sembrador de datos de demostración **por la API** (≥ 40 libros, 10 socios, 15 préstamos) | T-17 | Después de ejecutarlo, búsqueda y reporte tienen datos. Los tres vencidos no pueden generarse con el contrato actual (el cliente no fija `loaned_at`); conseguir vencidos requiere una estrategia aprobada que no añada endpoints ni cambie reglas: DT-008 bloquea esa parte de evidencia. Estos datos de demostración no son la carga sintética de RNF-1. |
| T-22 | `app`: login, catálogo, socios, consultar/listar préstamos, prestar, devolver, vencidos, reporte | T-16 | Recorrido manual de punta a punta en la feria; cubre HU-01 a HU-10 |
| T-23 | `page`: sitio público estático de presentación | — | Se abre sin la API; correcto en móvil y escritorio |

## Definición de terminado de la prueba

Quien llegue a T-17 con evidencia tiene el **núcleo** hecho. T-18 a T-23 son lo que separa un trabajo bueno de uno sobresaliente. Se evalúa
el desempeño con SDD, no la cantidad de tareas: una tarea con evidencia vale más que tres a medias.
