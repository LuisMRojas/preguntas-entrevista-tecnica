# 3. Cómo evitar memory leaks con RxJS

> **Pregunta típica:** "¿Cómo evitas fugas de memoria al usar observables en Angular?"

---

## Respuesta corta (para decir en entrevista)

> Un memory leak ocurre cuando un componente se destruye pero una **suscripción sigue viva**, porque el observable no se completa (un `interval`, un `Subject` de un servicio singleton, un `fromEvent`, un WebSocket). La suscripción mantiene referencias al componente, así que el garbage collector no lo libera, y el callback se sigue ejecutando. Para evitarlo, mi primera opción es **no suscribirme manualmente**: uso el **`async` pipe** o **`toSignal()`**, que se desuscriben solos. Si necesito suscribirme en código, uso **`takeUntilDestroyed()`** (o el patrón `takeUntil(destroy$)` en versiones antiguas). Y para observables que deberían emitir una sola vez uso `take(1)` / `first()`. Las peticiones de `HttpClient` se completan solas, pero igual conviene cancelarlas si el componente se destruye.

---

## Explicación a fondo

### ¿Por qué hay fuga?

```ts
@Component({ /* ... */ })
export class LeakyComponent {
  constructor() {
    interval(1000).subscribe(n => console.log('sigo vivo', n)); // ❌ nunca termina
  }
}
```

Navegas a otra ruta → el componente se destruye → **el `interval` sigue corriendo** y mantiene vivo el componente en memoria. Si entras y sales 10 veces, tienes 10 intervals.

### ¿Qué observables NO necesitan desuscripción?

Los que **completan solos** y de forma finita:
- `HttpClient` (`get`, `post`...) → emite una vez y completa.
- `of(...)`, `from([...])`.
- Observables con `take(n)`, `first()`.

> Aun así, si una petición HTTP es lenta y el usuario se va, la respuesta llegará y ejecutará el callback sobre un componente destruido. Por eso igual es buena práctica atarla al ciclo de vida.

### ¿Cuáles SÍ son peligrosos?

- `interval`, `timer` con periodo, `fromEvent(window, 'resize')`.
- `Subject` / `BehaviorSubject` de servicios `providedIn: 'root'` (viven toda la app).
- `router.events`, `activatedRoute.params`*, `form.valueChanges`*, `store.select(...)`.
- WebSockets, `EventSource`.

\* Angular limpia algunos internamente cuando el componente/form se destruye, pero no conviene depender de eso.

---

## Las técnicas, de mejor a peor

### 1. `async` pipe (o `toSignal`) — sin suscripción manual

```ts
@Component({
  selector: 'app-user-profile',
  imports: [AsyncPipe],
  template: `
    @if (user$ | async; as user) {
      <h2>{{ user.name }}</h2>
    }
  `,
})
export class UserProfileComponent {
  user$ = inject(UserService).currentUser$; // el pipe se suscribe y desuscribe solo
}
```

```ts
// Versión con signals
export class UserProfileComponent {
  user = toSignal(inject(UserService).currentUser$); // se desuscribe al destruirse
}
```

### 2. `takeUntilDestroyed()` — cuando necesitas `subscribe`

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class SearchComponent {
  search = new FormControl('');
  private api = inject(ProductsApi);

  constructor() {
    this.search.valueChanges
      .pipe(
        debounceTime(300),
        distinctUntilChanged(),
        switchMap(term => this.api.search(term ?? '')), // cancela la petición anterior
        takeUntilDestroyed(), // ✅ siempre al final de la cadena
      )
      .subscribe(results => (this.results = results));
  }
}
```

> Si se usa **fuera del constructor** (por ejemplo en `ngOnInit`), hay que pasarle el `DestroyRef`:
>
> ```ts
> private destroyRef = inject(DestroyRef);
> ngOnInit() {
>   source$.pipe(takeUntilDestroyed(this.destroyRef)).subscribe();
> }
> ```

### 3. Patrón clásico `takeUntil(destroy$)` (Angular < 16 / código legado)

```ts
export class LegacyComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.service.data$
      .pipe(takeUntil(this.destroy$))
      .subscribe(d => (this.data = d));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 4. Guardar la `Subscription` y llamar `unsubscribe()`

