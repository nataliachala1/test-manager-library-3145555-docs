# Contrato de la API — Manager Library

El contrato HTTP es **independiente del lenguaje** y esta carpeta es la referencia para la implementación en React + Laravel. Las
versiones Python y .NET conservan el mismo contrato de negocio. Qué hace cada operación está en [`spec.md`](spec.md); qué se guarda,
en [`data-model.md`](data-model.md).

## 1. Convenciones

| Tema | Regla |
|---|---|
| Base | `/api/v1`; las rutas siguientes son relativas a esta base. `GET /health` se expone como `GET /api/v1/health` |
| Formato | JSON, UTF-8. Nombres de campo en `snake_case` |
| Identificadores | UUID v4 como texto |
| Fechas | ISO 8601 en UTC, con `Z` (`2026-10-14T14:15:00Z`) |
| Dinero | No hay. El sistema no maneja importes |
| Autenticación | `Authorization: Bearer <token>` en todo salvo `POST /auth/login` y `GET /health` |
| Paginación | `?page=1&size=20`. `page` empieza en 1; `size` máximo 50. La respuesta de ejemplo es `{ "items": [...], "page": 1, "size": 20, "total": 134 }`. El valor por defecto no está especificado. |
| Idioma | Mensajes de error en español; campos y códigos en inglés |

### Cuerpo de error (único para todos los errores)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` puede ir vacío. **Nunca** incluye trazas, SQL, rutas del servidor ni la contraseña. Credenciales inválidas usan `INVALID_CREDENTIALS` con 401. El contrato exige 401 para token ausente, inválido o vencido pero no define un código de error para esos casos; queda pendiente en DT-012. No se distingue públicamente la causa sensible.

### Códigos de estado

| Estado | Cuándo |
|---|---|
| 200 / 201 / 204 | Éxito / creado / sin contenido |
| 401 | Sin token, token inválido o vencido; credenciales inválidas solo en `POST /auth/login` |
| 403 | Rol insuficiente |
| 404 | No existe (o está dado de baja) |
| 409 | Conflicto de estado: duplicado, sin ejemplares, ya devuelto, socio con vencidos, libro con préstamos abiertos |
| 422 | Validación: campo inválido, regla de entrada rota, rango de fechas invertido |

## 2. Códigos de error de negocio

| `code` | Estado | Origen |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Cualquier recurso |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Cualquier campo inválido |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |

## 3. Endpoints

Las rutas se muestran sin el prefijo común `/api/v1`. Todo endpoint protegido valida el token y después el rol; la matriz de roles de cada ruta prevalece. `401` significa que la autenticación no es válida y `403` que el usuario autenticado no tiene rol suficiente. Ningún `403` revela si el recurso existe.

### 3.1 Autenticación y salud

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/auth/login` | público | Devuelve token |
| GET | `/health` | público | `{ "status": "ok" }` |

`POST /auth/login`

```json
// petición
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Usuarios

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/users` | admin | Crea un bibliotecario |

```json
// petición  (role solo admite "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201  (sin password ni password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

### 3.3 Categorías

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/categories` | autenticado | Las cinco categorías, orden por nombre |

Respuesta: lista de `{ "id", "name" }`; son las cinco categorías fijas de [`data-model.md`](data-model.md) §9.

### 3.4 Libros

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/books` | autenticado | Búsqueda: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | autenticado | Detalle |
| POST | `/books` | admin | Crea |
| PUT | `/books/{id}` | admin | Reemplaza título, autor, categoría y total; el ISBN no se modifica |
| DELETE | `/books/{id}` | admin | Baja lógica |
| GET | `/books/{id}/cover` | autenticado | Devuelve la portada binaria o 404 |
| PUT | `/books/{id}/cover` | admin | Sube portada (`multipart/form-data`, campo `file`) |
| DELETE | `/books/{id}/cover` | admin | Quita la portada |

Libro (respuesta):

```json
{
  "id": "…", "title": "Cien años de soledad", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novela" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`GET /books` responde con la envoltura paginada; cada elemento usa la representación anterior. `GET /books/{id}` responde con un libro o 404. `GET /books/{id}/cover` responde el binario con `Content-Type` `image/jpeg` o `image/png`, no JSON en éxito.

`POST /books` (petición): `title`, `author`, `isbn`, `category_id`, `total_copies`. Los disponibles los calcula el servidor.
`cover_key` es una clave opaca: para pintar la portada se usa `GET /books/{id}/cover`, que devuelve el binario o 404.

`PUT /books/{id}` (petición): `title`, `author`, `category_id`, `total_copies`. El servidor conserva el `isbn`, calcula el nuevo
`available_copies` aplicando la diferencia de copias y rechaza la operación si dejaría disponibles por debajo de cero. Responde `200`
con el libro actualizado.

`GET /books/{id}/cover` requiere autenticación y responde con el binario y su `Content-Type` (`image/jpeg` o `image/png`). Una portada
ausente, un libro inexistente o un libro dado de baja responden `404` con el cuerpo de error único.

### 3.5 Socios

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/members` | admin, librarian | Lista y busca: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detalle con `active_loans` (conteo sin devolver) |
| POST | `/members` | admin, librarian | Crea: `full_name`, `document_number` |

