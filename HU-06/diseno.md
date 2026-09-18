# HU-06 · Gestión de proveedores — Diseño

---

## 1. Pantallas

### Listado de proveedores

```
┌────────────────────────────────────────────────────────────────────────────┐
│  Proveedores                                          [ + Nuevo proveedor ] │
├────────────────────────────────────────────────────────────────────────────┤
│  🔍 ┌──────────────────────┐   Estado ┌────────────┐                       │
│     │ Buscar por nombre    │          │ Activos  ▾ │                       │
│     └──────────────────────┘          └────────────┘                       │
├────────────────────────────────────────────────────────────────────────────┤
│  NOMBRE ▲                  CONTACTO        TELÉFONO      CORREO      ⋯     │
├────────────────────────────────────────────────────────────────────────────┤
│  Distribuidora Lácteos     Ana Mejía       605 277 1100  ✉ ventas@…  ⋮     │
│  Harinas del Caribe S.A.S. Jorge Mendoza   605 278 4410  ✉ ventas@…  ⋮     │
│  Insumos del Sinú          Marta Díaz      310 998 7712  —           ⋮     │
├────────────────────────────────────────────────────────────────────────────┤
│                        Elementos por página: 20 ▾   1–3 de 3   ◀  ▶       │
└────────────────────────────────────────────────────────────────────────────┘
```

El correo es un enlace `mailto:`. Cuando no hay, se muestra `—`, no una celda vacía: así se
distingue "sin dato" de "falló la carga".

### Formulario

```
┌────────────────────────────────────────────┐
│  Nuevo proveedor                      [X]  │
├────────────────────────────────────────────┤
│  Nombre *                                  │
│  ┌──────────────────────────────────────┐  │
│  │ Harinas del Caribe S.A.S.            │  │
│  └──────────────────────────────────────┘  │
│  Persona de contacto                       │
│  ┌──────────────────────────────────────┐  │
│  │ Jorge Mendoza                        │  │
│  └──────────────────────────────────────┘  │
│  Teléfono              Correo electrónico  │
│  ┌─────────────────┐  ┌─────────────────┐  │
│  │ 605 278 4410    │  │ ventas@hari...  │  │
│  └─────────────────┘  └─────────────────┘  │
│                                            │
│              [ Cancelar ]  [ Guardar ]     │
└────────────────────────────────────────────┘
```

Solo el nombre lleva asterisco: los demás son opcionales (CA-02).

---

## 2. Flujo

```
Encargado de compras      Frontend                Backend
       │                     │                       │
       │ Abre "Proveedores"  │ GET /proveedores      │
       ├────────────────────►├──────────────────────►│
       │◄────────────────────┤◄──────────────────────┤ 200
       │                     │                       │
       │ Nuevo proveedor     │ POST /proveedores     │
       ├────────────────────►├──────────────────────►│ valida correo (si viene)
       │                     │                       │ INSERT (estado = 1)
       │                     │                       │ bitácora
       │◄────────────────────┤◄──────────────────────┤ 201
       │                     │                       │
       │ Desactivar          │ PATCH .../estado      │
       ├────────────────────►├──────────────────────►│ UPDATE estado = 0
       │◄────────────────────┤◄──────────────────────┤ 200
       │                                             │
       │  (HU-13 ya no permitirá comprarle — R-02)   │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Búsqueda | `MatFormField` + `MatInput` |
| Estado | `MatChip` verde/gris |
| Correo | Enlace `mailto:` con `MatIcon` |
| Menú de fila | `MatMenu` |
| Formulario | `MatDialog` |
| Confirmación | `ConfirmarDialogComponent` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Nombre | requerido, máx. 100 | "El nombre es obligatorio" |
| Contacto | opcional, máx. 100 | — |
| Teléfono | opcional, máx. 20 | — |
| Correo | opcional; si se escribe, formato válido, máx. 100 | "Ingrese un correo válido" |

El correo **no bloquea el guardado si está vacío**. Solo se valida cuando tiene contenido.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío sin filtro | "No hay proveedores registrados" + "Registrar el primero" |
| Vacío con filtro | "Ningún proveedor coincide con «xxx»" |
| Error | Mensaje + "Reintentar" |
| Guardando | Botón con spinner |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
