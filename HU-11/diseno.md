# HU-11 · Gestión de productos — Diseño

---

## 1. Pantallas

### Catálogo de productos

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Productos                                                   [ + Nuevo producto ] │
├──────────────────────────────────────────────────────────────────────────────────┤
│  🔍 ┌────────────────────┐  Estado ┌───────────┐  ☐ Solo sin receta             │
│     │ Buscar por nombre  │         │ Activos ▾ │                                 │
│     └────────────────────┘         └───────────┘                                 │
├──────────────────────────────────────────────────────────────────────────────────┤
│  PRODUCTO ▲       DESCRIPCIÓN            PRECIO      UNIDAD  RECETA  STOCK   ⋯   │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Croissant        Hojaldre con mant…   $ 2.500,00    und      ✅      24    ⋮   │
│  Pan francés      Corteza crujiente…   $ 1.200,00    und      ✅     120    ⋮   │
│  Pan integral     Con salvado, 500 g   $ 4.800,00    und      ⚠️       0    ⋮   │
│  Torta de zana…   Torta húmeda 1 kg   $ 32.000,00    und      ✅       3    ⋮   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

⚠️ marca los productos **sin receta**: se pueden vender si hay stock, pero no se pueden
planificar (R-02). El icono enlaza directamente a crear la receta en HU-12.

La columna STOCK viene de `v_stock_producto_terminado` (V2 §C-02), que agrega los lotes
vigentes de HU-19.

### Formulario

```
┌────────────────────────────────────────────────┐
│  Nuevo producto                           [X]  │
├────────────────────────────────────────────────┤
│  Nombre *                                      │
│  ┌──────────────────────────────────────────┐  │
│  │ Pan francés                              │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  Descripción                                   │
│  ┌──────────────────────────────────────────┐  │
│  │ Pan de corteza crujiente, 80 g           │  │
│  └──────────────────────────────────────────┘  │
│                                    31 / 150    │
│                                                │
│  Precio de venta *        Unidad de venta *    │
│  ┌──────────────────┐    ┌──────────────────┐  │
│  │ $ 1.200,00       │    │ Unidad (und)  ▾  │  │
│  └──────────────────┘    └──────────────────┘  │
│                           Cómo se vende, no    │
│                           cómo se produce      │
│                                                │
│              [ Cancelar ]  [ Guardar ]         │
└────────────────────────────────────────────────┘
```

La nota bajo el selector aclara R-04: esta unidad es la de venta. La receta (HU-12) tiene
la suya, que puede ser distinta — se vende por unidad, se produce por tandas de 30.

---

## 2. Flujo

```
Administrador        Frontend                Backend
     │                  │                       │
     │ Nuevo producto   │ GET /unidades-medida  │ (caché de HU-07)
     ├─────────────────►│                       │
     │ Completa, guarda │ POST /productos       │
     ├─────────────────►├──────────────────────►│ ¿unidad existe?
     │                  │                       │ INSERT (estado = 1)
     │                  │                       │ bitácora
     │                  │◄──────────────────────┤ 201
     │ "Registrado"     │                       │
     │◄─────────────────┤                       │
     │                  │                       │
     │  ⚠️ sin receta   │                       │
     │  → enlace a HU-12 para crearla           │
```

**A diferencia de HU-08, no se crea ninguna fila de inventario**: el stock de producto
terminado entra con los lotes de HU-19.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Precio | Pipe `currency:'COP':'symbol-narrow':'1.2-2'` |
| Receta | `MatIcon` ✅/⚠️ con `MatTooltip` y `RouterLink` a HU-12 |
| Stock | Texto con color según cantidad |
| Unidad | `MatSelect` (catálogo cacheado) |
| Descripción | `MatInput` + `MatHint` con contador |
| Formulario | `MatDialog` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Nombre | requerido, máx. 100 | "El nombre es obligatorio" |
| Descripción | opcional, máx. 150 | "Máximo 150 caracteres" |
| Precio | requerido, ≥ 0, máx. 2 decimales | "Debe ser 0 o mayor" |
| Unidad | requerida | "Seleccione una unidad de venta" |

**Precio cero es válido** (R-06): sirve para muestras y promociones.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío sin filtro | "No hay productos registrados" + "Registrar el primero" |
| Vacío con filtro | "Ningún producto coincide con «xxx»" |
| Vacío filtrando sin receta | ✅ "Todos los productos tienen receta" |
| Sin unidades disponibles | Enlace a HU-07 |
| Error | Mensaje + "Reintentar" |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
