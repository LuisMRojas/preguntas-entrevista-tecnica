# 4. Inyección de dependencias y alcance de los providers

> **Pregunta típica:** "¿Cómo funciona la inyección de dependencias en Angular? ¿Qué diferencia hay entre `providedIn: 'root'`, proveer en una ruta y proveer en un componente?"

---

## Respuesta corta (para decir en entrevista)

> La inyección de dependencias (DI) es un patrón donde una clase **no crea** sus dependencias sino que **las pide**, y Angular se las entrega desde un **injector**. Los injectors forman una **jerarquía**: está el **environment injector** (raíz de la app y los de rutas lazy) y el **element injector** (uno por componente/directiva). Cuando pido una dependencia, Angular la busca desde el componente hacia arriba hasta encontrar un provider. El **alcance** depende de dónde lo registro: con `providedIn: 'root'` es un **singleton** para toda la app y además es *tree-shakable*; en los `providers` de una **ruta** se crea una instancia para esa sección de la app; y en los `providers` de un **componente** se crea **una instancia por cada instancia del componente**, que se destruye con él. Esto lo uso, por ejemplo, para que cada formulario o cada tab tenga su propio estado aislado.

---

## Explicación a fondo

### Formas de inyectar

```ts
// Moderno (preferido): función inject()
export class OrdersComponent {
  private api = inject(OrdersApi);
}

// Clásico: por constructor
export class OrdersComponent {
  constructor(private api: OrdersApi) {}
}
```

`inject()` solo funciona en un **contexto de inyección**: inicializadores de campos, constructor, factories, guards/resolvers/interceptores funcionales, o dentro de `runInInjectionContext()`.

### Jerarquía de injectors (de abajo hacia arriba)

```
Element injector del componente hijo   ← providers: [...] / viewProviders
        ↑
Element injector del componente padre
        ↑
Environment injector de la ruta lazy   ← providers en Route
        ↑
Root environment injector             ← providedIn: 'root' / bootstrapApplication providers
        ↑
Platform injector                      ← providedIn: 'platform'
        ↑
NullInjector                           → NullInjectorError: No provider for X!
```

Angular primero recorre los **element injectors** (árbol de componentes) y si no encuentra nada, sigue por los **environment injectors**.

### Tipos de providers

```ts
providers: [
  OrdersApi,                                          // useClass implícito
  { provide: Logger, useClass: ConsoleLogger },        // otra implementación
  { provide: API_URL, useValue: 'https://api.x.com' }, // valor
  { provide: Cache, useFactory: () => new Cache(inject(API_URL)) }, // fábrica
  { provide: OldLogger, useExisting: Logger },         // alias (misma instancia)
]
```

Para valores que no son clases se usa `InjectionToken`:

```ts
export const API_URL = new InjectionToken<string>('API_URL');
```

### Alcances: dónde registro el provider

| Dónde | Instancias | Vida | Uso típico |
|---|---|---|---|
| `@Injectable({ providedIn: 'root' })` | 1 en toda la app | Toda la app | Auth, HTTP APIs, stores globales |
| `bootstrapApplication(App, { providers })` / `app.config.ts` | 1 en toda la app | Toda la app | Config, interceptores, router |
| `providers` de una **Route** | 1 por ese injector de ruta | Mientras exista el injector de la ruta | Estado de un feature (ej. admin) |
| `providers` de un **Component** | **1 por instancia del componente** (compartida con sus hijos) | Se destruye con el componente | Estado local, wizards, formularios |
| `viewProviders` de un Component | Igual, pero **no** visible para contenido proyectado (`<ng-content>`) | Igual | Encapsular de verdad |

### Modificadores de resolución

```ts
inject(Logger, { optional: true }); // null si no existe, en vez de error
inject(Logger, { self: true });     // solo en el injector propio
inject(Logger, { skipSelf: true }); // empieza a buscar en el padre
inject(Logger, { host: true });     // no sube más allá del componente host
```

---

## Ejemplo de aplicación real

**Escenario:** una aplicación de **gestión de reservas de hotel**.

### 1. Singleton global → sesión del usuario

```ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly _user = signal<User | null>(null);
  readonly user = this._user.asReadonly();
  readonly isLoggedIn = computed(() => this._user() !== null);
  // ...
}
```

Todos los componentes deben ver **el mismo** usuario → `root`.

### 2. Configuración por entorno con `InjectionToken`

