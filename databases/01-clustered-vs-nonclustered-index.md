# 1. Clustered vs non-clustered index (and when to add one)

> **Question:** "What is the difference between a clustered and a non-clustered index? When would you add an index?"

---

## 🗣️ Answer (say it out loud, 1–2 min)

> An **index** is a data structure, usually a **B-tree**, that lets the database find rows without scanning the whole table.
>
> A **clustered index** defines the **physical order** in which the rows are stored. The leaf level of the index **is the table itself**, so there can be **only one per table**. In MySQL with InnoDB, the clustered index is always the **primary key**.
>
> A **non-clustered index**, which MySQL calls a **secondary index**, is a **separate structure** that stores the indexed columns plus a **reference to the row**. In InnoDB that reference is the primary key value, so a lookup through a secondary index usually means **two steps**: find the primary key in the secondary index, then fetch the row from the clustered index. You can have many non-clustered indexes per table.
>
> I add an index when a query that runs often **filters, joins or sorts** by a column and `EXPLAIN` shows a **full table scan**, especially on large tables. But indexes aren't free: they **speed up reads** and **slow down writes**, because every `INSERT`, `UPDATE` and `DELETE` also has to update each index, and they use disk and memory. So I index based on **real query patterns**, not on every column.

---

## Explicación (en español)

### Clustered vs non-clustered

| | Clustered | Non-clustered (secondary) |
|---|---|---|
| Qué es | Los datos de la tabla **ordenados por la clave** | Estructura **aparte** con la clave + referencia a la fila |
| Cuántos por tabla | **Solo uno** | Muchos |
| En MySQL InnoDB | Siempre la **primary key** (si no hay, el primer `UNIQUE NOT NULL`; si tampoco, una clave oculta) | Todos los demás índices |
| Qué guarda la hoja del B-tree | **La fila completa** | La columna indexada + **el valor de la PK** |
| Búsqueda | 1 paso | 2 pasos (índice → PK → fila), salvo que sea un **covering index** |

**Analogía:** el clustered es un **diccionario**, donde las palabras ya están ordenadas y la definición está ahí mismo. El non-clustered es el **índice al final de un libro**: te dice "ver página 142" y tienes que ir a esa página.

> ⚠️ Matiz que suma puntos: la "idea clave" dice "punteros". En **SQL Server** el índice non-clustered apunta a la clave del clustered (o a un RID físico si la tabla no tiene clustered index). En **InnoDB** guarda el **valor de la PK**, no una dirección física. Por eso en MySQL una **PK grande** (por ejemplo un UUID en texto) hace más pesados **todos** los índices secundarios.

### Conceptos que suelen salir

- **Composite index** (varias columnas): el orden importa por la regla del **leftmost prefix**. Un índice `(customer_id, created_at)` sirve para `WHERE customer_id = ?` y para `WHERE customer_id = ? AND created_at > ?`, pero **no** para `WHERE created_at > ?` solo.
- **Covering index**: si el índice contiene **todas** las columnas que pide la query, no hace falta ir a la tabla. `EXPLAIN` muestra `Using index`.
- **Selectividad**: un índice sobre una columna con pocos valores distintos (por ejemplo `is_active` con 0/1) casi nunca ayuda.
- **PK autoincremental vs UUID aleatorio**: con InnoDB, los UUID aleatorios insertan en posiciones al azar del clustered index, lo que fragmenta las páginas y hace más lentos los inserts. Si se necesita UUID, mejor uno ordenado por tiempo (UUIDv7) o guardado como `BINARY(16)`.

### ¿Cuándo agregar un índice?

✅ Sí:
- Columnas en `WHERE`, `JOIN ... ON`, `ORDER BY` o `GROUP BY` de **queries frecuentes**.
- Foreign keys (InnoDB las indexa automáticamente).
- Cuando `EXPLAIN` muestra `type: ALL` (full scan) sobre una tabla grande.

❌ Cuidado:
- Tablas con **muchísimas escrituras** y pocas lecturas (por ejemplo, un log de auditoría): cada índice extra encarece cada insert.
- Columnas de baja selectividad.
- Índices duplicados o redundantes: `(a)` sobra si ya existe `(a, b)`.

---

## Ejemplo real (MySQL)

**Escenario:** la tabla de **logs de auditoría** de tu historia de S3. Se consulta así: "los movimientos del usuario X entre dos fechas".

```sql
CREATE TABLE audit_log (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- clustered index
  user_id     BIGINT UNSIGNED NOT NULL,
  action      VARCHAR(50)     NOT NULL,
  payload     JSON,
  created_at  DATETIME(3)     NOT NULL
) ENGINE=InnoDB;

-- La query más frecuente
SELECT id, action, created_at
FROM audit_log
WHERE user_id = 1234
  AND created_at >= '2025-10-01'
  AND created_at <  '2025-10-31'   -- rango semiabierto: BETWEEN con '2025-10-30' dejaría fuera casi todo ese día
ORDER BY created_at DESC;
```

> Los números de `EXPLAIN` son **ilustrativos**, no del proyecto real.

**Antes:** `EXPLAIN` → `type: ALL`, `rows: 48,000,000`: recorre toda la tabla.

```sql
-- Índice compuesto: primero la igualdad (user_id), después el rango (created_at)
CREATE INDEX ix_audit_user_created ON audit_log (user_id, created_at);
```

**Después:** `EXPLAIN` → `type: range`, `key: ix_audit_user_created`, `rows: ~350`. Además, el `ORDER BY created_at` sale del mismo índice, sin `Using filesort`.

**El trade-off:** es una tabla donde se escribe **cada movimiento**, así que no le agregas un índice por cada filtro posible. Solo los que sirven a las consultas reales.

> El orden `(user_id, created_at)` y no `(created_at, user_id)` es intencional: **igualdad primero, rango después**. Con el rango primero, MySQL no podría usar bien la segunda columna.

---

## Useful vocabulary

| English | Español |
|---|---|
| full table scan | recorrer toda la tabla |
| lookup | búsqueda |
| leaf node / leaf level | nodo hoja / nivel hoja |
| covering index | índice que cubre la consulta |
| selectivity / cardinality | selectividad / cardinalidad |
| write overhead | costo extra en las escrituras |
| trade-off | compromiso, costo-beneficio |

---

## Follow-up questions (practice in English)

1. Why can a table have only one clustered index?
2. What is a covering index? How do you know a query is using one?
3. In a composite index `(a, b, c)`, which queries can use it? (*leftmost prefix rule*)
4. Why can a random UUID primary key hurt performance in InnoDB?
5. Can an index make a query **slower**? When? (writes, low selectivity, the optimizer picking a bad plan)
6. How do you find indexes that are never used? (`sys.schema_unused_indexes` in MySQL)
