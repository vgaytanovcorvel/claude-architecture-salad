# TypeScript - Angular 19+ Rules

> This file extends [typescript/coding-style.md](coding-style.md) with Angular 19+ specifics.

---
paths: ["**/*.ts", "**/*.tsx", "**/*.component.ts", "**/*.service.ts", "**/*.component.html"]
---

## Component File Structure (Mandatory)

**ALWAYS** use external template and style files. **NEVER** use inline `template` or `styles`.

Every component must have separate files:

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

// WRONG: Inline template
@Component({
  selector: 'app-feature-name',
  template: `<p>Don't do this</p>`,       // NEVER inline template
  styles: [`p { color: red; }`]            // NEVER inline styles
})
export class FeatureNameComponent { }
```

Use `styleUrl` (singular) for a single stylesheet. Use `styleUrls` (plural array) only when multiple stylesheets are genuinely needed.

## Standalone by Default

In Angular 19, `standalone: true` is the **implicit default** for all components, directives, and pipes. Do NOT write `standalone: true` — it is assumed. Only use `standalone: false` if wrapping third-party NgModule libraries.

`CommonModule` is no longer needed when using the built-in control flow syntax (`@if`, `@for`, `@switch`). Do NOT import it unless you specifically need `NgClass`, `NgStyle`, `DatePipe`, etc.

```typescript
// user-card.component.ts
import { Component, ChangeDetectionStrategy, input, output } from '@angular/core';
import { MatButtonModule } from '@angular/material/button';
import { MatCardModule } from '@angular/material/card';

@Component({
  selector: 'app-user-card',
  // No standalone: true needed — it is the default
  // No CommonModule needed — @if/@for are built-in
  imports: [
    MatButtonModule,
    MatCardModule
  ],
  templateUrl: './user-card.component.html',
  styleUrl: './user-card.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent {
  user = input.required<User>();
  edit = output<User>();
}
```

```html
<!-- user-card.component.html -->
<mat-card>
  <mat-card-header>
    <h2>{{ user().name }}</h2>
  </mat-card-header>
  <mat-card-content>
    <p>{{ user().email }}</p>
  </mat-card-content>
  <mat-card-actions>
    <button mat-button (click)="edit.emit(user())">Edit</button>
  </mat-card-actions>
</mat-card>
```

## Signal-Based Inputs (Mandatory)

**ALWAYS** use `input()` and `input.required()` instead of the `@Input()` decorator. Signal inputs return a `Signal<T>` — read the value with `()`.

```typescript
// user-age.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-age',
  templateUrl: './user-age.component.html',
  styleUrl: './user-age.component.scss'
})
export class UserAgeComponent {
  // Required input (replaces @Input({ required: true }) name!: string)
  name = input.required<string>();

  // Optional input with default (replaces @Input() age: number = 0)
  age = input<number>(0);

  // Input with transform
  count = input<number, string | number>(0, {
    transform: (value: string | number) => +value
  });

  // Input with alias
  userName = input<string>('', { alias: 'user-name' });
}

// WRONG: Decorator-based input
export class BadComponent {
  @Input({ required: true }) name!: string;  // Don't use @Input()
  @Input() age: number = 0;                  // Don't use @Input()
}
```

```html
<!-- user-age.component.html -->
<p>Age: {{ age() }}</p>
```

## Signal-Based Outputs (Mandatory)

**ALWAYS** use `output()` instead of `@Output()` + `EventEmitter`. The `output()` function returns an `OutputEmitterRef<T>`.

```typescript
// user-card.component.ts
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  styleUrl: './user-card.component.scss'
})
export class UserCardComponent {
  name = input.required<string>();

  // Replaces @Output() edit = new EventEmitter<string>()
  edit = output<string>();

  // Output with alias
  removed = output<void>({ alias: 'delete' });

  handleEdit() {
    this.edit.emit(this.name());
  }

  handleDelete() {
    this.removed.emit();
  }
}

// WRONG: Decorator-based output
export class BadComponent {
  @Output() edit = new EventEmitter<string>();  // Don't use @Output()
}
```

```html
<!-- user-card.component.html -->
<button (click)="handleEdit()">Edit</button>
<button (click)="handleDelete()">Delete</button>
```

## Signal-Based Queries

**ALWAYS** use `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()` instead of the decorator equivalents. They return signals.

```typescript
// parent.component.ts
import { Component, viewChild, viewChildren, contentChild, contentChildren, ElementRef, input } from '@angular/core';

