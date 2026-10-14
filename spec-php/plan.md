# Plan técnico — Manager Library (PHP/Laravel)

Cómo se construye lo que dicen [`spec.md`](spec.md) y [`constitution.md`](constitution.md). El esquema **no** se describe aquí: vive en
[`data-model.md`](data-model.md). La forma del sistema, en [`architecture.md`](architecture.md).

## 0. Fase documental obligatoria y puerta de desarrollo

Antes de crear código ejecutable se completa la revisión de [HU-01](hu/HU-01.md) a [HU-10](hu/HU-10.md): criterios frente a `spec.md`, reglas y contratos, modelo y arquitectura, tareas/repositorios responsables, y pruebas/evidencias. Los enlaces deben resolver y los bloqueos funcionales o técnicos deben tener decisión registrada. Se documentan los resultados de revisión para que el propietario pueda aprobarlos. **El desarrollo solo queda autorizado después de la aprobación expresa del propietario**; esta fase documental y su entrega no la sustituyen.

Estado de esta revisión: las fichas están trazadas, pero hay bloqueos pendientes DT-006 (retención/históricos), DT-008 (datos vencidos de demostración), DT-011 (concurrencia del límite de socio) y DT-012 (código de error de autenticación). Por tanto, no se declara lista la documentación para iniciar la implementación de las HU afectadas ni completamente aprobada la documentación.

## 1. Decisiones técnicas

| # | Decisión | Elección (PHP/Laravel) | Por qué |
|---|---|---|---|
| D-01 | Lenguaje y runtime | PHP 8.3+ | Stack establecido para el reto |
| D-02 | Framework HTTP | Laravel 11+ | Contrato en [`api-contract.md`](api-contract.md) |
| D-03 | Persistencia | Eloquent + migraciones Laravel sobre PostgreSQL 16 | El esquema es de las migraciones (ADR-001) |
| D-04 | Concurrencia | Testigo de versión por libro + `CHECK` de rango ([ADR-002](adr/adr-002-concurrencia-optimista.md)) | Último ejemplar (RNF-2) |
| D-05 | Fechas | Instantes UTC (`timestamptz`); el reloj es un puerto (`Clock`) | Probar vencimientos sin esperar 14 días |
| D-06 | Reporte | Una consulta agregada por un puerto de lectura ([ADR-004](adr/adr-004-reporte-agregado-y-congelado.md)) | Estable y barato |
| D-07 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: viven dentro de la fila de su dueño, sin tabla | No tienen identidad |
| D-08 | Portadas | Binario en almacenamiento externo (disco en local, S3 en producción); la base guarda solo `cover_key` | Clave opaca, nunca ruta |
| D-09 | Contraseñas | Argon2id detrás del puerto `PasswordHasher`; el dominio nunca ve la clave en claro | Artículo IX |
| D-10 | Arranque | La aplicación crea el admin inicial desde `ADMIN_USERNAME` / `ADMIN_PASSWORD` | La base no puede producir el hash |
| D-11 | Autenticación | JWT HS256, 8 horas, clave en `JWT_SECRET` | Simple y sin estado |
| D-12 | Pruebas | PHPUnit/Pest, con fakes para los puertos y una base real solo en pruebas de integración | Pirámide de pruebas |

## 2. Configuración por entorno

| Variable | Obligatoria | Ejemplo |
|---|---|---|
| `DATABASE_URL` | sí | `postgresql://library:library@db:5432/library` |
| `JWT_SECRET` | sí | cadena larga aleatoria |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | sí | definidos en `.env`, nunca en el repo |
| `COVERS_DIR` | no | `./var/covers` |
| `CORS_ORIGIN` | sí | `http://localhost:5173` |

## 3. Estrategia de pruebas

| Nivel | Qué prueba | Contra qué |
|---|---|---|
| Dominio | Invariantes: máximo 3, vencimiento a 14 días, disponibles en rango | Nada: objetos puros |
| Aplicación | Casos de uso: prestar, devolver, reporte | Fakes de los puertos |
| Integración | Repositorios y consulta del reporte | PostgreSQL real |
| Contrato | Cuerpo de error único y códigos de estado | La API levantada |
| Arquitectura | Artículos I, II y III | Pruebas de arquitectura para PHP |
| Concurrencia | Dos préstamos del último ejemplar | PostgreSQL real |

## 4. Orden de construcción

Fase documental aprobada → esqueleto y salud → migración inicial con semilla → dominio → casos de uso con fakes → adaptadores Laravel → API → pruebas de contrato y concurrencia → infraestructura, herramienta y frontend. No se inicia una fase dependiente mientras sus decisiones bloqueantes no estén resueltas. Cada paso requiere la evidencia de [`tasks.md`](tasks.md); la puerta verificable de implementación es aprobación expresa del propietario y ausencia de bloqueos de HU.

Condiciones técnicas para Laravel: PHP 8.3+, Laravel 11+, PostgreSQL 16; migraciones propietarias de todo el DDL (ADR-001); tablas singulares en `library`, UUID explícitos y timestamps desactivados según [`data-model.md`](data-model.md); configuración Eloquent explícita según [`architecture.md`](architecture.md) y DT-007; JWT HS256 y secretos por entorno. `infra` provee motor vacío, la aplicación ejecuta migraciones y provisiona admin desde entorno. Las versiones y configuración concretas deben verificarse al crear el proyecto, sin relajar el contrato.

## 5. Ciclo documental de arranque

1. `infra` levanta PostgreSQL, la red, el volumen y el usuario de base de datos; no crea tablas.
2. `api` ejecuta las migraciones Laravel y siembra las cinco categorías fijas.
3. `api` provisiona el administrador inicial desde `ADMIN_USERNAME` y `ADMIN_PASSWORD` usando el puerto de hash.
4. `app` puede iniciar sesión contra `/api/v1`.
5. `tool` solo carga datos de demostración mediante la API.

En este documento, «base vacía» significa sin libros, socios ni préstamos de demostración; no significa que el sistema deba permanecer sin
esquema después de arrancar. La forma exacta de empaquetar estos pasos en Docker queda registrada como deuda técnica hasta implementar `infra`.
