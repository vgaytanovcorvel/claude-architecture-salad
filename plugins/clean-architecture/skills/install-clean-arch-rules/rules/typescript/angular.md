# TypeScript - Angular 19+ Rules

> This file extends [typescript/coding-style.md](coding-style.md) with Angular 19+ specifics.

---
paths: ["**/*.ts", "**/*.tsx", "**/*.component.ts", "**/*.service.ts", "**/*.component.html"]
---

## Component File Structure (Mandatory)

**ALWAYS** use external template and style files. **NEVER** use inline `template` or `styles`.

```
feature-name/
├── feature-name.component.ts        ← Class + metadata
├── feature-name.component.html      ← Template
├── feature-name.component.scss      ← Styles
└── feature-name.component.spec.ts   ← Tests
```

```typescript
// CORRECT: External files
@Component({
  selector: 'app-feature-name',
  templateUrl: './feature-name.component.html',
  styleUrl: './feature-name.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class FeatureNameComponent { }
```

Use `styleUrl` (singular) for a single stylesheet. Use `styleUrls` (plural array) only when multiple stylesheets are genuinely needed.

## Standalone by Default

In Angular 19, `standalone: true` is the **implicit default**. Do NOT write `standalone: true` — it is assumed. Only use `standalone: false` if wrapping third-party NgModule libraries.

`CommonModule` is no longer needed when using built-in control flow syntax (`@if`, `@for`, `@switch`). Do NOT import it unless you specifically need `NgClass`, `NgStyle`, `DatePipe`, etc.

## Signal-Based Inputs (Mandatory)

**ALWAYS** use `input()` and `input.required()` instead of the `@Input()` decorator. Signal inputs return a `Signal<T>` — read the value with `()`.

```typescript
name = input.required<string>();          // replaces @Input({ required: true }) name!: string
age = input<number>(0);                   // optional with default
count = input<number, string | number>(0, { transform: (v) => +v });  // with transform
userName = input<string>('', { alias: 'user-name' });                 // with alias
```

## Signal-Based Outputs (Mandatory)

**ALWAYS** use `output()` instead of `@Output()` + `EventEmitter`. Returns an `OutputEmitterRef<T>`.

```typescript
edit = output<string>();                  // replaces @Output() edit = new EventEmitter<string>()
removed = output<void>({ alias: 'delete' });

handleEdit() { this.edit.emit(this.name()); }
```

## Signal-Based Queries

**ALWAYS** use `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()` instead of decorator equivalents. They return signals.

```typescript
searchInput = viewChild.required<ElementRef>('searchInput');  // replaces @ViewChild
optionalRef = viewChild<ElementRef>('optionalRef');           // optional
childItems = viewChildren(ChildItemComponent);                // replaces @ViewChildren
child = contentChild(ChildComponent);                         // replaces @ContentChild
children = contentChildren(ChildComponent);                   // replaces @ContentChildren
```

No need for `AfterViewInit` — signals are reactive.

## Signals (Preferred State Management)

Use signals for reactive state management. Prefer signals over RxJS observables for component state.

```typescript
count = signal(0);
double = computed(() => this.count() * 2);

constructor() {
  effect((onCleanup) => {
    const currentCount = this.count();
    onCleanup(() => { /* cleanup before next execution or destroy */ });
  });
}

// Mutation: use .set() for replacement, .update() for transformation
this.count.set(0);
this.count.update(value => value + 1);
```

### `linkedSignal()` — Writable Derived State

Use `linkedSignal()` for state that derives a default from another signal but can be overridden. Prefer over manual `effect()` + `signal()` combinations.

```typescript
shippingOptions = signal<ShippingMethod[]>([...]);
selectedOption = linkedSignal(() => this.shippingOptions()[0]); // resets when source changes
selectedOption.set(manualOverride); // user can override

// With previous value access
selectedOption = linkedSignal<ShippingMethod[], ShippingMethod>({
  source: this.shippingOptions,
  computation: (newOptions, previous) =>
    newOptions.find(opt => opt.id === previous?.value?.id) ?? newOptions[0]
});
```