@Component({
  selector: 'app-parent',
  templateUrl: './parent.component.html',
  styleUrl: './parent.component.scss'
})
export class ParentComponent {
  items = input.required<Item[]>();

  // Replaces @ViewChild('searchInput') searchInput!: ElementRef
  searchInput = viewChild.required<ElementRef>('searchInput');

  // Optional viewChild (returns Signal<ElementRef | undefined>)
  optionalRef = viewChild<ElementRef>('optionalRef');

  // Replaces @ViewChildren(ChildItemComponent)
  childItems = viewChildren(ChildItemComponent);

  // No need for AfterViewInit — signals are reactive
  focusSearch() {
    this.searchInput().nativeElement.focus();
  }
}
```

```html
<!-- parent.component.html -->
<input #searchInput />
@for (item of items(); track item.id) {
  <app-child-item [data]="item" />
}
```

```typescript
// wrapper.component.ts
@Component({
  selector: 'app-wrapper',
  templateUrl: './wrapper.component.html',
  styleUrl: './wrapper.component.scss'
})
export class WrapperComponent {
  child = contentChild(ChildComponent);
  children = contentChildren(ChildComponent);
}
```

```html
<!-- wrapper.component.html -->
<ng-content />
```

## Signals (Preferred State Management)

Use signals for reactive state management. Prefer signals over RxJS observables for component state.

### Signal Basics

```typescript
// counter.component.ts
import { Component, signal, computed, effect, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-counter',
  templateUrl: './counter.component.html',
  styleUrl: './counter.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CounterComponent {
  count = signal(0);

  double = computed(() => this.count() * 2);

  constructor() {
    effect((onCleanup) => {
      const currentCount = this.count();
      console.log(`Count changed to: ${currentCount}`);

      // Register cleanup that runs before next execution or on destroy
      onCleanup(() => {
        console.log(`Cleaning up for count: ${currentCount}`);
      });
    });
  }

  increment() {
    this.count.update(value => value + 1);
  }

  decrement() {
    this.count.update(value => value - 1);
  }

  reset() {
    this.count.set(0);
  }
}
```

```html
<!-- counter.component.html -->
<div>
  <p>Count: {{ count() }}</p>
  <p>Double: {{ double() }}</p>
  <button (click)="increment()">Increment</button>
  <button (click)="decrement()">Decrement</button>
  <button (click)="reset()">Reset</button>
</div>
```

### `linkedSignal()` — Writable Derived State

Use `linkedSignal()` for state that derives a default from another signal but can be overridden by the user. Prefer over manual `effect()` + `signal()` combinations for this pattern.

**Note**: Developer preview in v19, stable in v20.

```typescript
import { signal, linkedSignal } from '@angular/core';

@Component({
  selector: 'app-shipping',
  templateUrl: './shipping.component.html',
  styleUrl: './shipping.component.scss'
})
export class ShippingComponent {
  shippingOptions = signal<ShippingMethod[]>([
    { id: 1, name: 'Standard', price: 5 },
    { id: 2, name: 'Express', price: 15 }
  ]);

  // Automatically selects first option; resets when options change
  selectedOption = linkedSignal(() => this.shippingOptions()[0]);

  // User can manually override
  selectOption(option: ShippingMethod) {
    this.selectedOption.set(option);
  }
}

// With previous value access
selectedOption = linkedSignal<ShippingMethod[], ShippingMethod>({
  source: this.shippingOptions,
  computation: (newOptions, previous) => {
    const prev = previous?.value;
    return newOptions.find(opt => opt.id === prev?.id) ?? newOptions[0];
  }
});
```

### When to Use Signals vs RxJS

**Use Signals for:**
- Component local state
- Derived values that depend on other state
- Simple reactive data flows
- Values that change over time but don't need operators

**Use RxJS Observables for:**
- HTTP requests (unless using `resource()` / `httpResource`)
- Complex async operations with operators (debounce, retry, etc.)
- Event streams with multiple subscribers
- Integration with existing RxJS-based libraries

**Example: Combining Signals and RxJS**

```typescript
// user-search.component.ts
import { Component, signal, inject, ChangeDetectionStrategy } from '@angular/core';
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { debounceTime, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-user-search',
  templateUrl: './user-search.component.html',
  styleUrl: './user-search.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserSearchComponent {
  private userService = inject(UserService);

  searchTerm = signal('');

  private searchTerm$ = toObservable(this.searchTerm);

  private users$ = this.searchTerm$.pipe(
    debounceTime(300),
    switchMap(term => this.userService.searchUsers(term))
  );

  users = toSignal(this.users$, { initialValue: [] });

  loading = signal(false);

  onSearchChange(event: Event) {
    const term = (event.target as HTMLInputElement).value;
    this.searchTerm.set(term);
  }
}
```

```html
<!-- user-search.component.html -->
<input [value]="searchTerm()" (input)="onSearchChange($event)" />

