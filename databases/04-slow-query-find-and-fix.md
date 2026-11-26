# 4. How to find and fix a slow SQL query

> **Question:** "How would you find and fix a slow SQL query?"

---

## 🗣️ Answer (say it out loud, 1–2 min)

> First, I **measure** instead of guessing. To **find** slow queries, I use the **slow query log** in MySQL, or an APM tool, to see which queries are slow **and** which ones run most often. A query that takes 50 milliseconds but runs a million times a day can matter more than one that takes 5 seconds once a week.
>
> Then I run **`EXPLAIN`**, or **`EXPLAIN ANALYZE`** in MySQL 8, to see the execution plan. I look for **full table scans** (`type: ALL`), a high number of **rows examined**, and things like **`Using filesort`** or **`Using temporary`**.
>
> The most common fixes are:
> - adding the **right index**, often a **composite** one that matches the `WHERE` and the `ORDER BY`;
> - making sure the query **can use the index**, for example **not wrapping the indexed column in a function**;
> - **selecting only the columns I need** instead of `SELECT *`;
> - checking that **JOINs** are on indexed columns;
> - on the application side, fixing the **N+1 problem** in the ORM and **paginating** results.
>
> After the change, I **measure again** to confirm it actually improved, and I check that the new index doesn't hurt writes too much.

---

## Explicación (en español)

### El proceso, paso a paso

```
1. Encontrar   →  slow query log / APM / performance_schema
2. Reproducir  →  misma query, mismos parámetros, datos de volumen realista
3. Diagnosticar→  EXPLAIN / EXPLAIN ANALYZE
4. Corregir    →  índice, reescribir la query, cambiar el código de la app
5. Medir       →  ¿mejoró? ¿empeoró algo más (escrituras)?
```

### 1. Encontrarlas

```sql
-- Activar el slow query log (normalmente se configura en my.cnf o en el parameter group de RDS)
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;   -- registrar las queries de más de 1 segundo
```

```sql
-- Las queries que más tiempo total consumen (MySQL 8, esquema sys)
SELECT query, exec_count, total_latency, avg_latency, rows_examined_avg
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 10;
```

Herramientas: `pt-query-digest` (Percona) para resumir el slow log, **Performance Insights** en AWS RDS o un APM (New Relic, Datadog...).

### 2. Leer `EXPLAIN`

| Columna | Qué mirar |
|---|---|
| `type` | De peor a mejor: `ALL` (full scan) → `index` → `range` → `ref` → `eq_ref` → `const` |
| `key` | Qué índice usó (`NULL` = ninguno) |
| `rows` | Cuántas filas **estima** que va a examinar |
| `filtered` | Qué % de esas filas pasa el filtro (bajo = mucho trabajo desperdiciado) |
| `Extra` | 🚩 `Using filesort`, `Using temporary`; ✅ `Using index` (covering index) |

`EXPLAIN ANALYZE` (MySQL 8.0.18+) **ejecuta** la query y muestra los tiempos y las filas **reales**, no estimadas.

### 3. Causas típicas y su solución

| Problema | Ejemplo | Solución |
|---|---|---|
| Falta un índice | `WHERE customer_id = ?` sin índice | `CREATE INDEX` (compuesto si hay varios filtros) |
| **Función sobre la columna indexada** | `WHERE DATE(created_at) = '2025-11-01'` | Rango: `created_at >= '2025-11-01' AND created_at < '2025-11-02'` |
| `LIKE` con comodín al inicio | `WHERE name LIKE '%phone%'` | Índice `FULLTEXT` o un motor de búsqueda |
| Conversión implícita de tipos | `WHERE phone = 5551234` con `phone` VARCHAR | Comparar con el mismo tipo: `'5551234'` |
| `SELECT *` | Trae `JSON`/`TEXT` grandes que no se usan | Listar solo las columnas necesarias |
| JOIN sin índice | `ON o.customer_id = c.id` sin índice en `customer_id` | Indexar la columna del JOIN |
| `OFFSET` enorme | `LIMIT 20 OFFSET 500000` | **Keyset pagination**: `WHERE id < :lastId ORDER BY id DESC LIMIT 20` |
| `OR` entre columnas distintas | `WHERE a = 1 OR b = 2` | `UNION` de dos queries indexadas |
| **N+1 en el ORM** | 1 query de pedidos + 1 query por cada cliente | `Include` / proyección con `Select` |