**Note**: Developer preview in v19, stable in v20.

### When to Use Signals vs RxJS

**Signals**: Component local state, derived values, simple reactive data flows.

**RxJS Observables**: HTTP requests (unless using `resource()`/`httpResource`), complex async with operators (debounce, retry), event streams with multiple subscribers, existing RxJS library integration.

Bridge with `toSignal()` and `toObservable()` from `@angular/core/rxjs-interop`.

## `@let` Template Syntax

Use `@let` to declare local template variables. Variables are scoped to the current view and descendants. Cannot be reassigned (effectively `const`).

```html
@let subtotal = calculateSubtotal(items());
@let tax = subtotal * taxRate();
<p>Total: {{ subtotal + tax | currency }}</p>

@let currentUser = user$ | async;
@if (currentUser) { <h2>Hello, {{ currentUser.name }}</h2> }
```

## Control Flow Syntax (Mandatory)

**ALWAYS** use `@if`, `@for`, `@switch`. **NEVER** use `*ngIf`, `*ngFor`, `*ngSwitch`.

### @if / @else if / @else

```html
@if (user(); as user) {
  <h2>{{ user.name }}</h2>
  @if (user.isAdmin) { <span>Admin</span> }
  @else if (user.isModerator) { <span>Moderator</span> }
  @else { <span>User</span> }
} @else {
  <p>No user selected</p>
}
```

### @for with track (CRITICAL: always provide `track`)

```html
@for (user of users(); track user.id) {
  <li>{{ user.name }} @if ($index === 0) { <span>First</span> }</li>
} @empty { <li>No users found</li> }
```

Track preference: `track item.id` (best) > `track $index` (acceptable) > `track item` (avoid — object reference).

### @switch / @case

```html
@switch (user().role) {
  @case ('admin') { <span>Admin</span> }
  @case ('moderator') { <span>Moderator</span> }
  @default { <span>Guest</span> }
}
```

## `@defer` Blocks — In-Template Lazy Loading

Use `@defer` to lazy-load heavy components. All deferred components must be standalone.

```html
@defer (on viewport) {
  <heavy-chart [data]="chartData()" />
} @placeholder { <div class="skeleton">Loading area</div> }
  @loading (minimum 500ms) { <spinner /> }
  @error { <p>Failed to load</p> }
```

Triggers: `on viewport`, `on idle`, `on interaction`, `on hover`, `on timer(ms)`, `when condition`.

## Component Architecture

### Smart (Container) vs Presentational (Dumb) Pattern

**Smart Components**: Manage state and business logic, inject services, handle routing, pass data down.

**Presentational Components**: Receive data via `input()`, emit events via `output()`, no service injection, stateless or local UI state only.

### OnPush Change Detection (Default)

**ALWAYS** use `ChangeDetectionStrategy.OnPush` for all components. Signals automatically trigger change detection.

### Zoneless Change Detection (Developer Preview)

Remove Zone.js overhead entirely. Prerequisites: all components use OnPush, all state is signal-based.

```typescript
bootstrapApplication(AppComponent, {
  providers: [provideZonelessChangeDetection()]
});
```

## Dependency Injection

### inject() Function (Mandatory)

**ALWAYS** use `inject()` instead of constructor injection:

```typescript
export class UserDashboardComponent {
  private userService = inject(UserService);
  private router = inject(Router);
  private destroyRef = inject(DestroyRef);
}

// WRONG: constructor(private userService: UserService) {}
```

### Injection Tokens

```typescript
export const API_BASE_URL = new InjectionToken<string>('api.base.url');
// Provide: { provide: API_BASE_URL, useValue: 'https://api.example.com' }
// Use: private apiUrl = inject(API_BASE_URL);
```

## Reactive Forms

Use `FormBuilder` with `ReactiveFormsModule`. Always use typed `FormControl`.

```typescript
private fb = inject(FormBuilder);
loginForm = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', [Validators.required, Validators.minLength(8)]]
});
```