```ts
// app.config.ts
export const API_URL = new InjectionToken<string>('API_URL');

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    { provide: API_URL, useValue: environment.apiUrl },
  ],
};

// reservations.api.ts
@Injectable({ providedIn: 'root' })
export class ReservationsApi {
  private http = inject(HttpClient);
  private baseUrl = inject(API_URL);
  list() { return this.http.get<Reservation[]>(`${this.baseUrl}/reservations`); }
}
```

### 3. Provider a nivel de ruta → estado del módulo de administración

```ts
// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    canActivate: [adminGuard],
    providers: [AdminReportsStore], // solo existe dentro de /admin/**
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
  },
];
```

`AdminReportsStore` se comparte entre todas las páginas de `/admin`, pero el resto de la app no lo ve ni lo carga.

### 4. Provider a nivel de componente → un estado por instancia

El recepcionista puede tener **varias reservas abiertas en tabs** a la vez. Cada tab necesita su propio estado de edición (fechas, habitación, huéspedes, "hay cambios sin guardar").

```ts
// reservation-editor.store.ts
@Injectable() // ⚠️ sin providedIn: se provee en el componente
export class ReservationEditorStore {
  private api = inject(ReservationsApi);
  readonly draft = signal<Reservation | null>(null);
  readonly dirty = signal(false);

  load(id: string) { /* ... */ }
  update(changes: Partial<Reservation>) {
    this.draft.update(d => (d ? { ...d, ...changes } : d));
    this.dirty.set(true);
  }
}

// reservation-editor.component.ts
@Component({
  selector: 'app-reservation-editor',
  providers: [ReservationEditorStore], // ✅ nueva instancia por cada editor/tab
  imports: [GuestsFormComponent, RoomPickerComponent],
  template: `
    <app-room-picker />
    <app-guests-form />
    @if (store.dirty()) { <span>Cambios sin guardar</span> }
  `,
})
export class ReservationEditorComponent {
  store = inject(ReservationEditorStore);
  reservationId = input.required<string>();
  constructor() {
    effect(() => this.store.load(this.reservationId()));
  }
}

// guests-form.component.ts (hijo)
@Component({ selector: 'app-guests-form', template: `...` })
export class GuestsFormComponent {
  // recibe la MISMA instancia que su editor padre, no la de otra tab
  store = inject(ReservationEditorStore);
}
```

**Por qué:** si el store fuera `providedIn: 'root'`, editar la reserva de la tab 1 modificaría la tab 2. Con `providers` en el componente:
- cada tab tiene su propio store,
- los hijos (`room-picker`, `guests-form`) comparten el de **su** padre sin pasar `@Input`s por todos lados,
- al cerrar la tab, el store se destruye (y puede limpiar recursos con `inject(DestroyRef).onDestroy(...)`).

### 5. Sustituir implementaciones (útil en tests)

```ts
TestBed.configureTestingModule({
  providers: [{ provide: ReservationsApi, useValue: { list: () => of(mockReservations) } }],
});
```

---

## Errores comunes / trampas

- **`NullInjectorError: No provider for X`**: la clase tiene `@Injectable()` sin `providedIn` y nadie la provee.
- Poner un servicio en `providers` del componente **y** tener `providedIn: 'root'` → se crean **dos** instancias distintas; el componente usa la local. Fuente de bugs del tipo "el estado no se comparte".
- Proveer un servicio en una ruta lazy pensando que es global → en otras secciones habrá otra instancia (o error).
- Usar `inject()` fuera de un contexto de inyección (por ejemplo dentro de un `setTimeout` o un método llamado después).
- Confundir `providers` con `viewProviders` cuando se usa `<ng-content>`.
- En los antiguos `NgModule`: proveer un servicio en un módulo **lazy** creaba una instancia separada; en un módulo eager se "subía" al root. Por eso existía el patrón `forRoot()` / `forChild()`.

---

## Preguntas de seguimiento para practicar

1. ¿Por qué `providedIn: 'root'` es *tree-shakable* y `providers: [X]` en `AppModule` no?
2. ¿Qué pasa si dos componentes hermanos proveen el mismo servicio en sus `providers`?
3. ¿Cuál es la diferencia entre `useClass` y `useExisting`?
4. ¿Para qué sirve `InjectionToken` y cuándo es obligatorio?
5. ¿Qué hacen `skipSelf` y `optional`? Da un caso real (pista: evitar importar dos veces un módulo/servicio "core").
6. ¿Cuál es la diferencia entre `providers` y `viewProviders`?