@if (loading()) {
  <p>Loading...</p>
}

@if (users(); as users) {
  <ul>
    @for (user of users; track user.id) {
      <li>{{ user.name }}</li>
    }
  </ul>
}
```

## `@let` Template Syntax

Use `@let` to declare local template variables. Reduces duplication and improves readability.

```html
<!-- invoice.component.html -->
@let subtotal = calculateSubtotal(items());
@let tax = subtotal * taxRate();
@let total = subtotal + tax;

<p>Subtotal: {{ subtotal | currency }}</p>
<p>Tax: {{ tax | currency }}</p>
<p>Total: {{ total | currency }}</p>

@let currentUser = user$ | async;
@if (currentUser) {
  <h2>Hello, {{ currentUser.name }}</h2>
}
```

Scoping rules:
- `@let` variables are scoped to the current view and its descendants
- Available **after** the declaration in the template
- **Cannot be reassigned** — effectively `const` after declaration
- Automatically kept up-to-date with the given expression

## Control Flow Syntax (Mandatory)

**ALWAYS** use `@if`, `@for`, and `@switch`. NEVER use `*ngIf`, `*ngFor`, or `*ngSwitch`.

### @if / @else if / @else

```typescript
// user-status.component.ts
@Component({
  selector: 'app-user-status',
  templateUrl: './user-status.component.html',
  styleUrl: './user-status.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserStatusComponent {
  user = signal<User | null>(null);
}
```

```html
<!-- user-status.component.html -->
@if (user(); as user) {
  <div>
    <h2>{{ user.name }}</h2>

    @if (user.isAdmin) {
      <span class="badge-admin">Admin</span>
    } @else if (user.isModerator) {
      <span class="badge-moderator">Moderator</span>
    } @else {
      <span class="badge-user">User</span>
    }
  </div>
} @else {
  <p>No user selected</p>
}
```

### @for with track

**CRITICAL:** Always provide a `track` function for `@for` to optimize rendering.

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  styleUrl: './user-list.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  users = input<User[]>([]);
}
```

```html
<!-- user-list.component.html -->
<ul>
  @for (user of users(); track user.id) {
    <li>
      {{ user.name }} - {{ user.email }}

      @if ($index === 0) {
        <span class="badge">First</span>
      }
    </li>
  } @empty {
    <li>No users found</li>
  }
</ul>
```

```
Track function examples:
  track user.id        ← Best: unique identifier
  track $index         ← Acceptable: if no unique ID
  track user           ← Avoid: tracks by object reference
```

### @switch / @case

```typescript
// user-role-badge.component.ts
@Component({
  selector: 'app-user-role-badge',
  templateUrl: './user-role-badge.component.html',
  styleUrl: './user-role-badge.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserRoleBadgeComponent {
  user = input.required<User>();
}
```

```html
<!-- user-role-badge.component.html -->
@switch (user().role) {
  @case ('admin') {
    <span class="badge-admin">Admin</span>
  }
  @case ('moderator') {
    <span class="badge-moderator">Moderator</span>
  }
  @case ('user') {
    <span class="badge-user">User</span>
  }
  @default {
    <span class="badge-guest">Guest</span>
  }
}
```

## `@defer` Blocks — In-Template Lazy Loading

Use `@defer` to lazy-load heavy components that are not immediately needed. All deferred components must be standalone.

```typescript
// dashboard.component.ts
@Component({
  selector: 'app-dashboard',
  templateUrl: './dashboard.component.html',
  styleUrl: './dashboard.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class DashboardComponent {
  chartData = input.required<number[]>();
}
```

```html
<!-- dashboard.component.html -->
<h1>Dashboard</h1>

@defer (on viewport) {
  <heavy-chart-component [data]="chartData()" />
} @placeholder {
  <div class="skeleton">Chart loading area</div>
} @loading (minimum 500ms) {
  <spinner />
} @error {
  <p>Failed to load chart</p>
}
```

Trigger options: `on viewport`, `on idle`, `on interaction`, `on hover`, `on timer(ms)`, `when condition`.

## Component Architecture

### Smart (Container) vs Presentational (Dumb) Pattern

**Smart Components:**
- Manage state and business logic
- Inject services
- Handle routing and navigation
- Pass data to presentational components

**Presentational Components:**
- Receive data via `input()`
- Emit events via `output()`
- No service injection
- Stateless or local UI state only

**Example: Smart Container**

```typescript
// users-container.component.ts
import { Component, signal, inject, ChangeDetectionStrategy, DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { Router } from '@angular/router';
import { UserListComponent } from './user-list.component';
import { UserFormComponent } from './user-form.component';

@Component({
  selector: 'app-users-container',
  imports: [UserListComponent, UserFormComponent],
  templateUrl: './users-container.component.html',
  styleUrl: './users-container.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UsersContainerComponent {
  private userService = inject(UserService);
  private router = inject(Router);
  private destroyRef = inject(DestroyRef);

  users = signal<User[]>([]);

  ngOnInit() {
    this.loadUsers();
  }

  private loadUsers() {
    this.userService.getUsers()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(users => {
        this.users.set(users);
      });
  }

  onUserCreated(user: User) {
    this.userService.createUser(user)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(() => {
        this.loadUsers();
      });
  }

  onUserSelected(user: User) {
    this.router.navigate(['/users', user.id]);
  }
}
```

```html
<!-- users-container.component.html -->
<div class="users-page">
  <app-user-form (userCreated)="onUserCreated($event)" />
  <app-user-list
    [users]="users()"
    (userSelected)="onUserSelected($event)"
  />
</div>
```

**Example: Presentational Component**

```typescript
// user-list.component.ts
import { Component, input, output, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  styleUrl: './user-list.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  users = input.required<User[]>();
  userSelected = output<User>();
}
```

```html
<!-- user-list.component.html -->
<ul>
  @for (user of users(); track user.id) {
    <li (click)="userSelected.emit(user)">
      {{ user.name }}
    </li>
  }
</ul>
```

### OnPush Change Detection (Default)

**ALWAYS** use `ChangeDetectionStrategy.OnPush` for all components. Signals automatically trigger change detection on components that read them.

```typescript
// user-profile.component.ts
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html',
  styleUrl: './user-profile.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserProfileComponent {
  user = input.required<User>();
}
```

```html
<!-- user-profile.component.html -->
<h2>{{ user().name }}</h2>
```

### Zoneless Change Detection (Developer Preview)

Zoneless Angular removes the Zone.js overhead entirely. It is **developer preview in v19** — suitable for new projects, not yet recommended for production migration of existing apps.

When ready to adopt:

```typescript
import { provideZonelessChangeDetection } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection()
  ]
});
```

Prerequisites:
- All components use `OnPush`
- All state is signal-based
- Remove `zone.js` from `angular.json` polyfills

## Dependency Injection

### inject() Function (Mandatory)

**ALWAYS** use `inject()` instead of constructor injection:

```typescript
// user-dashboard.component.ts
import { Component, inject, signal, DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-user-dashboard',
  templateUrl: './user-dashboard.component.html',
  styleUrl: './user-dashboard.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserDashboardComponent {
  private userService = inject(UserService);
  private router = inject(Router);
  private route = inject(ActivatedRoute);
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.userService.getUsers()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(users => {
        // Handle users
      });
  }
}

// WRONG: Constructor injection
export class BadComponent {
  constructor(
    private userService: UserService,  // Don't use constructor injection
    private router: Router
  ) {}
}
```

### Service with inject()

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = '/api/users';

  getUsers() {
    return this.http.get<User[]>(this.apiUrl);
  }

  getUserById(id: number) {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: CreateUserRequest) {
    return this.http.post<User>(this.apiUrl, user);
  }

  updateUser(id: number, user: UpdateUserRequest) {
    return this.http.put<User>(`${this.apiUrl}/${id}`, user);
  }

  deleteUser(id: number) {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

### Injection Tokens

```typescript
import { InjectionToken } from '@angular/core';

export const API_BASE_URL = new InjectionToken<string>('api.base.url');

// Provide in main.ts or component
bootstrapApplication(AppComponent, {
  providers: [
    { provide: API_BASE_URL, useValue: 'https://api.example.com' }
  ]
});

// Use in service
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = inject(API_BASE_URL);

  getUsers() {
    return this.http.get<User[]>(`${this.apiUrl}/users`);
  }
}
```

## Reactive Forms

```typescript
// login-form.component.ts
import { Component, inject, ChangeDetectionStrategy } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-login-form',
  imports: [ReactiveFormsModule],
  templateUrl: './login-form.component.html',
  styleUrl: './login-form.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class LoginFormComponent {
  private fb = inject(FormBuilder);

  loginForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]]
  });

  get email() {
    return this.loginForm.get('email')!;
  }

  get password() {
    return this.loginForm.get('password')!;
  }

  onSubmit() {
    if (this.loginForm.valid) {
      console.log(this.loginForm.value);
    }
  }
}
```

```html
<!-- login-form.component.html -->
<form [formGroup]="loginForm" (ngSubmit)="onSubmit()">
  <div>
    <label for="email">Email</label>
    <input id="email" type="email" formControlName="email" />

    @if (email.invalid && email.touched) {
      <div class="error">
        @if (email.errors?.['required']) {
          Email is required
        }
        @if (email.errors?.['email']) {
          Invalid email format
        }
      </div>
    }
  </div>

  <div>
    <label for="password">Password</label>
    <input id="password" type="password" formControlName="password" />

    @if (password.invalid && password.touched) {
      <div class="error">
        @if (password.errors?.['required']) {
          Password is required
        }
        @if (password.errors?.['minlength']) {
          Password must be at least 8 characters
        }
      </div>
    }
  </div>

  <button type="submit" [disabled]="loginForm.invalid">Login</button>
</form>
```

### Custom Validators

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function passwordMatchValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const password = control.get('password');
    const confirmPassword = control.get('confirmPassword');

    if (!password || !confirmPassword) {
      return null;
    }

    return password.value === confirmPassword.value
      ? null
      : { passwordMismatch: true };
  };
}

