# ADR-004 · El reporte se calcula en el motor y agrupa por valores congelados

- **Estado:** aceptada
- **Contexto:** el reporte de más prestados (HU-10) cruza `loan` y `loan_item` sobre un rango. Si leyera el título o la categoría del catálogo
  vivo, renombrar un libro o una categoría cambiaría un reporte de un periodo ya cerrado.
- **Decisión:** (1) el reporte **no se persiste**: se calcula con una sola consulta agregada por un puerto de lectura. El rango
  inclusivo filtra `loan.loaned_at`. (2) Agrupa por `book_id, book_title, category_name` **tal como quedaron congelados en la línea**;
  `times_borrowed` cuenta las líneas de préstamo incluidas. (3) `loan_item.category_name` **no tiene FK** a propósito. (4) Si un
  libro cambió de categoría dentro del rango, el reporte muestra **una fila por combinación congelada**, por lo que pueden aparecer
  varias filas para el mismo `book_id`; no elige un ganador. (5) El orden es cantidad descendente, con título como desempate.
- **Alternativas descartadas:** unir con el catálogo vivo (el reporte deja de ser estable); tomar «el valor más reciente del rango»
  (un préstamo nuevo cambiaría lo ya leído); materializar una tabla de reporte (segunda fuente de verdad).
- **Consecuencias:** esta agrupación implementa CA-10.1 («una fila por libro y nombre de categoría congelado»); no equivale a una
  fila única por `book_id` cuando cambia la categoría durante el periodo. El reporte no incluye socios ni bibliotecarios (CA-10.3,
  privacidad). Una vez cerrada la consulta del periodo, renombrar libro/categoría no cambia sus etiquetas congeladas (CA-10.4).
- **Cómo se verifica:** prestar un libro, renombrarlo y cambiar su categoría, pedir reporte del mismo rango: título/categoría salen de
  la línea. Con dos categorías congeladas para ese libro dentro del rango, la agrupación mantiene una fila por cada combinación.
