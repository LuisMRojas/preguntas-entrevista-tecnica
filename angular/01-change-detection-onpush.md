# 1. Change detection y cuándo usar OnPush

> **Pregunta típica:** "¿Cómo funciona la detección de cambios en Angular y cuándo usarías `ChangeDetectionStrategy.OnPush`?"

---

## Respuesta corta (para decir en entrevista)

> La detección de cambios es el proceso con el que Angular compara el estado de los componentes con lo que está pintado en el DOM y actualiza lo que cambió. Con la estrategia **Default**, Angular revisa **todo el árbol** de componentes cada vez que ocurre algo asíncrono (un click, un HTTP, un `setTimeout`). Con **OnPush**, un componente solo se revisa cuando: cambia la **referencia** de un `@Input`, ocurre un **evento dentro** del componente, emite un `async` pipe / cambia un **signal** que usa la plantilla, o lo marco manualmente con `markForCheck()`. Uso OnPush casi por defecto, sobre todo en componentes de presentación y listas grandes, porque mejora el rendimiento y me obliga a trabajar con datos **inmutables**.

---

## Explicación a fondo

### ¿Qué dispara la detección de cambios?

Históricamente Angular usa **Zone.js**, que "parchea" las APIs asíncronas del navegador (eventos, `setTimeout`, `Promise`, XHR). Cuando una de esas tareas termina, Zone avisa a Angular y este ejecuta `ApplicationRef.tick()`, que recorre el árbol de componentes de arriba hacia abajo.

En versiones recientes Angular soporta (y en proyectos nuevos usa por defecto) el modo **zoneless**: ya no depende de Zone.js y la detección se agenda cuando cambia un **signal** leído en la plantilla, un `async` pipe emite, ocurre un evento de plantilla o se llama a `markForCheck()`. En zoneless, escribir componentes como si fueran OnPush es prácticamente obligatorio.

### Default vs OnPush

| | Default | OnPush |
|---|---|---|
| ¿Cuándo se revisa el componente? | En **cada** ciclo de detección | Solo si fue **marcado como "sucio"** |
| Cambio de `@Input` | Detecta incluso mutaciones (`obj.x = 1`) | Solo detecta **nueva referencia** (`obj = {...obj, x: 1}`) |
| Rendimiento | Peor en árboles grandes | Mucho mejor: salta subárboles enteros |
| Exige inmutabilidad | No | Sí |

### ¿Qué marca un componente OnPush como "sucio"?

1. Un `@Input()` / `input()` recibe una **nueva referencia**.
2. Un **evento** del DOM dentro de su plantilla (o de sus hijos) — `(click)`, `(input)`, etc.
3. El **`async` pipe** recibe un nuevo valor (internamente llama `markForCheck()`).
4. Un **signal** leído en la plantilla cambia.
5. Llamada manual a `ChangeDetectorRef.markForCheck()`.

> Importante: `markForCheck()` marca el componente **y todos sus ancestros** hasta la raíz, para que el próximo ciclo llegue hasta él. `detectChanges()` en cambio ejecuta la detección **ya**, de forma síncrona, solo en ese componente y sus hijos.

### ¿Qué NO dispara OnPush?

- Mutar un objeto o array de un input: `this.items.push(x)` → **no se actualiza la vista**.
- Cambiar una propiedad normal desde un `setTimeout` o una suscripción manual sin `markForCheck()`.

---

## Ejemplo de aplicación real

**Escenario:** un dashboard de pedidos de un e-commerce. Hay una tabla con cientos de filas que se actualiza por WebSocket, y a la vez el usuario escribe en un buscador y hay un reloj en el header que cambia cada segundo. Con estrategia Default, cada tick del reloj re-evaluaría las cientos de filas.

### Componente de presentación (fila) con OnPush