```ts
private sub = new Subscription();
ngOnInit()    { this.sub.add(a$.subscribe()); this.sub.add(b$.subscribe()); }
ngOnDestroy() { this.sub.unsubscribe(); }
```

Funciona, pero es fácil olvidarse de agregar una suscripción.

### 5. `take(1)` / `first()` — cuando solo necesito un valor

```ts
this.auth.user$.pipe(take(1)).subscribe(user => this.audit.log(user.id));
```

> Cuidado: si el observable **nunca emite**, `take(1)` nunca completa → sigue siendo un leak. `first()` además lanza error si completa sin emitir.

---

## Ejemplo de aplicación real

**Escenario:** una pantalla de **monitoreo en tiempo real** de una flota de vehículos. Al entrar se hace *polling* cada 5 segundos al backend y se escucha el `resize` de la ventana para redimensionar el mapa. Los usuarios entran y salen de la pantalla muchas veces al día; antes de corregirlo, el navegador terminaba haciendo decenas de requests por segundo y consumiendo cientos de MB.

```ts
// fleet-monitor.component.ts
import { ChangeDetectionStrategy, Component, inject, ElementRef } from '@angular/core';
import { takeUntilDestroyed, toSignal } from '@angular/core/rxjs-interop';
import { timer, fromEvent, switchMap, debounceTime, retry, shareReplay } from 'rxjs';

@Component({
  selector: 'app-fleet-monitor',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <app-map [vehicles]="vehicles() ?? []" />
    <p>{{ vehicles()?.length ?? 0 }} vehículos activos</p>
  `,
})
export class FleetMonitorComponent {
  private api = inject(FleetApi);
  private map = inject(MapService);

  // ✅ polling: toSignal se desuscribe al destruir el componente → se detiene el timer
  vehicles = toSignal(
    timer(0, 5000).pipe(
      switchMap(() => this.api.getVehicles()), // si una request tarda, se cancela la anterior
      retry({ delay: 2000 }),
    ),
  );

  constructor() {
    // ✅ listener global atado al ciclo de vida
    fromEvent(window, 'resize')
      .pipe(debounceTime(200), takeUntilDestroyed())
      .subscribe(() => this.map.invalidateSize());
  }
}
```

**Resultado:** al salir de la ruta se cancelan el `timer`, la request en curso y el listener de `resize`.

### Bonus: leaks en servicios con `shareReplay`

```ts
// ❌ shareReplay(1) sin refCount mantiene la suscripción a la fuente aunque nadie escuche
readonly prices$ = this.socket.prices$.pipe(shareReplay(1));

// ✅ cuando el último suscriptor se va, se desuscribe de la fuente
readonly prices$ = this.socket.prices$.pipe(shareReplay({ bufferSize: 1, refCount: true }));
```

---

## Cómo detectar un leak

1. Chrome DevTools → **Memory** → *Heap snapshot*.
2. Entra y sale varias veces de la pantalla sospechosa.
3. Toma otro snapshot y busca el nombre del componente (`FleetMonitorComponent`): si hay muchas instancias "Detached", hay leak.
4. Un `console.log` dentro del `subscribe` que se sigue imprimiendo tras salir de la ruta es la pista más rápida.

---

## Errores comunes / trampas

- Poner `takeUntil`/`takeUntilDestroyed` **antes** de `switchMap`/`mergeMap`: el observable interno puede seguir vivo. Ponlo **al final**.
- Suscripciones anidadas (`subscribe` dentro de `subscribe`) → usa `switchMap`, `mergeMap`, `concatMap` o `exhaustMap`.
- Usar `takeUntilDestroyed()` fuera de un contexto de inyección sin pasar `DestroyRef` → error en tiempo de ejecución.
- Pensar que los servicios `providedIn: 'root'` se destruyen: viven toda la aplicación.
- `shareReplay(1)` sin `refCount: true` sobre una fuente infinita.

---

## Preguntas de seguimiento para practicar

1. ¿Por qué hay que poner `takeUntil` al final del `pipe`?
2. ¿Hace falta desuscribirse de `HttpClient`? ¿Por qué igual podría convenir?
3. ¿Qué diferencia hay entre `switchMap`, `mergeMap`, `concatMap` y `exhaustMap`? ¿Cuál usarías para un botón de "Guardar"?
4. ¿Qué es `DestroyRef` y para qué sirve fuera de componentes?
5. ¿Cómo encontrarías un memory leak en producción?
