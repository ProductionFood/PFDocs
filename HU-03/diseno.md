# HU-03 · Inicio de sesión — Diseño

---

## 1. Pantallas

### Pantalla de acceso

```
              ┌────────────────────────────────────────┐
              │                                        │
              │            🥖 ProductionFood           │
              │     Gestión de producción de alimentos │
              │                                        │
              │  ┌──────────────────────────────────┐  │
              │  │ Correo electrónico               │  │
              │  │ maria@productionfood.local       │  │
              │  └──────────────────────────────────┘  │
              │                                        │
              │  ┌──────────────────────────────┬───┐  │
              │  │ Contraseña                   │ 👁 │  │
              │  │ ••••••••••                   │   │  │
              │  └──────────────────────────────┴───┘  │
              │                                        │
              │  ⚠ Correo o contraseña incorrectos     │
              │                                        │
              │  ┌──────────────────────────────────┐  │
              │  │        Iniciar sesión            │  │
              │  └──────────────────────────────────┘  │
              │                                        │
              └────────────────────────────────────────┘
```

El mensaje de error es siempre el mismo, exista o no el correo (R-01).

### Barra superior tras iniciar sesión

```
┌────────────────────────────────────────────────────────────────────────┐
│ ☰  ProductionFood                        María Gómez · VENTAS   [⏻]    │
└────────────────────────────────────────────────────────────────────────┘
```

Mostrar el rol activo evita la confusión de "¿por qué no veo el módulo de compras?".

---

## 2. Flujo

```
Usuario              Frontend                      Backend
   │                    │                             │
   │ Credenciales       │                             │
   ├───────────────────►│  POST /auth/login           │
   │                    ├────────────────────────────►│
   │                    │                             │ busca por correo
   │                    │                             │ matches(password)  ← siempre
   │                    │                             │ ¿estado = 1?
   │                    │                             │ genera JWT (1 h)
   │                    │◄────────────────────────────┤ 200 { token, usuario }
   │                    │ guarda en localStorage      │
   │                    │ actualiza signal de sesión  │
   │  Dashboard         │                             │
   │◄───────────────────┤                             │
   │                    │                             │
   │  (cualquier acción)│  GET /... + Bearer token    │
   │                    ├────────────────────────────►│ valida firma
   │                    │                             │ valida exp
   │                    │                             │ valida estado = 1  ← cada vez
   │                    │◄────────────────────────────┤ 200
```

**Al expirar el token:** cualquier petición devuelve `401`; el interceptor cierra la sesión
y redirige al login con el aviso "Su sesión expiró".

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjeta de login | `MatCard` centrada |
| Campos | `MatFormField` + `MatInput` |
| Ver contraseña | `MatIconButton` con `visibility` / `visibility_off` |
| Botón | `MatRaisedButton color="primary"`, ancho completo |
| Error | `MatError` sobre el formulario, no un toast |
| Cargando | `MatProgressSpinner` dentro del botón |
| Barra superior | `MatToolbar` + `MatMenu` de usuario |
| Menú lateral | `MatSidenav` + `MatNavList` filtrado por rol |

---

## 4. Validaciones en el formulario

| Campo | Reglas de cliente | Mensaje |
|---|---|---|
| Correo | requerido, formato correo | "Ingrese un correo válido" |
| Contraseña | requerida | "Ingrese su contraseña" |

**El cliente no valida la longitud de la contraseña en el login.** Hacerlo revelaría la
política de contraseñas a quien intenta adivinar, y además impediría entrar a usuarios
creados antes de un cambio de política.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Inicial | Formulario vacío, foco en el correo |
| Enviando | Botón con spinner, campos deshabilitados |
| Credenciales incorrectas | `MatError` genérico; la contraseña se limpia, el correo se conserva |
| Usuario inactivo | "Su usuario está desactivado. Contacte al administrador." |
| Sin conexión | "No se pudo conectar con el servidor." |
| Éxito | Redirección según rol |

Conservar el correo y limpiar solo la contraseña es lo que espera quien se equivocó al
teclear: no lo obliga a reescribir todo.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
