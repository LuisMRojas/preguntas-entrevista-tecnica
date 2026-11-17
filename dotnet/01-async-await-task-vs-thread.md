# 1. async/await y Task: qué pasa cuando hago `await`, y `Task` vs `Thread`

> **Pregunta típica:** "¿Qué pasa realmente cuando hago `await`? ¿Cuál es la diferencia entre un `Task` y un `Thread`?"

---

## Respuesta corta (para decir en entrevista)

> Un método `async` lo convierte el compilador en una **máquina de estados**. Cuando llego a un `await` sobre una tarea que **todavía no terminó**, el método **devuelve el control** a quien lo llamó (devuelve un `Task` incompleto) y **el hilo queda libre** para hacer otro trabajo. Cuando la operación termina, la **continuación** (el resto del método) se programa para seguir ejecutándose, normalmente en un hilo del **ThreadPool** o en el contexto original si existe uno, como el hilo de UI. `await` **no crea un hilo nuevo** ni bloquea: en operaciones de I/O (HTTP, base de datos, archivos) mientras se espera **no hay ningún hilo ocupado**.
>
> Un **`Thread`** es un hilo real del sistema operativo: caro de crear (aprox. 1 MB de stack) y de bajo nivel. Un **`Task`** es una **promesa de un resultado futuro**: representa una operación, que puede ejecutarse en un hilo del ThreadPool (`Task.Run`) o **no usar ningún hilo** (I/O asíncrono). Los `Task` se pueden componer (`WhenAll`, `WhenAny`), cancelar con `CancellationToken`, devolver valores y propagar excepciones. En código moderno casi nunca uso `Thread` directamente.

---

## Explicación a fondo

### Qué hace el compilador con `async`/`await`

```csharp
public async Task<Order> GetOrderAsync(int id)
{
    Console.WriteLine("1. Antes del await");          // corre síncrono, en el hilo que llamó
    var order = await _db.Orders.FindAsync(id);       // ← punto de suspensión
    Console.WriteLine("2. Después del await");        // continuación
    return order;
}
```

Paso a paso:

1. El método empieza **síncrono** en el hilo del llamador hasta el primer `await`.
2. `FindAsync` inicia la consulta y devuelve un `Task` **incompleto**.
3. `await` revisa: ¿ya terminó la tarea?
   - **Sí** → sigue de largo, síncrono, sin cambiar de hilo.
   - **No** → registra "el resto del método" como **continuación**, guarda las variables locales en la máquina de estados y **retorna** al llamador un `Task<Order>` incompleto.
4. El hilo queda libre. En ASP.NET Core vuelve al ThreadPool y puede atender **otra petición HTTP**.
5. Cuando la base de datos responde, el sistema operativo avisa (I/O completion port) y la continuación se ejecuta en un hilo del ThreadPool (que puede ser **otro hilo distinto**).
6. Al hacer `return order`, el `Task<Order>` se completa y se despierta a quien estaba esperando.

### "There is no thread": I/O vs CPU

| Tipo de trabajo | Ejemplo | ¿Usa un hilo mientras espera? | Qué usar |
|---|---|---|---|
| **I/O-bound** | HTTP, BD, archivos, colas | **No** | `await` de la API async (`GetAsync`, `ToListAsync`...) |
| **CPU-bound** | Generar un PDF, procesar una imagen, cálculos pesados | Sí, un hilo trabajando | `Task.Run` (en apps de UI/escritorio) o `Parallel` |

> En ASP.NET Core **no** se recomienda envolver código en `Task.Run` dentro de un endpoint: solo cambias un hilo del pool por otro y agregas overhead.

### `Task` vs `Thread`

| | `Thread` | `Task` |
|---|---|---|
| Qué es | Hilo del sistema operativo | Abstracción de "operación que terminará en el futuro" |
| Costo | Alto (stack ~1 MB, creación lenta) | Bajo; reutiliza el ThreadPool o no usa hilo |
| Valor de retorno | No (hay que usar variables compartidas) | Sí, `Task<T>` |
| Excepciones | Si no se capturan, tumban el proceso | Se guardan en el Task y se relanzan al hacer `await` |
| Composición | Manual (`Join`) | `await`, `WhenAll`, `WhenAny`, `ContinueWith` |
| Cancelación | Manual | `CancellationToken` |
| Cuándo usarlo | Casos muy específicos: hilo de larga vida con prioridad, STA/COM | **Casi siempre** |

### `SynchronizationContext` y `ConfigureAwait(false)`

- En apps de **UI** (WPF, WinForms, MAUI) hay un contexto que hace que la continuación vuelva al **hilo de UI** para poder tocar controles.
- En **ASP.NET Core no hay** `SynchronizationContext`, así que la continuación corre en cualquier hilo del pool.
- En **librerías** se usa `await algo.ConfigureAwait(false)` para no capturar el contexto: mejora el rendimiento y evita deadlocks si alguien llama la librería desde UI.

### El deadlock clásico: `.Result` / `.Wait()`

```csharp
// WinForms / ASP.NET clásico (.NET Framework)
private void Button_Click(object sender, EventArgs e)
{
    var data = LoadAsync().Result; // ❌ bloquea el hilo de UI...
}

private async Task<string> LoadAsync()
{
    await Task.Delay(1000);        // ...y la continuación quiere volver al hilo de UI → DEADLOCK
    return "ok";
}
```

