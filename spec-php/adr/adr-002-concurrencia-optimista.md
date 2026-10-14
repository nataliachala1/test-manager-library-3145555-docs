# ADR-002 · Concurrencia optimista sobre los ejemplares

- **Estado:** aceptada
- **Contexto:** dos bibliotecarios pueden prestar el **último ejemplar** del mismo libro al mismo tiempo (RNF-2). Si ambos leen
  `available_copies = 1` y ambos escriben, el sistema presta un libro que no tiene.
- **Decisión:** control de concurrencia **optimista**. `book.version` es un entero obligatorio que inicia en 1. Al guardar, el adaptador
  ejecuta una actualización condicionada por `id` y la versión leída; si la versión cambió, la operación falla y el servidor responde
  **409 `BOOK_NOT_AVAILABLE`**. Una actualización exitosa incrementa la versión exactamente una vez.
  La **última barrera** es el `CHECK` `ck_book_available_range` (`0 <= available_copies <= total_copies`).
- **Alternativas descartadas:** bloqueo pesimista de la fila (`SELECT … FOR UPDATE`) en cada préstamo: serializa toda la biblioteca por
  un caso raro; contador en memoria: no sobrevive a más de un proceso.
- **Consecuencias:** si el `CHECK` salta, **algo escribió fuera del adaptador**: es un defecto, no un caso de negocio. Devolver dos veces
  la misma línea no suma dos veces porque `returned_at` ya está lleno (CA-07.2).
- **Límite conocido para HU-06:** `book.version` protege ejemplares por libro, pero no el máximo de tres líneas abiertas por socio
  (CA-06.4). Dos solicitudes concurrentes del mismo socio que tomen libros distintos pueden leer el mismo conteo y excederlo en
  conjunto. La especificación no define cómo serializar ese agregado. Elegir una estrategia (por ejemplo, serializar por socio u otra
  alternativa transaccional equivalente) requiere decisión técnica; hasta entonces, la garantía concurrente de CA-06.4 y la
  implementación completa de HU-06 quedan bloqueadas (DT-011). No se afirma que el testigo del libro resuelva este caso.
- **Devoluciones concurrentes:** la operación actualiza la línea solo cuando `returned_at IS NULL` y reintegra una unidad dentro de la
  misma transacción. Una petición confirma ambos cambios; las competidoras que ya no actualicen la línea responden 409
  `ITEM_ALREADY_RETURNED` y no cambian existencias. Esto concreta la protección de CA-07.1–CA-07.3.
- **Cómo se verifica:** prueba concurrente con dos peticiones simultáneas sobre un libro con un ejemplar: exactamente una recibe 201,
  la otra 409, y la versión queda incrementada una sola vez. Tras resolver DT-011, probar también el límite de tres líneas por socio.
  Dos devoluciones simultáneas de una línea deben producir una sola actualización y un solo reintegro; la otra responde 409.
