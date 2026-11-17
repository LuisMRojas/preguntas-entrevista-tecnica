# 3. IEnumerable vs IQueryable (LINQ y EF Core)

> **Pregunta típica:** "¿Cuál es la diferencia entre `IEnumerable<T>` e `IQueryable<T>`? ¿Dónde se ejecuta la consulta?"

---

## Respuesta corta (para decir en entrevista)

> Las dos representan una secuencia que se puede recorrer y las dos tienen **ejecución diferida**: no hacen nada hasta que las recorro (`foreach`, `ToList`, `Count`, `First`...). La diferencia está en **dónde y cómo se ejecutan los operadores LINQ**:
> - Con **`IEnumerable<T>`** (LINQ to Objects), los `Where`, `Select`, etc. reciben **delegados** (`Func<T, bool>`) y se ejecutan **en memoria**, en mi proceso .NET, sobre datos que ya están cargados.
> - Con **`IQueryable<T>`** (por ejemplo un `DbSet` de EF Core), los operadores reciben **árboles de expresión** (`Expression<Func<T, bool>>`). El provider los **traduce a SQL** y la consulta se ejecuta **en la base de datos**.
>
> El error típico es pasar a `IEnumerable` demasiado pronto, con un `ToList()` o `AsEnumerable()` antes del `Where`, o devolviendo `IEnumerable` desde un repositorio. Entonces se traen **todas las filas** de la tabla y se filtran en memoria. La regla es filtrar, ordenar, paginar y proyectar con `IQueryable`, y **materializar al final**.

---

## Explicación a fondo

### Las firmas lo dicen todo

```csharp
// System.Linq.Enumerable — para IEnumerable<T>
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate);

// System.Linq.Queryable — para IQueryable<T>
public static IQueryable<T> Where<T>(this IQueryable<T> source, Expression<Func<T, bool>> predicate);
```

- `Func<T, bool>` es **código compilado**: solo se puede ejecutar.
- `Expression<Func<T, bool>>` es una **estructura de datos** que describe el código: `p => p.Price > 100` se guarda como "propiedad `Price`, operador `>`, constante `100`". EF Core la **lee** y genera `WHERE [p].[Price] > 100`.

Como `IQueryable<T>` **hereda** de `IEnumerable<T>`, el compilador elige la sobrecarga según el **tipo estático** de la variable. Por eso basta un cambio de tipo para pasar, sin darte cuenta, de SQL a memoria.

### Comparación

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Namespace de los operadores | `System.Linq.Enumerable` | `System.Linq.Queryable` |
| Recibe | Delegados (`Func<>`) | Árboles de expresión (`Expression<Func<>>`) |
| Dónde se ejecuta | **En memoria** (proceso .NET) | **En el origen de datos** (SQL Server, PostgreSQL...) a través de un provider |
| Ejecución diferida | Sí | Sí |
| Puede usar cualquier método C# | Sí | Solo lo que el provider sepa traducir |
| Ideal para | Colecciones en memoria, resultados ya cargados | Consultas a la base de datos |

### Ejecución diferida en ambos

```csharp
var query = db.Products.Where(p => p.Price > 100); // aún NO se ejecutó nada

var count = query.Count();       // 1ª consulta: SELECT COUNT(*) ...
var list  = query.ToList();      // 2ª consulta: SELECT ... (vuelve a ir a la BD)
```

**Operadores que materializan** (disparan la ejecución): `ToList`, `ToArray`, `ToDictionary`, `First`, `Single`, `Any`, `Count`, `Sum`, `Max`, `foreach`, y sus versiones `Async` en EF Core (`ToListAsync`, `FirstOrDefaultAsync`...).

### ¿Qué pasa si EF Core no puede traducir algo?

```csharp
db.Products.Where(p => MyHelper.IsPremium(p)).ToList();
// EF Core 3+: InvalidOperationException: "The LINQ expression ... could not be translated"
```

Desde EF Core 3.0 **no hay evaluación silenciosa en cliente** en el `Where` (antes sí había, y causaba cargas completas de tablas sin avisar). Solo se permite evaluar en cliente en la **proyección final** (`Select`).

---

## Ejemplo de aplicación real

**Escenario:** un **catálogo de productos** con una tabla de ~2 millones de filas. Hay un endpoint `GET /products?search=&category=&page=` que devuelve 20 productos paginados.

### ❌ El bug: un repositorio que devuelve `IEnumerable`

```csharp
public class ProductRepository(AppDbContext db)
{
    public IEnumerable<Product> GetAll() => db.Products; // ⚠️ el tipo de retorno "oculta" el IQueryable
}

app.MapGet("/products", (string? search, int page, ProductRepository repo) =>
{
    var result = repo.GetAll()                                   // IEnumerable<Product>
        .Where(p => search == null || p.Name.Contains(search))   // Enumerable.Where → en memoria
        .OrderBy(p => p.Name)                                    // en memoria
        .Skip((page - 1) * 20).Take(20)                          // en memoria
        .ToList();
    return result;
});
```

SQL generado:

```sql
SELECT [p].[Id], [p].[Name], [p].[Description], [p].[Price], ... FROM [Products] AS [p]
-- ¡sin WHERE, sin ORDER BY, sin paginación!
```