// Usage
this.registerForm = this.fb.group({
  password: ['', [Validators.required, Validators.minLength(8)]],
  confirmPassword: ['', Validators.required]
}, { validators: passwordMatchValidator() });
```

## HTTP Client Patterns

### Service with Error Handling

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { catchError, retry, throwError } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class UserApiService {
  private http = inject(HttpClient);
  private apiUrl = '/api/users';

  getUsers() {
    return this.http.get<ApiResponse<User[]>>(this.apiUrl).pipe(
      retry(2),
      catchError(this.handleError)
    );
  }

  getUserById(id: number) {
    return this.http.get<ApiResponse<User>>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  createUser(user: CreateUserRequest) {
    return this.http.post<ApiResponse<User>>(this.apiUrl, user).pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse) {
    let errorMessage = 'An unknown error occurred';

    if (error.error instanceof ErrorEvent) {
      errorMessage = `Error: ${error.error.message}`;
    } else {
      errorMessage = `Error Code: ${error.status}\nMessage: ${error.message}`;
    }

    console.error(errorMessage);
    return throwError(() => new Error(errorMessage));
  }
}
```

### Functional HTTP Interceptor

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();

  if (token) {
    req = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }

  return next(req);
};

// Register in app.config.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor])
    )
  ]
});
```

### `resource()` and `rxResource()` — Signal-Based Data Loading (Experimental)

For reactive data fetching tied to signal parameters. Experimental in v19 — API may change.

```typescript
// user-detail.component.ts
import { Component, resource, signal, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-user-detail',
  templateUrl: './user-detail.component.html',
  styleUrl: './user-detail.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserDetailComponent {
  userId = signal<number | undefined>(undefined);

  userResource = resource({
    params: () => {
      const id = this.userId();
      return id !== undefined ? { id } : undefined;
    },
    loader: async ({ params, abortSignal }) => {
      const response = await fetch(`/api/users/${params.id}`, {
        signal: abortSignal
      });
      return response.json() as Promise<User>;
    }
  });
}
```

```html
<!-- user-detail.component.html -->
@switch (userResource.status()) {
  @case ('loading') { <spinner /> }
  @case ('error') { <p>Error loading user</p> }
  @case ('resolved') {
    @if (userResource.value(); as user) {
      <h2>{{ user.name }}</h2>
    }
  }
}
```

Observable-based variant with `rxResource()`:

```typescript
import { rxResource } from '@angular/core/rxjs-interop';