El hilo de UI espera al Task, y el Task espera al hilo de UI. **Regla: "async all the way"**, no mezclar código síncrono bloqueante con async.

---

## Ejemplo de aplicación real

**Escenario:** una API de **e-commerce** con el endpoint `GET /orders/{id}/summary` que necesita:
- el pedido desde la base de datos (EF Core),
- el estado del envío desde la API de la empresa de transporte,
- los puntos de fidelidad desde otro microservicio.

### ❌ Versión bloqueante (lo que encontré en un proyecto legado)

```csharp
app.MapGet("/orders/{id}/summary", (int id, AppDbContext db, ShippingClient shipping, LoyaltyClient loyalty) =>
{
    var order    = db.Orders.Find(id);                         // bloquea un hilo ~40 ms
    var tracking = shipping.GetTrackingAsync(id).Result;       // bloquea ~300 ms
    var points   = loyalty.GetPointsAsync(order!.CustomerId).Result; // bloquea ~150 ms
    return new OrderSummary(order, tracking, points);
});
```

Con mucho tráfico, cada petición tiene un hilo **bloqueado sin hacer nada** ~500 ms. El ThreadPool se queda sin hilos (*thread pool starvation*), las peticiones se encolan y la latencia se dispara aunque el CPU esté al 10 %.

### ✅ Versión async + concurrente

```csharp
app.MapGet("/orders/{id}/summary", async (
    int id,
    AppDbContext db,
    ShippingClient shipping,
    LoyaltyClient loyalty,
    CancellationToken ct) =>            // se cancela si el cliente cierra la conexión
{
    var order = await db.Orders
        .AsNoTracking()
        .FirstOrDefaultAsync(o => o.Id == id, ct);

    if (order is null) return Results.NotFound();

    // Lanzo las dos llamadas externas a la vez: son independientes entre sí
    Task<Tracking> trackingTask = shipping.GetTrackingAsync(id, ct);
    Task<int>      pointsTask   = loyalty.GetPointsAsync(order.CustomerId, ct);

    await Task.WhenAll(trackingTask, pointsTask);   // espera ~300 ms (el máximo), no 450 ms (la suma)

    return Results.Ok(new OrderSummary(order, trackingTask.Result, pointsTask.Result));
    //                                        ↑ .Result aquí es seguro: las tareas ya terminaron
});
```

**Resultado:**
- Mientras se espera a la BD y a las APIs, **ningún hilo está bloqueado** → el mismo servidor atiende muchas más peticiones.
- La latencia baja porque las dos llamadas externas corren **en paralelo**.
- Si el usuario cancela, el `CancellationToken` cancela las llamadas y no se desperdician recursos.

### Caso CPU-bound: generar un PDF en una app de escritorio (WPF)

```csharp
private async void GenerateInvoice_Click(object sender, RoutedEventArgs e) // async void solo en event handlers
{
    StatusText.Text = "Generando...";
    byte[] pdf = await Task.Run(() => _pdfGenerator.Render(_invoice)); // CPU en el ThreadPool, la UI no se congela
    StatusText.Text = "Listo";                                         // vuelve al hilo de UI
    await File.WriteAllBytesAsync(path, pdf);                           // I/O async
}
```

---

## Errores comunes / trampas

- **`async void`**: las excepciones no se pueden capturar con `try/catch` del llamador y pueden tumbar el proceso. Úsalo **solo** en event handlers.
- **`.Result` / `.Wait()` / `.GetAwaiter().GetResult()`** sobre tareas no terminadas → deadlocks o *thread pool starvation*.
- **`await` en un bucle cuando las operaciones son independientes** → se ejecutan una tras otra. Usa `Task.WhenAll` (con cuidado de no saturar el servicio; para miles de elementos, `Parallel.ForEachAsync` con `MaxDegreeOfParallelism`).
- **Olvidar el `await`**: el método sigue sin esperar ("fire and forget" accidental) y las excepciones se pierden. El compilador avisa con la advertencia CS4014.
- **Usar el mismo `DbContext` en tareas paralelas** → `InvalidOperationException`: `DbContext` no es thread-safe. Por eso en el ejemplo la consulta a la BD va antes del `WhenAll`.
- **`Task.Run` en ASP.NET Core** para "hacerlo async" → no ganas nada.
- Pensar que `async` = "corre en otro hilo". `async` solo **habilita** `await`; un método async sin `await` corre totalmente síncrono.

---

## Preguntas de seguimiento para practicar

1. Si un método `async` nunca llega a un `await` incompleto, ¿cambia de hilo? ¿Qué devuelve?
2. ¿Por qué `.Result` puede causar deadlock en WinForms pero normalmente no en ASP.NET Core?
3. ¿Qué es `ConfigureAwait(false)` y cuándo lo usarías?
4. ¿Qué diferencia hay entre `Task.WhenAll` y hacer `await` uno por uno? ¿Qué pasa con las excepciones en `WhenAll`?
5. ¿Qué es `ValueTask` y cuándo conviene sobre `Task`? (pista: métodos que muchas veces terminan síncronos, como una lectura de caché).
6. ¿Qué es *thread pool starvation* y cómo lo detectarías? (pista: `dotnet-counters`, cola del ThreadPool).
7. ¿Cómo cancelas una operación async? ¿Qué pasa si ignoras el `CancellationToken`?