**Síntomas en producción:** el endpoint tardaba ~8 segundos, el servidor usaba más de 1 GB de RAM por petición y la BD transfería todas las columnas de 2 millones de filas para mostrar 20. En desarrollo, con 50 productos de prueba, todo parecía funcionar bien.

### ✅ La corrección: componer sobre `IQueryable` y materializar al final

```csharp
app.MapGet("/products", async (
    string? search, int? categoryId, int page, AppDbContext db, CancellationToken ct) =>
{
    IQueryable<Product> query = db.Products.AsNoTracking(); // solo lectura: sin change tracking

    // Filtros condicionales: se van agregando al árbol de expresión
    if (!string.IsNullOrWhiteSpace(search))
        query = query.Where(p => p.Name.Contains(search));
    if (categoryId is not null)
        query = query.Where(p => p.CategoryId == categoryId);

    var total = await query.CountAsync(ct);                 // SELECT COUNT(*) ... WHERE ...

    var items = await query
        .OrderBy(p => p.Name)
        .Skip((page - 1) * 20)
        .Take(20)
        .Select(p => new ProductListItemDto(                // proyección: solo las columnas necesarias
            p.Id,
            p.Name,
            p.Price,
            p.Category.Name))                               // JOIN automático, sin cargar la entidad completa
        .ToListAsync(ct);                                   // ← aquí se ejecuta, una sola vez

    return Results.Ok(new PagedResult<ProductListItemDto>(items, total, page));
});
```

SQL generado (aprox., SQL Server):

```sql
SELECT [p].[Id], [p].[Name], [p].[Price], [c].[Name]
FROM [Products] AS [p]
INNER JOIN [Categories] AS [c] ON [p].[CategoryId] = [c].[Id]
WHERE [p].[Name] LIKE @search_contains ESCAPE N'\' AND [p].[CategoryId] = @categoryId
ORDER BY [p].[Name]
OFFSET @skip ROWS FETCH NEXT @take ROWS ONLY
```

**Resultado:** ~40 ms, 20 filas transferidas y 4 columnas en lugar de todas.

> Para ver el SQL: `query.ToQueryString()` o activar `LogTo(Console.WriteLine)` / el nivel `Microsoft.EntityFrameworkCore.Database.Command` en el logging.

### Cuándo SÍ conviene pasar a memoria a propósito: `AsEnumerable()`

Cuando necesito lógica que SQL no puede hacer, **primero filtro en la BD** y después cambio a memoria:

```csharp
var report = await db.Orders
    .Where(o => o.CreatedAt >= from && o.CreatedAt < to)    // ← en SQL
    .Select(o => new { o.Id, o.Total, o.ShippingAddress }) // ← en SQL, solo lo necesario
    .ToListAsync(ct);                                       // materializo un conjunto ya reducido

var byRegion = report                                       // ← en memoria (LINQ to Objects)
    .GroupBy(o => GeoHelper.ResolveRegion(o.ShippingAddress)) // método C# no traducible a SQL
    .Select(g => new { Region = g.Key, Total = g.Sum(x => x.Total) });
```

---

## Errores comunes / trampas

- **`ToList()` / `AsEnumerable()` antes de filtrar** → se trae la tabla completa.
- **Repositorios que devuelven `IEnumerable<T>`** sobre EF Core → toda composición posterior ocurre en memoria. Mejor: métodos específicos que ya filtran (`GetPagedAsync(filter)`) o devolver DTOs ya materializados.
- **Enumeración múltiple**: recorrer el mismo `IQueryable` dos veces = **dos consultas**. Recorrer el mismo `IEnumerable` "lazy" dos veces = recalcular todo dos veces. Materializa una vez si lo vas a usar varias veces.
- **N+1 queries**: recorrer pedidos y acceder a `order.Customer` con *lazy loading* → 1 consulta + N consultas. Solución: `Include`, o mejor una proyección con `Select`.
- **Traer entidades completas cuando solo se muestran 3 campos** → usa `Select` a un DTO.
- **`AsNoTracking()` olvidado en consultas de solo lectura** → más memoria y CPU por el change tracker.
- **Usar métodos propios en un `Where` sobre `IQueryable`** → excepción "could not be translated".
- **Devolver un `IQueryable` desde un endpoint después de que el `DbContext` fue liberado** → `ObjectDisposedException` al serializar.

---

## Preguntas de seguimiento para practicar

1. ¿Por qué el compilador elige `Enumerable.Where` o `Queryable.Where`? ¿Qué papel juega el **tipo estático** de la variable?
2. ¿Qué es un árbol de expresión y por qué EF Core lo necesita?
3. ¿Qué es la ejecución diferida? Da un ejemplo donde cause un bug (pista: recorrer dos veces, o el `DbContext` ya liberado).
4. ¿Qué diferencia hay entre `AsEnumerable()` y `ToList()`?
5. ¿Qué pasó con la *client evaluation* en EF Core 3.0 y por qué se cambió?
6. ¿Qué es el problema N+1 y cómo lo resuelves con EF Core?
7. ¿Cuándo usarías `AsNoTracking()`? ¿Y `AsSplitQuery()`?
