# HU-21 · Productos en plan de producción — Diseño

---

## 1. Pantallas

### Tabla de productos del plan

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  PRODUCTOS DEL PLAN #9                                    [ + Agregar producto ]  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  PRODUCTO         PLANIFICADO   PRODUCIDO   AVANCE           RECETA        ⋯     │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Pan francés         90 und     ┌────────┐  ██████░░░░  67%    ✅          ✏️🗑️ │
│                                 │ 60     │                                       │
│                                 └────────┘                                       │
│  Croissant           30 und     ┌────────┐  ██████████ 100%    ✅          ✏️🗑️ │
│                                 │ 30     │                                       │
│                                 └────────┘                                       │
│  Pan integral        20 und     ┌────────┐  ░░░░░░░░░░   0%    ⚠️ sin      ✏️🗑️ │
│                                 │  0     │                        receta         │
├──────────────────────────────────────────────────────────────────────────────────┤
│  ℹ Registre el lote de producto terminado para que lo producido                   │
│    esté disponible para la venta.                        [ Registrar lote → ]    │
└──────────────────────────────────────────────────────────────────────────────────┘
```

Los campos de **PRODUCIDO son editables en línea** cuando el plan está `EN_PROCESO` (R-03).

⚠️ **El aviso del pie es importante** (R-07): registrar producción aquí no pone el producto
a la venta. Sin ese recordatorio y su enlace a HU-19, es fácil completar un plan y que el
producto fabricado nunca aparezca en el inventario vendible.

`Pan integral` no tiene receta: se puede planificar, pero quedará fuera del análisis de
consumo y eficiencia (R-06).

### Agregar producto al plan

```
┌──────────────────────────────────────────────────────────┐
│  Agregar producto al plan                           [X]  │
├──────────────────────────────────────────────────────────┤
│  Producto *                                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Pan francés (und)                               ▾  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Cantidad a producir *                                   │
│  ┌──────────────────┐                                    │
│  │ 90           und │                                    │
│  └──────────────────┘                                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Materia prima que se necesitará                   │  │
│  │                                                    │  │
│  │  Harina de trigo   15,000 kg   disponible 112,50 ✅│  │
│  │  Levadura           0,150 kg   disponible   0,00 🔴│  │
│  │  Sal                0,300 kg   disponible   8,20 ✅│  │
│  │                                                    │  │
│  │  ⚠ No hay levadura suficiente para esta producción │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│              [ Cancelar ]  [ Agregar ]                   │
└──────────────────────────────────────────────────────────┘
```

**El contraste con el inventario disponible es lo más útil de esta pantalla.** Descubrir que
falta levadura ahora, al planificar, es muy distinto a descubrirlo a mitad de tanda con la
masa ya hecha.

También sirve de segunda verificación de la receta (HU-12 R-02): si planificar 90 panes
pidiera 450 kg de harina, el error de `cantidad_producir` salta a la vista aquí.

### Producto sin receta

```
┌──────────────────────────────────────────────────────────┐
│  ⚠  Este producto no tiene receta                        │
│                                                          │
│  «Pan integral» se puede agregar al plan, pero:          │
│    • no se calculará su consumo teórico de materia prima │
│    • quedará fuera del análisis de eficiencia (HU-26)    │
│                                                          │
│  [ Crear receta primero ]  [ Agregar de todos modos ]    │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Flujo

```
Productor            Frontend                  Backend
    │                   │                         │
    │ "+ Agregar"       │ GET /productos          │
    ├──────────────────►│ (excluye los ya presentes)
    │ Elige y escribe   │                         │
    │ cantidad          │ (calcula consumo teórico│
    │                   │  y lo contrasta con     │
    │                   │  GET /inventario)       │
    │◄──────────────────┤                         │
    │ Ve si alcanza     │                         │
    │ Confirma          │ POST .../detalles       │
    ├──────────────────►├────────────────────────►│ ¿plan PLANIFICADO?
    │                   │                         │ ¿producto existe?
    │                   │                         │ ¿duplicado?      ← R-05
    │                   │                         │ INSERT (producida = 0)
    │                   │◄────────────────────────┤ 201
    │                   │                         │
    │  ... plan pasa a EN_PROCESO (HU-20) ...     │
    │                   │                         │
    │ Escribe 60 en     │ PATCH .../21/producido  │
    │ PRODUCIDO         ├────────────────────────►│ ¿plan EN_PROCESO? ← R-03
    ├──────────────────►│                         │ UPDATE cantidad_producida
    │                   │◄────────────────────────┤ 200, avance 66,67%
    │                   │                         │
    │  ⚠ Recordatorio: registrar el lote (HU-19)  │  ← R-07
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` con edición en línea |
| Avance | `MatProgressBar` con color por tramo |
| Producido | `MatInput[type=number]` en línea, con guardado al perder el foco |
| Receta | `MatIcon` ✅/⚠️ con `MatTooltip` |
| Consumo teórico | `MatCard` en el diálogo, con semáforo de disponibilidad |
| Agregar | `MatDialog` |
| Recordatorio del lote | `MatCard` informativa con `RouterLink` a HU-19 |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Producto | requerido, no repetido | "Seleccione un producto" |
| Cantidad planificada | requerida, > 0 | "Debe ser mayor que cero" |
| Cantidad producida | requerida, ≥ 0, **sin tope superior** | "No puede ser negativa" |

**La cantidad producida no tiene límite superior** (R-04): producir más de lo planificado es
un resultado posible y un dato que HU-26 debe poder reflejar.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Plan sin productos | ⚠️ "Agregue productos antes de iniciar la producción" |
| Producto sin receta | ⚠️ Advertencia con opción de crear la receta primero |
| Materia prima insuficiente | 🔴 En el diálogo, junto al ingrediente que falta |
| Plan `PLANIFICADO` | Editable lo planificado; campo de producido deshabilitado |
| Plan `EN_PROCESO` | Producido editable; planificado bloqueado |
| Plan cerrado | 🔒 Todo en modo lectura |
| Avance > 100% | Barra azul con el porcentaje, no como error |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