userResource = rxResource({
  params: () => {
    const id = this.userId();
    return id !== undefined ? { id } : undefined;
  },
  loader: ({ params }) => {
    return this.http.get<User>(`/api/users/${params.id}`);
  }
});
```

## Routing

### Standalone Route Configuration

```typescript
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: '/users', pathMatch: 'full' },
  {
    path: 'users',
    loadComponent: () => import('./users/users.component').then(m => m.UsersComponent)
  },
  {
    path: 'users/:id',
    loadComponent: () => import('./users/user-detail.component').then(m => m.UserDetailComponent)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
    canActivate: [authGuard]
  },
  { path: '**', redirectTo: '/users' }
];

// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes)
  ]
});
```

### Functional Route Guards

```typescript
import { inject } from '@angular/core';
import { Router, CanActivateFn } from '@angular/router';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};
```

## RxJS Patterns

### takeUntilDestroyed for Subscriptions

```typescript
// user-dashboard.component.ts
import { Component, inject, DestroyRef, signal, ChangeDetectionStrategy } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-user-dashboard',
  templateUrl: './user-dashboard.component.html',
  styleUrl: './user-dashboard.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserDashboardComponent {
  private userService = inject(UserService);
  private destroyRef = inject(DestroyRef);

  users = signal<User[]>([]);

  ngOnInit() {
    this.userService.getUsers()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(users => {
        this.users.set(users);
      });
  }
}
```

### Search with debounceTime + switchMap

```typescript
// user-search.component.ts
import { Component, inject, signal, DestroyRef, ChangeDetectionStrategy } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-user-search',
  imports: [ReactiveFormsModule],
  templateUrl: './user-search.component.html',
  styleUrl: './user-search.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserSearchComponent {
  private userService = inject(UserService);
  private destroyRef = inject(DestroyRef);

  searchControl = new FormControl('');
  users = signal<User[]>([]);

  ngOnInit() {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => this.userService.searchUsers(term || '')),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(users => {
      this.users.set(users);
    });
  }
}
```

```html
<!-- user-search.component.html -->
<input [formControl]="searchControl" placeholder="Search users..." />

