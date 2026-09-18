# HU-01 · Registro de usuarios — Diseño

---

## 1. Pantallas

### Formulario de registro (diálogo modal sobre el listado de HU-02)

```
┌──────────────────────────────────────────────────┐
│  Registrar usuario                          [X]  │
├──────────────────────────────────────────────────┤
│                                                  │
│  Nombre completo *                               │
│  ┌────────────────────────────────────────────┐  │
│  │ María Gómez                                │  │
│  └────────────────────────────────────────────┘  │
│                                      12 / 100    │
│                                                  │
│  Correo electrónico *                            │
│  ┌────────────────────────────────────────────┐  │
│  │ maria@productionfood.local                 │  │
│  └────────────────────────────────────────────┘  │
│  ⚠ Ya existe un usuario con este correo          │
│                                                  │
│  Contraseña *                                    │
│  ┌────────────────────────────────────────┬───┐  │
│  │ ••••••••••                             │ 👁 │  │
│  └────────────────────────────────────────┴───┘  │
│  Fortaleza: ████████░░  Buena                    │
│                                                  │
│  Confirmar contraseña *                          │
│  ┌────────────────────────────────────────────┐  │
│  │ ••••••••••                                 │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  Rol *                                           │
│  ┌────────────────────────────────────────────┐  │
│  │ Ventas                                  ▾  │  │
│  └────────────────────────────────────────────┘  │
│  Atiende clientes, pedidos y devoluciones        │
│                                                  │
│              [ Cancelar ]  [ Registrar usuario ] │
└──────────────────────────────────────────────────┘
```

El texto bajo el selector de rol muestra `roles.descripcion`. Sin él, quien registra tiene
que adivinar qué implica cada rol — y asignar permisos a ciegas es cómo terminan todos los
usuarios siendo administradores.

---

## 2. Flujo

```
Administrador                Frontend                 Backend
     │                          │                        │
     │  Abre "Nuevo usuario"    │                        │
     ├─────────────────────────►│                        │
     │                          │  GET /roles            │
     │                          ├───────────────────────►│
     │                          │◄───────────────────────┤
     │  Ve el formulario        │   5 roles              │
     │◄─────────────────────────┤                        │
     │                          │                        │
     │  Completa y envía        │                        │
     ├─────────────────────────►│  POST /usuarios        │
     │                          ├───────────────────────►│
     │                          │                        │ valida rol
     │                          │                        │ valida correo único
     │                          │                        │ BCrypt(password)
     │                          │                        │ INSERT
     │                          │                        │ bitácora (HU-04)
     │                          │◄───────────────────────┤
     │   "Usuario registrado"   │   201 + Location       │
     │◄─────────────────────────┤                        │
```

Si el correo ya existe, el backend devuelve `409 CORREO_DUPLICADO` y el formulario marca
el campo de correo sin perder lo ya escrito.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Diálogo | `MatDialog` |
| Campos de texto | `MatFormField` + `MatInput` |
| Contraseña | `MatInput[type=password]` + `MatIconButton` para mostrar/ocultar |
| Fortaleza | `MatProgressBar` con color según nivel |
| Rol | `MatSelect` + `MatOption` (poblado desde la API) |
| Botones | `MatButton` (Cancelar) · `MatRaisedButton color="primary"` (Registrar) |
| Avisos | `MatSnackBar` vía `NotificacionService` |

---

## 4. Validaciones en el formulario

| Campo | Reglas de cliente | Mensaje |
|---|---|---|
| Nombre | requerido, máx. 100 | "El nombre es obligatorio" / "Máximo 100 caracteres" |
| Correo | requerido, formato correo, máx. 100 | "Ingrese un correo válido" |
| Contraseña | requerida, 8–72 caracteres | "Mínimo 8 caracteres" |
| Confirmar | requerida, igual a contraseña | "Las contraseñas no coinciden" |
| Rol | requerido | "Seleccione un rol" |

La validación del cliente es para comodidad del usuario. **La autoridad es el backend**:
el correo duplicado solo puede detectarse allí.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando roles | Selector deshabilitado con spinner |
| Enviando | Botón deshabilitado, texto "Registrando…" |
| Error de validación | Mensaje bajo cada campo afectado |
| Error de servidor | `MatSnackBar` rojo, formulario conserva los datos |
| Éxito | Diálogo se cierra, `MatSnackBar` verde, listado se recarga |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
