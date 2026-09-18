# 07 — Arquitectura Frontend

Angular 22 · TypeScript 5.9 · Angular Material 22 · componentes standalone

> Angular cambió bastante entre la versión 17 (la del stack original) y la 22. Los
> ejemplos de este documento usan la sintaxis actual: **sin `NgModule`**, con `@if`/`@for`
> en las plantillas y **signals** para el estado. Un tutorial con `*ngIf` y
> `standalone: true` explícito es de una versión anterior.

---

## 1. Arranque

```bash
npm install -g @angular/cli@22
ng new productionfood-frontend --routing --style=scss --ssr=false
cd productionfood-frontend
ng add @angular/material@22
```

`--ssr=false`: es una aplicación interna tras autenticación, no un sitio público. El
renderizado en servidor añade complejidad sin beneficio.

La estructura de carpetas está en `01-CONVENCIONES.md` §4.

---

## 2. Configuración raíz

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
    provideAnimationsAsync(),
    { provide: MAT_DATE_LOCALE, useValue: 'es-CO' },
    { provide: LOCALE_ID, useValue: 'es-CO' }
  ]
};
```

`LOCALE_ID = 'es-CO'` hace que los pipes `date`, `number` y `currency` usen el formato
colombiano: `$ 12.500,50` en lugar de `$12,500.50`. Sin esto, los precios se muestran con
la coma y el punto invertidos respecto a lo que el usuario espera, y nadie lo nota hasta
la sustentación.

Registrar el locale en `main.ts`:

```typescript
import { registerLocaleData } from '@angular/common';
import localeEsCO from '@angular/common/locales/es-CO';
registerLocaleData(localeEsCO);
```

---

## 3. Autenticación

### `AuthService`

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);
  private readonly router = inject(Router);

  private readonly _usuario = signal<UsuarioSesion | null>(this.leerSesion());
  readonly usuario = this._usuario.asReadonly();
  readonly autenticado = computed(() => this._usuario() !== null);
  readonly rol = computed(() => this._usuario()?.rol ?? null);

  login(correo: string, password: string): Observable<LoginResponse> {
    return this.http.post<LoginResponse>(`${API}/auth/login`, { correo, password })
      .pipe(tap(r => {
        localStorage.setItem('token', r.token);
        this._usuario.set(this.decodificar(r.token));
      }));
  }

  logout(): void {
    localStorage.removeItem('token');
    this._usuario.set(null);
    this.router.navigate(['/login']);
  }

  tieneAlgunRol(roles: Rol[]): boolean {
    const r = this.rol();
    return r !== null && roles.includes(r);
  }

  private leerSesion(): UsuarioSesion | null {
    const token = localStorage.getItem('token');
    if (!token) return null;
    const datos = this.decodificar(token);
    // Un token expirado en localStorage produciría una sesión fantasma:
    // el menú aparece, y cada petición devuelve 401.
    return datos && datos.exp * 1000 > Date.now() ? datos : null;
  }
}
```

> **Sobre `localStorage`.** El stack lo especifica y así se implementa. Conviene saber que
> un token en `localStorage` es legible por cualquier script de la página, así que una
> vulnerabilidad XSS permite robarlo. La alternativa —cookie `HttpOnly`— exige cambios en
> el backend que están fuera de este alcance. Se documenta como decisión consciente, no
> como descuido, y se compensa evitando `innerHTML` con datos del servidor.

### Interceptores

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  return token
    ? next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }))
    : next(req);
};

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const noti = inject(NotificacionService);

  return next(req).pipe(
    catchError((e: HttpErrorResponse) => {
      switch (e.status) {
        case 401:
          auth.logout();
          noti.advertencia('Su sesión expiró. Inicie sesión nuevamente.');
          break;
        case 403:
          // No se cierra la sesión: el usuario está autenticado, solo no autorizado
          noti.error(e.error?.message ?? 'No tiene permisos para esta acción.');
          break;
        case 0:
          noti.error('No se pudo conectar con el servidor.');
          break;
        case 500:
          noti.error('Ocurrió un error en el servidor. Intente nuevamente.');
          break;
        // 400, 404, 409, 422 los maneja el componente: son errores de negocio
        // que deben mostrarse junto al campo o en el formulario, no en un toast global.
      }
      return throwError(() => e);
    })
  );
};
```

La distinción entre `401` y `403` importa: cerrar la sesión ante un `403` expulsa al
usuario por pulsar un botón que no le correspondía.

### Guards

```typescript
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.autenticado() ? true : router.createUrlTree(['/login']);
};

