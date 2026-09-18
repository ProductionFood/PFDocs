# 03 — Matriz de Roles y Permisos

Las historias de usuario hablan de cinco perfiles distintos: *administrador*, *usuario*,
*comprador*, *vendedor* y *productor*. El PDF de Entrega 1 define solo tres actores. Se
adopta el modelo de **cinco roles**, que es el que las historias realmente exigen.
La diferencia está registrada en `08-DESVIACIONES-PDF.md`.

---

## 1. Los cinco roles

| id | Código | Nombre visible | Quién es | HU que lo mencionan |
|---|---|---|---|---|
| 1 | `ADMIN` | Administrador | Configura el sistema y sus catálogos | HU-01, 02, 05, 06, 07, 08, 09, 10, 11, 12, 19, 23, 24, 25, 26, 27 |
| 2 | `PRODUCCION` | Producción | Planifica y ejecuta la fabricación | HU-20, 21, 22 |
| 3 | `COMPRAS` | Compras | Adquiere materia prima | HU-13, 14, 15 |
| 4 | `VENTAS` | Ventas | Atiende clientes y pedidos | HU-16, 17, 18 |
| 5 | `CONSULTA` | Consulta | Solo lectura | PDF §6 |

Un usuario tiene **exactamente un rol** (`usuarios.id_rol` es una FK simple, no hay tabla
puente). Si más adelante se requieren roles múltiples, es un cambio de esquema, no una
configuración.

---

## 2. Principios

1. **`ADMIN` puede todo.** Es el superusuario del sistema; no se le restringe nada.
2. **`CONSULTA` solo lee.** Ningún `POST`, `PUT`, `PATCH` ni `DELETE`, en ningún módulo.
3. **Los roles operativos leen lo que necesitan para trabajar.** `VENTAS` necesita ver el
   catálogo de productos para armar un pedido, aunque no pueda editarlo. `PRODUCCION`
   necesita ver materias primas y recetas. Restringir la lectura transversal rompería los
   flujos sin ganar seguridad real.
4. **Escribir es lo que se restringe.** Cada rol escribe únicamente en su dominio.
5. **Nadie ve la bitácora salvo `ADMIN`.** Es el registro de auditoría; que el auditado
   pueda consultarlo íntegro no tiene sentido.

---

## 3. Matriz por módulo

`L` = leer · `E` = escribir (crear/editar) · `—` = sin acceso

| Módulo | ADMIN | PRODUCCION | COMPRAS | VENTAS | CONSULTA |
|---|---|---|---|---|---|
| Usuarios y roles | L E | — | — | — | — |
| Bitácora | L | — | — | — | — |
| Clientes | L E | L | — | **L E** | L |
| Proveedores | L E | — | **L E** | — | L |
| Unidades de medida | L E | L | L | L | L |
| Materias primas | L E | L | L | — | L |
| Lotes de materia prima | L E | **L E** | **L E** | — | L |
| Inventario de MP (consulta) | L | L | L | — | L |
| Productos | L E | L | — | L | L |
| Recetas | L E | **L E** | — | — | L |
| Compras y su detalle | L E | L | **L E** | — | L |
| Recepción de compra | L E | — | **L E** | — | — |
| Pedidos y su detalle | L E | L | — | **L E** | L |
| Devoluciones | L E | — | — | **L E** | L |
| Aprobar devolución | L E | — | — | **L E** | — |
| Lotes de producto terminado | L E | **L E** | — | L | L |
| Planes de producción | L E | **L E** | — | — | L |
| Consumo de materia prima | L E | **L E** | — | — | L |
| Dashboard | L | L | L | L | L |
| Reportes | L | L | L | L | L |

**En negrita**, el dominio propio de cada rol.

### Decisiones que merecen justificación

- **`PRODUCCION` escribe lotes de materia prima.** Producción es quien físicamente recibe
  y abre los sacos de harina; obligar a que Compras registre cada lote genera un cuello
  de botella operativo.
- **`PRODUCCION` escribe lotes de producto terminado.** El lote de pan nace en el horno,
  no en la oficina. HU-19 lo atribuye a "administrador", pero operativamente lo registra
  quien produce; `ADMIN` conserva el permiso.
- **`VENTAS` aprueba devoluciones.** HU-18 CA-07 hace que aprobar devuelva stock al
  inventario — una operación con efecto material. Se consideró reservarla a `ADMIN`, pero
  la devolución la recibe quien atiende al cliente. Queda en `VENTAS` **con registro
  obligatorio en bitácora y kardex**, que es el control que importa.