---

## Ejemplo real (MySQL + .NET)

**Escenario:** la pantalla "Mis pedidos" de un e-commerce se volvió lenta (~4 s) cuando la tabla `orders` pasó de unos miles a millones de filas.

### Paso 1: encontrar

En el slow query log aparece esta query, que se ejecuta miles de veces por hora:

```sql
SELECT * FROM orders
WHERE customer_id = 1234
  AND YEAR(created_at) = 2025
ORDER BY created_at DESC
LIMIT 20;
```

### Paso 2: diagnosticar

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 1234 AND YEAR(created_at) = 2025 ORDER BY created_at DESC LIMIT 20;
```

| type | key | rows | Extra |
|---|---|---|---|
| `ALL` | `NULL` | 3,200,000 | `Using where; Using filesort` |

> Números **ilustrativos**.

Tres problemas:
1. No hay índice sobre `customer_id` → **full table scan**.
2. `YEAR(created_at)` envuelve la columna en una función → aunque hubiera índice en `created_at`, **no podría usarlo**.
3. `SELECT *` trae columnas grandes (`shipping_address`, `notes`) que la pantalla no muestra.

### Paso 3: corregir

```sql
CREATE INDEX ix_orders_customer_created ON orders (customer_id, created_at);
```

```sql
SELECT id, status, total, created_at
FROM orders
WHERE customer_id = 1234
  AND created_at >= '2025-01-01' AND created_at < '2026-01-01'   -- rango en vez de YEAR()
ORDER BY created_at DESC
LIMIT 20;
```

### Paso 4: medir de nuevo

| type | key | rows | Extra |
|---|---|---|---|
| `range` | `ix_orders_customer_created` | 20 | `Using index condition; Backward index scan` |

Sin full scan, sin filesort (el orden sale del índice) y menos datos transferidos: de segundos a milisegundos.

### El mismo problema en el ORM: N+1 (EF Core)

```csharp
// ❌ N+1: 1 query para los pedidos + 1 query POR CADA pedido para cargar el cliente (lazy loading)
var orders = await db.Orders.Where(o => o.Status == "Pending").ToListAsync(ct);
foreach (var o in orders)
    Console.WriteLine($"{o.Id} - {o.Customer.Name}");   // ← cada acceso dispara otra query

// ✅ 1 sola query con JOIN y solo las columnas necesarias
var orders = await db.Orders
    .Where(o => o.Status == "Pending")
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToListAsync(ct);
```

> 🔗 Relacionado: [IEnumerable vs IQueryable](../dotnet/03-ienumerable-vs-iqueryable.md). Filtrar en memoria en lugar de en SQL es otra causa muy común de "query lenta" que en realidad es un problema del código.
>
> 🔗 Y tu historia de S3 es un caso de este mismo tema: las **búsquedas por fechas antiguas eran lentas** porque la tabla de auditoría era enorme. A veces la solución no es un índice, sino **reducir el volumen** de la tabla.

---

## Useful vocabulary

| English | Español |
|---|---|
| execution plan / query plan | plan de ejecución |
| rows examined | filas examinadas |
| full table scan | recorrido completo de la tabla |
| to wrap a column in a function | envolver la columna en una función |
| bottleneck | cuello de botella |
| keyset / cursor pagination | paginación por clave |
| to measure, not guess | medir, no adivinar |

---

## Follow-up questions (practice in English)

1. What's the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?
2. Why does `WHERE YEAR(created_at) = 2025` not use an index on `created_at`?
3. What is the N+1 problem and how do you fix it in an ORM?
4. Why is `OFFSET` pagination slow on large tables? What's the alternative?
5. The query is fast in development but slow in production. Why could that be? (data volume, missing index in prod, different statistics, locks, server load)
6. A query was fast yesterday and is slow today, with no code changes. What would you check? (data growth, stale statistics → `ANALYZE TABLE`, locks, a changed execution plan)
