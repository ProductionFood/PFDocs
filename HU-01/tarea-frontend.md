# HU-01 · Registro de usuarios — Tarea Frontend

Formulario de alta de usuario. Es la primera pantalla del proyecto: aquí se establecen el `NotificacionService`, el manejo de `fieldErrors` y el layout base que reutilizan las demás.

---

## 1. Pasos

1. Estructura base: `core/`, `shared/`, `features/` según `01-CONVENCIONES.md` §4.
2. `shared/models/`: `PageResponse<T>`, `ApiError`.
3. `core/ui/notificacion.service.ts` — envoltorio de `MatSnackBar` (éxito / error / aviso).
4. `features/usuario/models/usuario.model.ts`: `Usuario`, `CrearUsuarioDto`, `Rol`.
5. `features/usuario/services/usuario.service.ts`: `crear()`, y `rol.service.ts`: `listar()`.
6. `usuario-formulario.component`: Reactive Form con los cuatro campos y confirmación de
   contraseña.
7. Selector de rol poblado desde `GET /api/v1/roles` (nunca una lista escrita a mano en
   el código: si se agrega un rol en la base, el formulario queda desactualizado).
8. Indicador de fuerza de contraseña y botón de mostrar/ocultar.
9. Mapear `fieldErrors` de la respuesta al control correspondiente.
10. Ruta con `rolGuard(['ADMIN'])`.

---

## 2. Archivos

```
frontend/src/app/
├── core/ui/notificacion.service.ts
├── shared/models/{page-response,api-error}.model.ts
└── features/usuario/
    ├── models/usuario.model.ts
    ├── services/{usuario,rol}.service.ts
    ├── pages/usuario-formulario/
    └── usuario.routes.ts
```

---

## 3. Puntos de cuidado

- **La confirmación de contraseña es solo del cliente**: no se envía al backend.
- El correo se envía en minúsculas y sin espacios, igual que lo normaliza el servidor.
- `maxLength(72)` en el campo de contraseña, coherente con R-03.
- Deshabilitar el botón mientras la petición está en vuelo: evita crear el usuario dos veces.
- El `409 CORREO_DUPLICADO` se muestra **junto al campo de correo**, no en un toast.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
