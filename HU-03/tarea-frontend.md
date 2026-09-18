# HU-03 · Inicio de sesión — Tarea Frontend

Pantalla de login, `AuthService`, interceptores y guards. Bloquea toda la navegación del resto de la aplicación.

---

## 1. Pasos

1. `core/auth/auth.service.ts` con signals (`usuario`, `autenticado`, `rol`),
   `login()`, `logout()`, `tieneAlgunRol()`.
2. Decodificación del JWT para leer los claims. **Solo para mostrar datos en la interfaz**,
   nunca para decidir permisos reales.
3. Validar `exp` al leer el token de `localStorage`: un token vencido produciría una sesión
   fantasma, con el menú visible y `401` en cada petición.
4. `core/auth/auth.interceptor.ts` — adjunta `Authorization: Bearer`.
5. `core/http/error.interceptor.ts` — `401` cierra sesión, `403` solo avisa.
6. `authGuard` y `rolGuard(roles)`.
7. `login.component`: formulario con correo y contraseña, mostrar/ocultar contraseña.
8. Redirección tras login según rol: ADMIN → dashboard, VENTAS → pedidos, etc.
9. Layout con barra superior: nombre del usuario, rol y "Cerrar sesión".
10. Menú lateral que muestra solo los módulos del rol (directiva `*siRol`).

---

## 2. Archivos

```
frontend/src/app/
├── core/auth/{auth.service,auth.guard,rol.guard,auth.interceptor}.ts
├── core/http/error.interceptor.ts
├── features/auth/login.component.{ts,html,scss}
├── layout/layout.component.{ts,html,scss}
└── shared/directives/si-rol.directive.ts
```

---

## 3. Puntos de cuidado

- **El token en `localStorage` es legible por cualquier script de la página.** Es lo que
  especifica el stack; se compensa evitando `innerHTML` con datos del servidor. Documentado
  en `07-ARQUITECTURA-FRONTEND.md` §3.
- **`403` no debe cerrar la sesión**: el usuario está autenticado, solo no autorizado para
  esa acción. Cerrarle la sesión por pulsar un botón equivocado es desconcertante.
- El menú filtrado por rol es **cosmética**: la seguridad real está en el backend.
- Mostrar el mismo mensaje genérico del servidor en el `401`; no inventar "el correo no
  existe" en el cliente (anularía R-01).
- Limpiar `localStorage` en `logout()`, incluido cualquier estado cacheado.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