`GET /members` admite `q`, `page` y `size`; `q` busca por nombre o documento. Responde con envoltura paginada. `GET /members/{id}` incluye `active_loans`, el conteo de líneas sin devolver. `POST /members` recibe `full_name` y `document_number` y responde 201 con el socio creado. Los atributos son los definidos en [`data-model.md`](data-model.md); no existe operación de eliminación.

### 3.6 Préstamos

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Registra un préstamo |
| GET | `/loans` | admin, librarian | Lista: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Detalle con líneas y estado |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Devuelve una línea |
| GET | `/loans/overdue` | admin, librarian | Líneas vencidas; admite `page` y `size` |

En `GET /loans`, `from` y `to` son fechas `YYYY-MM-DD` inclusivas y filtran `loaned_at`; `status` admite `open`, `overdue` o `closed`, calculado al consultar; `member_id` es UUID. La respuesta es paginada y ordenada por `loaned_at` descendente (CA-08.2). `GET /loans/{id}` responde el préstamo con `items` y `status`, o 404.

`POST /loans`

```json
// petición  (1 a 3 elementos, sin libros repetidos)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "Cien años de soledad", "isbn": "9780307474728",
      "category_name": "Novela", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

`member_id` y cada `book_id` son UUID. Se admite de 1 a 3 elementos, sin `book_id` repetido. El servidor toma el instante UTC de `loaned_at` y calcula cada vencimiento a 14 días. La creación, el descuento de existencias y la creación de todas las líneas son atómicos: si falla una línea, no se registra ninguna. La disponibilidad o un socio con vencidos produce 409; exceder el máximo de tres líneas sin devolver produce 422. La carrera entre préstamos simultáneos de un socio está pendiente de decisión; ver ADR-002 y DT-011.

`status` es calculado: `open` (hay líneas sin devolver y ninguna vencida), `overdue` (alguna sin devolver y vencida), `closed` (todas devueltas).

`POST /loans/{loan_id}/items/{item_id}/return` → 200 con la línea (`returned_at` lleno). Marcar la devolución y reintegrar una unidad son atómicos. Si dos devoluciones compiten por la misma línea, solo una surte efecto; la otra responde 409 `ITEM_ALREADY_RETURNED` sin incrementar de nuevo las existencias.

`GET /loans/overdue` → respuesta paginada con `items`: `{ "loan_id", "item_id", "book_title", "member_name", "due_at", "days_late" }`.
Aquí sí se muestra el nombre del socio, porque es un listado operativo; **el reporte del §3.7 no**.

### 3.7 Reporte (HU-10)

| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | `from` y `to` obligatorios (fecha UTC, ambos inclusive) |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "Cien años de soledad", "category_name": "Novela", "times_borrowed": 12 }
  ]
}
```

`from` y `to` son fechas inclusivas `YYYY-MM-DD` aplicadas a `loan.loaned_at`. Si `from` es posterior a `to`, responde 422 `INVALID_DATE_RANGE`; un rango válido sin préstamos responde `rows: []`. Cada fila agrupa `book_id`, `book_title` y `category_name` congelados en la línea; si un libro cambió de categoría dentro del rango, pueden aparecer dos filas para ese libro. `times_borrowed` cuenta las líneas prestadas en el rango. Orden: cantidad descendente y, como desempate, título ascendente. El reporte no expone socios ni bibliotecarios. Estas reglas corresponden a CA-10.1–CA-10.5 y ADR-004.

## 4. Seguridad

| Aspecto | Regla |
|---|---|
| Token | JWT firmado, 8 horas, con `sub` y `role`. Sin datos personales dentro |
| Contraseñas | Solo se guarda el hash; el hash nunca sale en ninguna respuesta |
| Autorización | Se evalúa en cada endpoint por rol; un 403 no revela si el recurso existe |
| CORS | Solo el origen del frontend configurado por entorno |
| Subida de portada | Solo `image/jpeg` y `image/png`, máx. 2 MB; el tipo se valida por contenido, no por extensión |
