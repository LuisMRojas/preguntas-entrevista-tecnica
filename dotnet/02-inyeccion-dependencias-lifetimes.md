# 2. Inyección de dependencias en .NET: Singleton, Scoped y Transient

> **Pregunta típica:** "¿Cuáles son los tiempos de vida de un servicio en .NET? ¿Cuándo usarías cada uno? ¿Qué problema tiene inyectar un Scoped en un Singleton?"

> 🔗 **Relacionado:** [Angular: inyección de dependencias y alcance de los providers](../angular/04-inyeccion-dependencias-providers.md). La idea es la misma: **dónde registras el servicio define cuántas instancias hay y cuánto viven.**

---

## Respuesta corta (para decir en entrevista)

> .NET trae un contenedor de DI integrado (`IServiceCollection` / `IServiceProvider`). Al registrar un servicio elijo su **tiempo de vida**:
> - **Singleton**: **una sola instancia** para toda la aplicación, creada la primera vez que se pide. Sirve para servicios sin estado o con estado compartido y **thread-safe**: caché en memoria, configuración, `HttpClient` gestionado.
> - **Scoped**: **una instancia por scope**, que en ASP.NET Core equivale a **una por petición HTTP**. Todos los que la piden dentro de la misma request reciben la misma. Es el caso típico del `DbContext` y de los servicios de negocio que lo usan.
> - **Transient**: **una instancia nueva cada vez** que se resuelve. Sirve para servicios livianos y sin estado.
>
> La regla clave es que un servicio **no debe depender de otro con un tiempo de vida más corto**. Si inyecto un Scoped (como un `DbContext`) en un Singleton, queda "capturado" para siempre: es la **captive dependency**. Termina compartido entre peticiones y entre hilos, lo que causa bugs de concurrencia y datos viejos. Si un Singleton necesita algo Scoped, creo un scope con `IServiceScopeFactory`.

---

## Explicación a fondo

### Registro

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddTransient<IPriceCalculator, PriceCalculator>();

builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlServer(cs)); // Scoped por defecto
builder.Services.AddHttpClient<ShippingClient>();                     // cliente tipado con IHttpClientFactory
```

### Inyección (por constructor, aquí con *primary constructor* de C# 12+)

```csharp
public class OrderService(AppDbContext db, IPriceCalculator calculator, IClock clock) : IOrderService
{
    public async Task<Order> CreateAsync(CreateOrderDto dto, CancellationToken ct) { /* ... */ }
}
```

En Minimal APIs también se pueden pedir como parámetros del endpoint:

```csharp
app.MapPost("/orders", (CreateOrderDto dto, IOrderService orders, CancellationToken ct) => orders.CreateAsync(dto, ct));
```

### Comparación de tiempos de vida

| | Singleton | Scoped | Transient |
|---|---|---|---|
| Instancias | 1 en toda la app | 1 por scope (= por request HTTP) | 1 **por cada resolución** |
| Se destruye (`Dispose`) | Al apagar la app | Al terminar la request | Al terminar el scope que lo creó |
| Debe ser thread-safe | **Sí** (lo usan muchas requests a la vez) | No normalmente | No |
| Ejemplos | `IMemoryCache`, configuración, `IClock`, clientes de colas | `DbContext`, unit of work, usuario actual, servicios de negocio | Validadores, mappers, calculadoras sin estado |

### Qué puede depender de qué

| El servicio... | puede depender de Singleton | de Scoped | de Transient |
|---|---|---|---|
| **Singleton** | ✅ | ❌ captive dependency | ⚠️ el transient queda vivo como singleton |
| **Scoped** | ✅ | ✅ | ✅ |
| **Transient** | ✅ | ✅ | ✅ |

> En el entorno **Development**, ASP.NET Core activa `ValidateScopes` y lanza una excepción del tipo *"Cannot consume scoped service 'AppDbContext' from singleton..."*. En producción esa validación está apagada por defecto, así que el bug pasa silencioso. Se puede forzar:
>
> ```csharp
> builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });
> ```

---

## Conexión con Angular

| Angular | .NET | Comentario |
|---|---|---|
| `@Injectable({ providedIn: 'root' })` | `AddSingleton` | Una instancia para toda la app. |
| `providers` en una **ruta** | `AddScoped` (más o menos) | Una instancia por "contexto de trabajo": en Angular la sección de la app, en .NET la petición HTTP. |
| `providers` en un **componente** | `AddScoped` dentro de un scope propio | Una instancia por componente, compartida con sus hijos y destruida con él. Es lo más parecido a "crear un scope". |
| No hay equivalente directo | `AddTransient` | En Angular un servicio provisto en un componente se **comparte** con los hijos; Transient en cambio da una instancia nueva **a cada uno** que la pide. |
| `InjectionToken` + `useValue` | `Configure<T>()` / `IOptions<T>` o `AddSingleton(instancia)` | Inyectar configuración. |
| `{ provide: X, useClass: Y }` | `AddScoped<IX, Y>()` | Registrar una implementación para una abstracción. |
| `{ provide: X, useFactory: ... }` | `AddScoped<IX>(sp => new Y(...))` | Fábrica. |
| `inject(X, { optional: true })` | `sp.GetService<X>()` (devuelve `null`) | `GetRequiredService` lanza excepción si no existe. |
| `NullInjectorError: No provider for X` | `InvalidOperationException: Unable to resolve service for type X` | Mismo error, distinto framework. |
| Jerarquía de injectors (componente → ruta → root) | Scope → root provider | En .NET la jerarquía es más plana: solo root y scopes. |

**Diferencia importante:** en Angular la jerarquía sigue el **árbol de componentes** (todo en un solo hilo, en el navegador). En .NET el scope sigue la **petición HTTP** y el Singleton lo comparten **muchos hilos a la vez**, por eso ahí la thread-safety sí importa.

---

## Ejemplo de aplicación real

**Escenario:** la API del **sistema de reservas de hotel** (el mismo dominio del ejemplo de Angular). Hay:
- una caché de tarifas que se recalcula cada pocos minutos,
- servicios de reservas que usan EF Core,
- el usuario actual (sacado del JWT),
- un **BackgroundService** que cada noche cancela reservas no pagadas.

### Registro

```csharp
// Program.cs
builder.Services.AddDbContext<HotelDbContext>(o => o.UseNpgsql(cs));    // Scoped

builder.Services.AddSingleton<IRateCache, InMemoryRateCache>();           // compartido + thread-safe
builder.Services.AddSingleton(TimeProvider.System);                        // reloj abstracto (testeable)

builder.Services.AddScoped<ICurrentUser, HttpCurrentUser>();               // el usuario de ESTA request
builder.Services.AddScoped<IReservationService, ReservationService>();     // usa DbContext → Scoped

builder.Services.AddTransient<IValidator<CreateReservationDto>, CreateReservationValidator>(); // sin estado

builder.Services.AddHostedService<UnpaidReservationsCleaner>();            // los hosted services son Singleton
```

### Singleton: caché thread-safe

```csharp
public class InMemoryRateCache : IRateCache
{
    private readonly ConcurrentDictionary<(int RoomTypeId, DateOnly Date), decimal> _rates = new(); // thread-safe

    public decimal? Get(int roomTypeId, DateOnly date) =>
        _rates.TryGetValue((roomTypeId, date), out var price) ? price : null;

    public void Set(int roomTypeId, DateOnly date, decimal price) => _rates[(roomTypeId, date)] = price;
}
```

> Si usara un `Dictionary` normal, dos requests concurrentes podrían corromperlo. **Singleton ⇒ thread-safe.**

### Scoped: usuario actual + servicio de negocio

```csharp
public class HttpCurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    public int Id => int.Parse(accessor.HttpContext!.User.FindFirst("sub")!.Value);
}