Custom validators return `ValidatorFn` — apply at group level with `{ validators: myValidator() }`.

## HTTP Client Patterns

### Functional HTTP Interceptor

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next(req);
};

// Register: provideHttpClient(withInterceptors([authInterceptor]))
```

### `resource()` and `rxResource()` — Signal-Based Data Loading (Experimental)

```typescript
userResource = resource({
  params: () => {
    const id = this.userId();
    return id !== undefined ? { id } : undefined;
  },
  loader: async ({ params, abortSignal }) => {
    const response = await fetch(`/api/users/${params.id}`, { signal: abortSignal });
    return response.json() as Promise<User>;
  }
});
// Template: userResource.status() returns 'loading' | 'error' | 'resolved'
// RxJS variant: rxResource() with loader returning Observable
```

## Routing

Use standalone route configuration with lazy loading:

```typescript
export const routes: Routes = [
  { path: '', redirectTo: '/users', pathMatch: 'full' },
  { path: 'users', loadComponent: () => import('./users/users.component').then(m => m.UsersComponent) },
  { path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES), canActivate: [authGuard] },
  { path: '**', redirectTo: '/users' }
];
```

### Functional Route Guards

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  return authService.isAuthenticated() ? true : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};
```

## RxJS Patterns

### takeUntilDestroyed for Subscriptions

**ALWAYS** use `takeUntilDestroyed(this.destroyRef)` for subscriptions in components/services.

### Search Pattern

```typescript
searchControl = new FormControl('');
searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.userService.searchUsers(term || '')),
  takeUntilDestroyed(this.destroyRef)
).subscribe(users => this.users.set(users));
```

## Angular Material Integration

Import only needed component modules for tree-shaking — no wildcard imports.

## State Management

- **Local state**: Use signals in components (`signal()`, `computed()`)
- **Shared state**: Use signal-based services with `asReadonly()` for public access

```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<CartItem[]>([]);
  readonly items$ = this.items.asReadonly();
  readonly total = computed(() => this.items().reduce((sum, i) => sum + i.price * i.quantity, 0));

  addItem(item: CartItem) { this.items.update(items => [...items, item]); }
  removeItem(id: number) { this.items.update(items => items.filter(i => i.id !== id)); }
}
```

## Virtual Scrolling for Large Lists

Use `ScrollingModule` from `@angular/cdk/scrolling` with `cdk-virtual-scroll-viewport` and `*cdkVirtualFor`.

## Testing

See [typescript/testing.md](testing.md) for testing patterns.

Quick guide:
- Use `TestBed` for component and service testing
- Set signal inputs via `fixture.componentRef.setInput('name', value)`
- Test signal outputs via `component.outputName.subscribe()`
- Use component harnesses for Material components

## Angular 19 Checklist

Before committing Angular code:
- [ ] No explicit `standalone: true` (it's the default)
- [ ] No `CommonModule` imports (unless needed for `NgClass`/`NgStyle`/pipes)
- [ ] External template files (`templateUrl`) — no inline `template`
- [ ] External style files (`styleUrl`) — no inline `styles`
- [ ] `input()` / `input.required()` — no `@Input()` decorator
- [ ] `output()` — no `@Output()` + `EventEmitter`
- [ ] `viewChild()` / `contentChild()` — no `@ViewChild` / `@ContentChild` decorators
- [ ] Signals for component state
- [ ] `@if`, `@for`, `@switch` — never `*ngIf`, `*ngFor`, `*ngSwitch`
- [ ] `@for` loops include `track`
- [ ] `inject()` — no constructor injection
- [ ] `ChangeDetectionStrategy.OnPush` on all components
- [ ] `takeUntilDestroyed()` for subscriptions
- [ ] Lazy loading with `loadComponent` / `loadChildren`
- [ ] Required inputs use `input.required<T>()`
- [ ] HTTP services have `catchError` handling
- [ ] Material imports are specific (no wildcards)
- [ ] No NgModules for new components
- [ ] Reactive forms with typed `FormControl`
- [ ] `effect()` uses `onCleanup` for cleanup
