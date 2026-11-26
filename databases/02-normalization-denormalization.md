# 2. Normalization, and when to denormalize

> **Question:** "What is database normalization, and when would you denormalize?"

---

## 🗣️ Answer (say it out loud, 1–2 min)

> **Normalization** is the process of organizing tables so that **each piece of data is stored in only one place**. You split the data into related tables connected by **foreign keys**. The goal is to avoid **redundancy** and the **anomalies** that come with it: if a customer's email is stored in every order, updating it means changing many rows, and if you miss one, the data becomes inconsistent.
>
> In practice, most transactional databases aim for **third normal form**, which roughly means: no repeating groups, and every column depends on **the key, the whole key, and nothing but the key**.
>
> **Denormalization** is intentionally adding redundancy to **make reads faster**. You avoid expensive joins or aggregations at the cost of more storage and more work to keep the copies in sync. I'd denormalize for **reporting and read-heavy queries**, for **pre-computed values** like counters or totals, or for a **read model** used by a dashboard. The rule I follow is: **normalize first**, and denormalize only when there is a **measured** performance problem, with a clear plan for keeping the data consistent.

---

## Explicación (en español)

### Las formas normales (versión práctica)

| Forma | Regla | Ejemplo de lo que la rompe |
|---|---|---|
| **1NF** | Valores **atómicos**, sin grupos repetidos | Una columna `phones = "555-1, 555-2"` |
| **2NF** | 1NF + cada columna depende de **toda** la clave (importa con claves compuestas) | En `order_items(order_id, product_id, product_name)`, `product_name` depende solo de `product_id` |
| **3NF** | 2NF + **sin dependencias transitivas** (una columna no clave no depende de otra columna no clave) | `orders(customer_id, customer_email)`: el email depende del cliente, no del pedido |

> Frase para recordar 3NF: *"every non-key column depends on the key, the whole key, and nothing but the key"*.

### Anomalías que evita normalizar

- **Update anomaly**: cambiar un dato obliga a actualizar muchas filas.
- **Insert anomaly**: no puedes guardar algo sin inventar otro dato (por ejemplo, un producto sin pedido).
- **Delete anomaly**: al borrar una fila pierdes información que no querías perder.

### Normalizar vs desnormalizar

| | Normalizado | Desnormalizado |
|---|---|---|
| Duplicación | Mínima | Intencional |
| Escrituras | Simples y consistentes | Más trabajo: hay que actualizar las copias |
| Lecturas | Pueden necesitar muchos JOINs | Rápidas, menos JOINs |
| Ideal para | **OLTP**: sistemas transaccionales | **OLAP**: reportes, dashboards, búsquedas |

### Formas comunes de desnormalizar

- **Columnas calculadas o contadores**: `posts.comment_count` en lugar de `COUNT(*)` en cada carga.
- **Tablas de reporte / resumen**: se llenan con un job (por ejemplo, ventas por día).
- **Copiar un dato para evitar un JOIN**: guardar `customer_name` en la tabla de facturas.
- **Read models** (CQRS): una tabla o vista pensada solo para una pantalla.
- **Vistas materializadas**: MySQL no las tiene nativas, se simulan con tablas + jobs o triggers.

> 💡 **No todo duplicado es desnormalización.** Guardar `unit_price` en `order_items` es **correcto**: es el precio **en el momento de la compra**, un dato histórico distinto del precio actual del producto. Un buen ejemplo para mostrar criterio en la entrevista.

---

## Ejemplo real (MySQL)

**Escenario:** un sistema de pedidos. El dashboard de gerencia muestra **ventas por día y por categoría** de los últimos 12 meses.

### Modelo normalizado (OLTP)