export const rolGuard = (roles: Rol[]): CanActivateFn => () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.tieneAlgunRol(roles) ? true : router.createUrlTree(['/sin-acceso']);
};
```

---

## 4. Servicios HTTP

Uno por módulo, tipado:

```typescript
@Injectable({ providedIn: 'root' })
export class ClienteService {
  private readonly http = inject(HttpClient);
  private readonly url = `${environment.apiUrl}/clientes`;

  listar(filtro: ClienteFiltro): Observable<PageResponse<Cliente>> {
    let params = new HttpParams()
      .set('page', filtro.page)
      .set('size', filtro.size)
      .set('sort', filtro.sort);
    if (filtro.nombre) params = params.set('nombre', filtro.nombre);
    if (filtro.activo !== undefined) params = params.set('activo', filtro.activo);
    return this.http.get<PageResponse<Cliente>>(this.url, { params });
  }

  crear(dto: CrearClienteDto): Observable<Cliente> {
    return this.http.post<Cliente>(this.url, dto);
  }
}
```

Modelos espejo de los DTO del backend; nunca `any`:

```typescript
export interface PageResponse<T> {
  content: T[];
  page: number; size: number;
  totalElements: number; totalPages: number;
  first: boolean; last: boolean;
}

export interface ApiError {
  timestamp: string; status: number; error: string;
  code: string; message: string; path: string;
  fieldErrors: { field: string; message: string }[];
}
```

---

## 5. Componente de listado

```typescript
@Component({
  selector: 'app-cliente-lista',
  imports: [MatTableModule, MatPaginatorModule, MatButtonModule, ReactiveFormsModule],
  templateUrl: './cliente-lista.component.html'
})
export class ClienteListaComponent {
  private readonly service = inject(ClienteService);

  readonly cargando = signal(false);
  readonly error    = signal<string | null>(null);
  readonly datos    = signal<PageResponse<Cliente> | null>(null);

  readonly filtroNombre = new FormControl('');

  constructor() {
    // debounce: sin esto se dispara una petición por cada tecla pulsada
    this.filtroNombre.valueChanges
      .pipe(debounceTime(350), distinctUntilChanged(), takeUntilDestroyed())
      .subscribe(() => this.cargar(0));
    this.cargar(0);
  }

  cargar(page: number): void {
    this.cargando.set(true);
    this.error.set(null);
    this.service.listar({ page, size: 20, sort: 'nombre,asc', nombre: this.filtroNombre.value ?? '' })
      .subscribe({
        next: r => { this.datos.set(r); this.cargando.set(false); },
        error: (e: HttpErrorResponse) => {
          this.error.set(e.error?.message ?? 'No se pudieron cargar los clientes.');
          this.cargando.set(false);
        }
      });
  }
}
```

Plantilla con los **tres estados obligatorios** (`05-ESTANDARES-QA.md` §7):

```html
@if (cargando()) {
  <mat-progress-bar mode="indeterminate" />
} @else if (error()) {
  <div class="estado-error">
    <p>{{ error() }}</p>
    <button mat-stroked-button (click)="cargar(0)">Reintentar</button>
  </div>
} @else if (datos()?.content?.length === 0) {
  <div class="estado-vacio">
    <p>No hay clientes registrados.</p>
    <button mat-raised-button color="primary" *siRol="['ADMIN','VENTAS']" (click)="nuevo()">
      Registrar el primero
    </button>
  </div>
} @else {
  <table mat-table [dataSource]="datos()!.content">
    @for (col of columnas; track col) { <!-- ... --> }
  </table>
  <mat-paginator [length]="datos()!.totalElements"
                 [pageSize]="datos()!.size"
                 [pageIndex]="datos()!.page"
                 (page)="cargar($event.pageIndex)" />
}
```

El estado vacío con una acción sugerida evita la pantalla en blanco que hace dudar al
usuario de si la aplicación falló o si de verdad no hay datos.

---

## 6. Formularios y errores del servidor

```typescript
export class ClienteFormularioComponent {
  readonly form = inject(FormBuilder).group({
    nombre:   ['', [Validators.required, Validators.maxLength(100)]],
    contacto: ['', Validators.maxLength(100)],
    telefono: ['', Validators.maxLength(20)]
  });