<ul>
  @for (user of users(); track user.id) {
    <li>{{ user.name }}</li>
  }
</ul>
```

## Angular Material Integration

**Import only needed components** for tree-shaking:

```typescript
// user-table.component.ts
import { Component, input, ChangeDetectionStrategy } from '@angular/core';
import { MatTableModule } from '@angular/material/table';
import { MatSortModule } from '@angular/material/sort';
import { MatButtonModule } from '@angular/material/button';

@Component({
  selector: 'app-user-table',
  imports: [
    MatTableModule,
    MatSortModule,
    MatButtonModule
  ],
  templateUrl: './user-table.component.html',
  styleUrl: './user-table.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserTableComponent {
  users = input.required<User[]>();
  displayedColumns = ['name', 'email'];
}
```

```html
<!-- user-table.component.html -->
<table mat-table [dataSource]="users()" matSort>
  <ng-container matColumnDef="name">
    <th mat-header-cell *matHeaderCellDef mat-sort-header>Name</th>
    <td mat-cell *matCellDef="let user">{{ user.name }}</td>
  </ng-container>

  <ng-container matColumnDef="email">
    <th mat-header-cell *matHeaderCellDef mat-sort-header>Email</th>
    <td mat-cell *matCellDef="let user">{{ user.email }}</td>
  </ng-container>

  <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
  <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>
</table>
```

## State Management

### Signals for Local State (Default)

```typescript
// shopping-cart.component.ts
import { Component, signal, computed, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-shopping-cart',
  templateUrl: './shopping-cart.component.html',
  styleUrl: './shopping-cart.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ShoppingCartComponent {
  items = signal<CartItem[]>([]);
  total = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  addItem(item: CartItem) {
    this.items.update(items => [...items, item]);
  }

  removeItem(itemId: number) {
    this.items.update(items => items.filter(i => i.id !== itemId));
  }
}
```

### Service with Signals for Shared State

```typescript
import { Injectable, signal, computed } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class CartService {
  private items = signal<CartItem[]>([]);

  readonly items$ = this.items.asReadonly();

  readonly total = computed(() =>
    this.items().reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  addItem(item: CartItem) {
    this.items.update(items => [...items, item]);
  }

  removeItem(itemId: number) {
    this.items.update(items => items.filter(i => i.id !== itemId));
  }

  clear() {
    this.items.set([]);
  }
}
```

## Performance Optimization

### Virtual Scrolling for Large Lists

```typescript
// virtual-list.component.ts
import { Component, input, ChangeDetectionStrategy } from '@angular/core';
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  selector: 'app-virtual-list',
  imports: [ScrollingModule],
  templateUrl: './virtual-list.component.html',
  styleUrl: './virtual-list.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class VirtualListComponent {
  items = input.required<{ id: number; name: string }[]>();

  trackById(index: number, item: { id: number }) {
    return item.id;
  }
}
```

```html
<!-- virtual-list.component.html -->
<cdk-virtual-scroll-viewport itemSize="50" style="height: 400px;">
  <div *cdkVirtualFor="let item of items(); trackBy: trackById" class="item">
    {{ item.name }}
  </div>
</cdk-virtual-scroll-viewport>
```

## Testing Considerations

See [typescript/testing.md](testing.md) for comprehensive testing patterns.

**Quick Guide:**
- Use `TestBed` for component and service testing
- Use component harnesses for Material components
- Test signal inputs by setting them via `componentRef.setInput()`
- Test signal outputs by subscribing to `component.outputName.subscribe()`
- Mock services with `jasmine.createSpyObj` or custom mocks

```typescript
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserListComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
  });

  it('should display users', () => {
    // Set signal input via componentRef
    fixture.componentRef.setInput('users', [
      { id: 1, name: 'John', email: 'john@example.com' },
      { id: 2, name: 'Jane', email: 'jane@example.com' }
    ]);

    fixture.detectChanges();

    const items = fixture.nativeElement.querySelectorAll('li');
    expect(items.length).toBe(2);
  });

  it('should emit userSelected on click', () => {
    const user = { id: 1, name: 'John', email: 'john@example.com' };
    fixture.componentRef.setInput('users', [user]);
    fixture.detectChanges();

    let selectedUser: User | undefined;
    component.userSelected.subscribe(u => selectedUser = u);

    fixture.nativeElement.querySelector('li').click();
    expect(selectedUser).toEqual(user);
  });
});
```

## Angular 19 Development Checklist

Before committing Angular code:
- [ ] No explicit `standalone: true` (it is the default — remove it)
- [ ] No `CommonModule` imports (unless specifically needed for `NgClass`/`NgStyle`/pipes)
- [ ] **External template files** (`templateUrl`) — no inline `template`
- [ ] **External style files** (`styleUrl`) — no inline `styles`
- [ ] `input()` / `input.required()` used instead of `@Input()` decorator
- [ ] `output()` used instead of `@Output()` + `EventEmitter`
- [ ] `viewChild()` / `contentChild()` used instead of `@ViewChild` / `@ContentChild` decorators
- [ ] Signals used for component state instead of plain properties
- [ ] New control flow syntax (`@if`, `@for`, `@switch`) used — never `*ngIf`, `*ngFor`, `*ngSwitch`
- [ ] `@for` loops include `track` function
- [ ] `inject()` function used instead of constructor injection
- [ ] `ChangeDetectionStrategy.OnPush` set on all components
- [ ] `takeUntilDestroyed()` used for subscriptions
- [ ] Lazy loading with `loadComponent` / `loadChildren` for routes
- [ ] Required inputs marked with `input.required<T>()`
- [ ] HTTP services have error handling with `catchError`
- [ ] Material imports are specific (no wildcard imports)
- [ ] No NgModules for new components
- [ ] Forms use reactive forms with typed `FormControl`
- [ ] `effect()` uses `onCleanup` for cleanup logic