public class ReservationService(HotelDbContext db, ICurrentUser user, IRateCache rates, TimeProvider clock)
    : IReservationService
{
    public async Task<Reservation> CreateAsync(CreateReservationDto dto, CancellationToken ct)
    {
        var price = rates.Get(dto.RoomTypeId, dto.CheckIn) ?? throw new InvalidOperationException("Sin tarifa");
        var reservation = new Reservation
        {
            GuestId   = user.Id,
            RoomTypeId = dto.RoomTypeId,
            CheckIn   = dto.CheckIn,
            Total     = price * dto.Nights,
            CreatedAt = clock.GetUtcNow(),
        };
        db.Reservations.Add(reservation);
        await db.SaveChangesAsync(ct);
        return reservation;
    }
}
```

Dentro de una misma request, el controlador, el `ReservationService` y cualquier otro servicio reciben **el mismo** `HotelDbContext`. Así comparten la misma transacción y el mismo *change tracker*.

### El bug real: captive dependency en el BackgroundService

```csharp
// ❌ MAL: los hosted services son Singleton → el DbContext queda capturado para siempre
public class UnpaidReservationsCleaner(HotelDbContext db) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            // el mismo DbContext durante días: el change tracker crece sin límite (memory leak),
            // datos cacheados viejos, y si algo falla una vez, el contexto queda inservible
            await db.Reservations.Where(...).ExecuteUpdateAsync(...);
            await Task.Delay(TimeSpan.FromHours(24), ct);
        }
    }
}
```

```csharp
// ✅ BIEN: crear un scope por cada ejecución
public class UnpaidReservationsCleaner(IServiceScopeFactory scopeFactory, ILogger<UnpaidReservationsCleaner> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(24));
        do
        {
            await using var scope = scopeFactory.CreateAsyncScope();               // "una request" artificial
            var db = scope.ServiceProvider.GetRequiredService<HotelDbContext>();   // DbContext nuevo

            var cancelled = await db.Reservations
                .Where(r => r.Status == ReservationStatus.PendingPayment
                         && r.CreatedAt < DateTimeOffset.UtcNow.AddHours(-48))
                .ExecuteUpdateAsync(s => s.SetProperty(r => r.Status, ReservationStatus.Cancelled), ct);

            log.LogInformation("Reservas canceladas: {Count}", cancelled);
        }                                                                          // al salir: Dispose del scope y del DbContext
        while (await timer.WaitForNextTickAsync(ct));
    }
}
```

> Es la misma idea que en Angular cuando cada **tab** del editor tenía su propio store: **cada unidad de trabajo tiene su propia instancia** y se limpia al terminar.

---

## Errores comunes / trampas

- **Captive dependency**: Scoped/Transient dentro de un Singleton (incluye `BackgroundService`, middleware convencional y `IHostedService`).
- **Singleton con estado no thread-safe**: `List<T>`, `Dictionary<,>` o campos mutables compartidos entre requests.
- **`DbContext` como Singleton o compartido entre tareas paralelas** → `DbContext` no es thread-safe.
- **`new HttpClient()` en cada llamada** → agota sockets (*socket exhaustion*). Y un `HttpClient` Singleton "a mano" no respeta cambios de DNS. Lo correcto es `IHttpClientFactory` / `AddHttpClient`.
- **Transient que implementa `IDisposable` resuelto desde el root provider** → el contenedor lo guarda para hacer `Dispose` al final y se acumula en memoria.
- **Service Locator**: inyectar `IServiceProvider` en todos lados y hacer `GetService` a mano. Esconde las dependencias y complica los tests. Úsalo solo en fábricas o en código de infraestructura.
- **Middleware**: se construye una sola vez (como Singleton). Los servicios Scoped se piden como parámetro de `InvokeAsync`, no en el constructor.

---

## Preguntas de seguimiento para practicar

1. ¿Qué es una *captive dependency* y cómo la detecta ASP.NET Core?
2. ¿Por qué `DbContext` se registra como Scoped y no como Singleton ni Transient?
3. Si registro dos implementaciones de la misma interfaz, ¿cuál se inyecta? ¿Cómo obtengo todas? (pista: `IEnumerable<IX>`). ¿Qué son los *keyed services* (`AddKeyedScoped`, `[FromKeyedServices]`)?
4. ¿Cómo usas un servicio Scoped dentro de un `BackgroundService`?
5. ¿Qué diferencia hay entre `GetService` y `GetRequiredService`?
6. Compara: ¿qué equivale en .NET a proveer un servicio en los `providers` de un componente Angular? ¿Por qué no hay un equivalente exacto de Transient en Angular?
7. ¿Por qué se recomienda `IHttpClientFactory` en lugar de `new HttpClient()`?
