# 2. Observable, Subject y BehaviorSubject

> **Pregunta típica:** "¿Cuál es la diferencia entre un Observable, un Subject y un BehaviorSubject? ¿Cuándo usarías cada uno?"

---

## Respuesta corta (para decir en entrevista)

> Un **Observable** es un flujo de datos *lazy* y normalmente **unicast**: no hace nada hasta que alguien se suscribe, y cada suscriptor recibe su propia ejecución (por ejemplo, cada `subscribe` a `http.get` hace una petición nueva). Un **Subject** es a la vez Observable y Observer: puedo empujarle valores con `next()` y es **multicast**, todos los suscriptores comparten la misma emisión, pero quien se suscribe tarde **se pierde** los valores anteriores. Un **BehaviorSubject** es un Subject que **necesita un valor inicial** y **guarda el último valor**: cualquier suscriptor nuevo lo recibe inmediatamente, y además puedo leerlo de forma síncrona con `.value`. Uso Observable para fuentes como HTTP, Subject para **eventos** (clicks, "se guardó algo") y BehaviorSubject para **estado** (usuario logueado, carrito, filtros).

---

## Explicación a fondo

### Observable

```ts
const obs$ = new Observable<number>(subscriber => {
  console.log('Ejecutando...');
  subscriber.next(Math.random());
  subscriber.complete();
});

obs$.subscribe(v => console.log('A', v)); // Ejecutando... A 0.12
obs$.subscribe(v => console.log('B', v)); // Ejecutando... B 0.87  ← otra ejecución, otro valor
```

- **Cold / lazy**: el productor se crea al suscribirse.
- **Unicast**: una ejecución por suscriptor.

### Subject

```ts
const subject = new Subject<number>();

subject.subscribe(v => console.log('A', v));
subject.next(1);                         // A 1
subject.subscribe(v => console.log('B', v));
subject.next(2);                         // A 2, B 2   ← B se perdió el 1
```

- **Hot / multicast**: una sola fuente compartida.
- Sin memoria: los suscriptores tardíos no reciben nada previo.

### BehaviorSubject

```ts
const state = new BehaviorSubject<number>(0);

state.subscribe(v => console.log('A', v)); // A 0  ← recibe el valor actual al instante
state.next(1);                             // A 1
state.subscribe(v => console.log('B', v)); // B 1  ← recibe el último valor
console.log(state.value);                  // 1    ← lectura síncrona
```

### Tabla comparativa

| | Observable | Subject | BehaviorSubject | ReplaySubject(n) |
|---|---|---|---|---|
| Valor inicial | — | No | **Sí (obligatorio)** | No |
| Suscriptor tardío recibe... | Su propia ejecución | Nada previo | **Último valor** | Últimos **n** valores |
| Multicast | No (por defecto) | Sí | Sí | Sí |
| Puede emitir desde fuera (`next`) | No | Sí | Sí | Sí |
| Lectura síncrona | No | No | `.value` | No |
| Uso típico | HTTP, timers, eventos DOM | Eventos / acciones | **Estado** | Caché / historial |

> Bonus: **AsyncSubject** solo emite el último valor y solo cuando se completa (poco común).

---

## Ejemplo de aplicación real

**Escenario:** una app con login, carrito de compras y notificaciones tipo "toast".

### BehaviorSubject → estado del carrito (servicio tipo "store")

```ts
// cart.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, map } from 'rxjs';

export interface CartItem { productId: string; name: string; price: number; qty: number; }

@Injectable({ providedIn: 'root' })
export class CartService {
  // privado: solo el servicio puede modificar el estado
  private readonly _items$ = new BehaviorSubject<CartItem[]>([]);

  // público y de solo lectura: los componentes solo pueden observar
  readonly items$ = this._items$.asObservable();
  readonly count$ = this.items$.pipe(map(items => items.reduce((n, i) => n + i.qty, 0)));
  readonly total$ = this.items$.pipe(map(items => items.reduce((t, i) => t + i.price * i.qty, 0)));

  add(item: CartItem) {
    const current = this._items$.value; // lectura síncrona del estado actual
    const existing = current.find(i => i.productId === item.productId);
    const next = existing
      ? current.map(i => (i.productId === item.productId ? { ...i, qty: i.qty + 1 } : i))
      : [...current, { ...item, qty: 1 }];
    this._items$.next(next); // inmutable
  }

  remove(productId: string) {
    this._items$.next(this._items$.value.filter(i => i.productId !== productId));
  }
}
```

¿Por qué **BehaviorSubject**? Si el usuario navega a `/checkout` después de agregar productos, el componente del checkout se suscribe *tarde* y **igual necesita** ver el carrito actual. Con un `Subject` vería un carrito vacío hasta el próximo cambio.

```ts
// header.component.ts
@Component({
  selector: 'app-header',
  imports: [AsyncPipe],
  template: `🛒 {{ cart.count$ | async }}`,
})
export class HeaderComponent {
  cart = inject(CartService);
}
```

### Subject → eventos de un solo uso (toasts)

```ts
// toast.service.ts
@Injectable({ providedIn: 'root' })
export class ToastService {
  private readonly _messages$ = new Subject<{ text: string; type: 'ok' | 'error' }>();
  readonly messages$ = this._messages$.asObservable();

  success(text: string) { this._messages$.next({ text, type: 'ok' }); }
  error(text: string)   { this._messages$.next({ text, type: 'error' }); }
}
```

¿Por qué **Subject** y no BehaviorSubject? Un toast es un **evento**: si un componente se monta más tarde, **no** queremos que vuelva a mostrar "Producto agregado" de hace 5 minutos.

### Observable → petición HTTP

```ts
@Injectable({ providedIn: 'root' })
export class ProductsApi {
  private http = inject(HttpClient);
  getProducts() {
    return this.http.get<Product[]>('/api/products'); // cold: cada subscribe = 1 request
  }
}
```

Si varios componentes necesitan la misma respuesta, se convierte en multicast con `shareReplay`:

```ts
readonly products$ = this.http.get<Product[]>('/api/products').pipe(
  shareReplay({ bufferSize: 1, refCount: true }),
);
```

### Equivalente moderno con signals

En Angular moderno, para estado síncrono de UI muchas veces se usa `signal()` en lugar de BehaviorSubject. Son conceptos parecidos (valor actual + notificación de cambios), pero los signals no manejan tiempo/asincronía. Regla práctica:

- **Estado** sincrónico en componentes/servicios → `signal` / `computed`.
- **Flujos asíncronos** (HTTP, WebSocket, debounce, cancelación) → RxJS.
- Puente entre ambos: `toSignal()` y `toObservable()`.

---

## Errores comunes / trampas

- **Exponer el Subject público**: cualquier componente podría hacer `.next()`. Expón `asObservable()`.
- Usar `BehaviorSubject` para eventos → los suscriptores nuevos reciben un evento "viejo".
- Usar `Subject` para estado → los suscriptores tardíos ven "nada".
- Abusar de `.value` en todos lados: rompe el modelo reactivo; úsalo solo dentro del servicio para calcular el siguiente estado.
- Olvidar que un `Subject` completado (`complete()`) ya no emite más.

---

## Preguntas de seguimiento para practicar

1. ¿Qué es un Observable *cold* vs *hot*? Da un ejemplo de cada uno.
2. ¿Qué diferencia hay entre `BehaviorSubject` y `ReplaySubject(1)`?
3. ¿Qué hace `shareReplay` y qué riesgo tiene si no usas `refCount: true`?
4. ¿Cuándo usarías signals en lugar de BehaviorSubject?
5. ¿Por qué es buena práctica exponer `asObservable()`?