  readonly guardando = signal(false);

  guardar(): void {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    this.guardando.set(true);                      // deshabilita el botón: evita doble envío

    this.service.crear(this.form.getRawValue() as CrearClienteDto).subscribe({
      next: () => { this.noti.exito('Cliente registrado.'); this.cerrar(true); },
      error: (e: HttpErrorResponse) => {
        this.guardando.set(false);
        this.aplicarErroresDelServidor(e.error);
      }
    });
  }

  /** Coloca cada fieldError del contrato en su control correspondiente. */
  private aplicarErroresDelServidor(err: ApiError): void {
    for (const fe of err.fieldErrors ?? []) {
      this.form.get(fe.field)?.setErrors({ servidor: fe.message });
    }
    if (!err.fieldErrors?.length) this.noti.error(err.message);
  }
}
```

Mostrar el error del servidor junto al campo —y no solo en un aviso flotante— es lo que
permite al usuario corregir sin adivinar cuál de los cinco campos estaba mal.

---

## 7. Directiva de rol

```typescript
@Directive({ selector: '[siRol]' })
export class SiRolDirective {
  private readonly auth = inject(AuthService);
  private readonly vista = inject(ViewContainerRef);
  private readonly plantilla = inject(TemplateRef<unknown>);

  @Input() set siRol(roles: Rol[]) {
    this.vista.clear();
    if (this.auth.tieneAlgunRol(roles)) this.vista.createEmbeddedView(this.plantilla);
  }
}
```

> Esto es **cosmética, no seguridad**. Ocultar un botón mejora la experiencia; cualquiera
> puede llamar el endpoint con Postman. La autorización real está en el backend
> (`03-MATRIZ-ROLES.md` §5).

---

## 8. Rutas

```typescript
export const routes: Routes = [
  { path: 'login', loadComponent: () => import('./features/auth/login.component').then(m => m.LoginComponent) },
  {
    path: '',
    canActivate: [authGuard],
    loadComponent: () => import('./layout/layout.component').then(m => m.LayoutComponent),
    children: [
      { path: 'dashboard', loadChildren: () => import('./features/dashboard/dashboard.routes') },
      { path: 'clientes',  canActivate: [rolGuard(['ADMIN','VENTAS','PRODUCCION','CONSULTA'])],
                           loadChildren: () => import('./features/cliente/cliente.routes') },
      { path: 'usuarios',  canActivate: [rolGuard(['ADMIN'])],
                           loadChildren: () => import('./features/usuario/usuario.routes') },
      { path: '', redirectTo: 'dashboard', pathMatch: 'full' }
    ]
  },
  { path: 'sin-acceso', loadComponent: () => import('./shared/sin-acceso.component').then(m => m.SinAccesoComponent) },
  { path: '**', redirectTo: '' }
];
```

Carga diferida por módulo: el usuario de VENTAS no descarga el código de producción.

---

## 9. Formato de cantidades y precios

```html
{{ producto.precio | currency:'COP':'symbol-narrow':'1.2-2' }}   <!-- $ 12.500,00 -->
{{ inventario.cantidadDisponible | number:'1.2-2' }} {{ unidad }}
```

Con `LOCALE_ID = 'es-CO'`, el separador de miles es el punto y el decimal la coma.

**Los totales no se calculan en el frontend.** El total de un pedido o de una compra lo
devuelve el backend. Si el cliente suma por su cuenta, dos capas calculan lo mismo con
reglas de redondeo distintas y aparecen diferencias de centavos imposibles de explicar.
El frontend muestra; el backend calcula.

---

## 10. Entornos

```typescript
// environments/environment.ts
export const environment = { production: false, apiUrl: 'http://localhost:8080/api/v1' };

// environments/environment.prod.ts
export const environment = { production: true, apiUrl: '/api/v1' };
```

En producción se usa ruta relativa: el frontend se sirve desde el mismo origen que la API y
desaparece el problema de CORS.
