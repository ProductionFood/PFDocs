# HU-02 · Gestión de usuarios — Diseño

---

## 1. Pantallas

### Listado de usuarios

```
┌────────────────────────────────────────────────────────────────────────┐
│  Usuarios                                        [ + Nuevo usuario ]   │
├────────────────────────────────────────────────────────────────────────┤
│  🔍 ┌──────────────────────┐  Rol ┌────────────┐  Estado ┌──────────┐  │
│     │ Buscar nombre/correo │      │ Todos    ▾ │         │ Todos  ▾ │  │
│     └──────────────────────┘      └────────────┘         └──────────┘  │
├────────────────────────────────────────────────────────────────────────┤
│  NOMBRE ▲        CORREO                   ROL          ESTADO   ⋯      │
├────────────────────────────────────────────────────────────────────────┤
│  Administrador   admin@production...      ADMIN        ●Activo  ⋮      │
│  Carlos Cochero  carlos@production...     COMPRAS      ●Activo  ⋮      │
│  Ismael Batalla  ismael@production...     PRODUCCION   ○Inactivo ⋮     │
│  María Gómez     maria@production...      VENTAS       ●Activo  ⋮      │
├────────────────────────────────────────────────────────────────────────┤
│                        Elementos por página: 20 ▾   1–4 de 4   ◀  ▶   │
└────────────────────────────────────────────────────────────────────────┘
```

Menú `⋮` de cada fila: **Editar** · **Desactivar** (o **Activar**).
**No hay opción de eliminar** (CA-05).

### Confirmación de desactivación

```
┌──────────────────────────────────────────────┐
│  Desactivar usuario                          │
├──────────────────────────────────────────────┤
│  ¿Desactivar a María Gómez?                  │
│                                              │
│  No podrá iniciar sesión ni continuar su     │
│  sesión actual. Sus registros históricos se  │
│  conservan.                                  │
│                                              │
│           [ Cancelar ]  [ Desactivar ]       │
└──────────────────────────────────────────────┘
```

El texto explica la consecuencia real. "¿Está seguro?" no informa nada: quien lo lee ya
pulsó el botón a propósito.

---

## 2. Flujo

```
Administrador            Frontend                    Backend
     │                      │                           │
     │  Escribe "mar"       │                           │
     ├─────────────────────►│ (debounce 350 ms)         │
     │                      ├──────────────────────────►│ GET /usuarios?busqueda=mar
     │                      │◄──────────────────────────┤ 200 PageResponse
     │  Ve resultados       │                           │
     │◄─────────────────────┤                           │
     │                      │                           │
     │  Menú ⋮ → Desactivar │                           │
     ├─────────────────────►│                           │
     │  Confirma            │ PATCH /usuarios/12/estado │
     ├─────────────────────►├──────────────────────────►│
     │                      │                           │ ¿es él mismo? → 409
     │                      │                           │ ¿último ADMIN? → 409
     │                      │                           │ UPDATE estado = 0
     │                      │                           │ bitácora
     │                      │◄──────────────────────────┤ 200
     │  "Usuario desactivado"                           │
     │◄─────────────────────┤ (recarga el listado)      │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Búsqueda | `MatFormField` + `MatInput` + `MatIcon` (lupa) |
| Filtros | `MatSelect` (rol, estado) |
| Estado | `MatChip` — verde activo / gris inactivo |
| Menú de fila | `MatMenu` + `MatIconButton` |
| Confirmación | `MatDialog` (`ConfirmarDialogComponent`) |
| Carga | `MatProgressBar` |

---

## 4. Validaciones en el formulario

El formulario de edición reutiliza el de HU-01 **sin los campos de contraseña** (R-03).

| Campo | Reglas |
|---|---|
| Nombre | requerido, máx. 100 |
| Correo | requerido, formato correo, máx. 100 |
| Rol | requerido |

El campo de búsqueda no valida: cualquier texto es una búsqueda legítima, incluidos los
caracteres especiales (que el backend escapa).

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` sobre la tabla; filtros activos |
| Vacío sin filtro | "No hay usuarios registrados" + botón "Registrar el primero" |
| Vacío con filtro | "Ningún usuario coincide con «mar»" + "Limpiar filtros" |
| Error | Mensaje + botón "Reintentar" |
| Con datos | Tabla + paginador |

Distinguir el vacío por filtro del vacío real evita que el usuario crea que se perdieron
los datos.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