- **Solo `COMPRAS` y `ADMIN` reciben compras.** Es el punto donde entra stock al sistema
  (ver `02-CORRECCIONES-DB.md` §C-03). Es la operación de mayor impacto sobre el inventario
  y se mantiene acotada.

---

## 4. Implementación en Spring Security

### Nombres de authority

Spring espera el prefijo `ROLE_`. El código del rol en base de datos **no lo lleva**; se
agrega al construir el `UserDetails`:

```java
// UsuarioDetailsService
new SimpleGrantedAuthority("ROLE_" + usuario.getRol().getNombre())  // ROLE_ADMIN
```

En las anotaciones se usa `hasRole`, que agrega el prefijo automáticamente:

```java
@PreAuthorize("hasRole('ADMIN')")                      // ✅
@PreAuthorize("hasAnyRole('ADMIN','COMPRAS')")         // ✅
@PreAuthorize("hasAuthority('ADMIN')")                 // ❌ no coincide con ROLE_ADMIN
```

Este desajuste entre `hasRole` y `hasAuthority` es la causa más común de un `403` que
"no debería pasar". Se usa `hasRole` en todo el proyecto.

### Dónde se declara

En el **controlador**, sobre cada método. No en el servicio (se pierde el contexto HTTP y
complica las pruebas) ni solo en `SecurityConfig` (queda lejos del endpoint y se
desincroniza al refactorizar).

```java
@RestController
@RequestMapping("/api/v1/materias-primas")
public class MateriaPrimaController {

    @GetMapping
    @PreAuthorize("hasAnyRole('ADMIN','PRODUCCION','COMPRAS','CONSULTA')")
    public PageResponse<MateriaPrimaResponse> listar(...) { ... }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<MateriaPrimaResponse> crear(@Valid @RequestBody CrearMateriaPrimaRequest req) { ... }
}
```

Requiere `@EnableMethodSecurity` en `SecurityConfig`. Sin esa anotación **las
`@PreAuthorize` se ignoran en silencio** y todo queda abierto. Es un fallo que no produce
ningún error visible: la aplicación arranca, los endpoints responden, y la autorización
simplemente no existe. Hay un caso de prueba dedicado a esto en `05-ESTANDARES-QA.md`.

### Rutas públicas

Solo dos, declaradas en `SecurityConfig`:

```java
.requestMatchers("/api/v1/auth/login").permitAll()
.requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()  // solo perfil dev
.anyRequest().authenticated()
```

Swagger se cierra fuera de desarrollo mediante perfil de Spring.

---

## 5. Implementación en Angular

### Guard de rol

```typescript
export const rolGuard = (rolesPermitidos: Rol[]): CanActivateFn => () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  if (auth.tieneAlgunRol(rolesPermitidos)) return true;
  router.navigate(['/sin-acceso']);
  return false;
};

// app.routes.ts
{
  path: 'usuarios',
  canActivate: [authGuard, rolGuard(['ADMIN'])],
  loadChildren: () => import('./features/usuario/usuario.routes')
}
```

### Ocultar lo que no se puede usar

Un botón que siempre devuelve `403` es una mala experiencia. Se oculta con una directiva
estructural:

```html
<button mat-raised-button *siRol="['ADMIN']" (click)="crear()">Nuevo usuario</button>
```

> **Esto es cosmética, no seguridad.** El menú y los botones ocultos mejoran la interfaz;
> cualquiera puede llamar al endpoint con Postman. **La autorización real está en el
> backend, siempre.** El frontend nunca es la barrera.

---

## 6. Respuesta ante falta de permisos

| Situación | HTTP | Código | Mensaje |
|---|---|---|---|
| Sin token o token inválido/expirado | `401` | `NO_AUTENTICADO` | "Su sesión ha expirado. Inicie sesión nuevamente." |
| Token válido, rol insuficiente | `403` | `SIN_PERMISO` | "No tiene permisos para realizar esta acción." |
| Usuario desactivado (HU-03 CA-03) | `403` | `USUARIO_INACTIVO` | "Su usuario está desactivado. Contacte al administrador." |

El interceptor de Angular distingue los tres: `401` cierra sesión y redirige al login;
`403` muestra un aviso sin cerrar la sesión.