```sql
CREATE TABLE customers  (id BIGINT PRIMARY KEY, name VARCHAR(100), email VARCHAR(150) UNIQUE);
CREATE TABLE categories (id INT PRIMARY KEY, name VARCHAR(50));
CREATE TABLE products   (id BIGINT PRIMARY KEY, category_id INT NOT NULL, name VARCHAR(100), price DECIMAL(10,2),
                         FOREIGN KEY (category_id) REFERENCES categories(id));
CREATE TABLE orders     (id BIGINT PRIMARY KEY, customer_id BIGINT NOT NULL, created_at DATETIME NOT NULL,
                         FOREIGN KEY (customer_id) REFERENCES customers(id));
CREATE TABLE order_items(order_id BIGINT, product_id BIGINT, quantity INT,
                         unit_price DECIMAL(10,2),               -- precio histórico, NO es redundancia
                         PRIMARY KEY (order_id, product_id),
                         FOREIGN KEY (order_id) REFERENCES orders(id),
                         FOREIGN KEY (product_id) REFERENCES products(id));
```

La query del dashboard sobre este modelo:

```sql
SELECT DATE(o.created_at) AS day, c.name AS category, SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id
JOIN categories c   ON c.id = p.category_id
WHERE o.created_at >= NOW() - INTERVAL 12 MONTH
GROUP BY day, c.name;
```

Con millones de pedidos, esta query recorre y agrupa muchísimas filas **cada vez** que alguien abre el dashboard.

### Desnormalización: tabla de resumen

```sql
CREATE TABLE daily_sales_by_category (
  day          DATE        NOT NULL,
  category_id  INT         NOT NULL,
  category     VARCHAR(50) NOT NULL,     -- copiado a propósito para no hacer JOIN
  revenue      DECIMAL(14,2) NOT NULL,
  orders_count INT         NOT NULL,
  PRIMARY KEY (day, category_id)
);

-- Un job nocturno recalcula el día anterior (idempotente gracias al upsert)
INSERT INTO daily_sales_by_category (day, category_id, category, revenue, orders_count)
SELECT DATE(o.created_at), c.id, c.name, SUM(oi.quantity * oi.unit_price), COUNT(DISTINCT o.id)
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id
JOIN categories c   ON c.id = p.category_id
WHERE o.created_at >= CURDATE() - INTERVAL 1 DAY AND o.created_at < CURDATE()
GROUP BY DATE(o.created_at), c.id, c.name
ON DUPLICATE KEY UPDATE revenue = VALUES(revenue), orders_count = VALUES(orders_count), category = VALUES(category);
-- Nota: VALUES() en ON DUPLICATE KEY UPDATE está deprecado desde MySQL 8.0.20 (todavía funciona);
-- la alternativa nueva es usar un alias de fila: INSERT ... AS new ... UPDATE revenue = new.revenue
```

El dashboard ahora lee ~365 × N categorías filas ya calculadas:

```sql
SELECT day, category, revenue FROM daily_sales_by_category WHERE day >= CURDATE() - INTERVAL 12 MONTH;
```

**El trade-off que tienes que mencionar:** el dashboard muestra datos **hasta ayer**, no en tiempo real, y si se renombra una categoría hay que actualizar la copia. Para un reporte de gerencia eso es aceptable; para el carrito de compras, no.

> 🔗 Conexión con tu historia de S3: separar los logs recientes (MySQL) de los antiguos (S3) es otra forma de **elegir el almacenamiento según cómo se lee el dato**. Es el mismo tipo de razonamiento.

---

## Useful vocabulary

| English | Español |
|---|---|
| redundancy | redundancia, duplicación |
| update / insert / delete anomaly | anomalía de actualización / inserción / borrado |
| foreign key | clave foránea |
| read-heavy / write-heavy | con muchas lecturas / muchas escrituras |
| summary table / read model | tabla de resumen / modelo de lectura |
| to keep in sync | mantener sincronizado |
| eventual consistency | consistencia eventual |

---

## Follow-up questions (practice in English)

1. Explain 1NF, 2NF and 3NF with an example.
2. What's the difference between OLTP and OLAP databases?
3. If you denormalize, how do you keep the duplicated data consistent? (jobs, triggers, application events)
4. Is storing `unit_price` in `order_items` a violation of normalization? Why not?
5. What is a star schema? (fact table + dimension tables, used in data warehouses)
6. When would you choose a NoSQL document database instead of a normalized relational model?