```ts
// order-row.component.ts
import { ChangeDetectionStrategy, Component, input, output } from '@angular/core';
import { CurrencyPipe } from '@angular/common';

export interface Order {
  id: string;
  customer: string;
  total: number;
  status: 'pending' | 'shipped' | 'delivered';
}

@Component({
  selector: 'tr[app-order-row]', // selector de atributo para usarlo dentro de <table>
  standalone: true,
  imports: [CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <td>{{ order().id }}</td>
    <td>{{ order().customer }}</td>
    <td>{{ order().total | currency }}</td>
    <td>{{ order().status }}</td>
    <td><button (click)="ship.emit(order().id)">Enviar</button></td>
  `,
})
export class OrderRowComponent {
  order = input.required<Order>();
  ship = output<string>();
}
```

### Componente contenedor: actualiza de forma inmutable

```ts
// orders-page.component.ts
import { ChangeDetectionStrategy, Component, inject, signal, computed } from '@angular/core';
import { OrderRowComponent, Order } from './order-row.component';
import { OrdersSocketService } from './orders-socket.service';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-orders-page',
  standalone: true,
  imports: [OrderRowComponent],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input placeholder="Buscar cliente" (input)="search.set($any($event.target).value)" />
    <table>
      @for (o of filtered(); track o.id) {
        <tr app-order-row [order]="o" (ship)="markShipped($event)"></tr>
      }
    </table>
  `,
})
export class OrdersPageComponent {
  private socket = inject(OrdersSocketService);

  orders = signal<Order[]>([]);
  search = signal('');
  filtered = computed(() =>
    this.orders().filter(o =>
      o.customer.toLowerCase().includes(this.search().toLowerCase()),
    ),
  );

  constructor() {
    this.socket.updates$
      .pipe(takeUntilDestroyed())
      .subscribe(updated => {
        // ✅ Inmutable: solo la fila modificada recibe una nueva referencia
        this.orders.update(list =>
          list.map(o => (o.id === updated.id ? updated : o)),
        );
      });
  }

  markShipped(id: string) {
    // ❌ MAL con OnPush: this.orders().find(o => o.id === id)!.status = 'shipped';
    // ✅ BIEN:
    this.orders.update(list =>
      list.map(o => (o.id === id ? { ...o, status: 'shipped' } : o)),
    );
  }
}
```

**Por qué funciona bien:**
- Cuando llega una actualización, solo **una** fila recibe un objeto nuevo → solo esa fila se re-renderiza.
- El `track o.id` evita que Angular destruya y recree los nodos del DOM.
- El reloj del header (en otro componente) ya no fuerza a revisar las filas.

### Caso con código legado: suscripción manual + `markForCheck()`

```ts
@Component({
  selector: 'app-notifications-badge',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<span class="badge">{{ count }}</span>`,
})
export class NotificationsBadgeComponent {
  count = 0;
  private cdr = inject(ChangeDetectorRef);

  constructor() {
    inject(NotificationsService).count$
      .pipe(takeUntilDestroyed())
      .subscribe(c => {
        this.count = c;
        this.cdr.markForCheck(); // sin esto, el badge nunca se actualiza
      });
  }
}
```

> Mejor aún: evitar la suscripción manual usando `count = toSignal(service.count$)` o `{{ count$ | async }}`.

---

## Errores comunes / trampas

- **Mutar arrays/objetos** con OnPush y preguntarse por qué no se actualiza la vista.
- Usar `detectChanges()` en todas partes como "parche" → oculta problemas de diseño.
- Llamar **funciones en la plantilla** (`{{ calcularTotal() }}`): con Default se ejecutan en cada ciclo. Usa `computed()` o un pipe puro.
- Pensar que OnPush "congela" el componente: los eventos internos y el `async` pipe siguen funcionando.
- Error `ExpressionChangedAfterItHasBeenCheckedError`: aparece en desarrollo cuando cambias un valor *después* de que fue revisado (ej. en `ngAfterViewInit`). No se arregla con OnPush; se arregla moviendo la lógica o usando signals.

---

## Preguntas de seguimiento para practicar

1. ¿Cuál es la diferencia entre `markForCheck()` y `detectChanges()`?
2. Si un componente padre es OnPush y el hijo es Default, ¿se revisa el hijo cuando cambia algo fuera del padre?
3. ¿Por qué el `async` pipe funciona con OnPush sin hacer nada extra?
4. ¿Qué cambia con Angular zoneless? ¿Qué papel juegan los signals?
5. ¿Qué es `NgZone.runOutsideAngular()` y cuándo lo usarías? (pista: listeners de `scroll`/`mousemove`, animaciones con `requestAnimationFrame`).
