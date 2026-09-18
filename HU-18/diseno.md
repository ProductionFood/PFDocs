# HU-18 · Registro de devoluciones — Diseño

---

## 1. Pantallas

### Bandeja de devoluciones

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Devoluciones                                                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Estado ┌──────────────┐  Período ┌─────────────────────────┐                   │
│         │ Pendientes ▾ │          │ 01/09/2026 – 30/09/2026 │                   │
│         └──────────────┘          └─────────────────────────┘                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  PEDIDO  CLIENTE            PRODUCTO       CANT.  MOTIVO            FECHA   ⋯   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  #47     Tienda La Esquina  Pan francés      3    Producto en mal…  18/09   ⋮   │
│  #46     Supermercado Cen…  Croissant        2    Error en el ped…  17/09   ⋮   │
│  #45     Panadería El Trigo Torta de zana…   1    Cliente insatis…  16/09   ⋮   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

Filtrada por `PENDIENTE` de entrada: es la bandeja de trabajo de quien revisa devoluciones.
Menú `⋮`: **Aprobar** · **Rechazar** · **Ver pedido**.

### Registrar devolución (desde el detalle del pedido)

```
┌────────────────────────────────────────────────────────┐
│  Registrar devolución                             [X]  │
├────────────────────────────────────────────────────────┤
│  Pedido #47 · Tienda La Esquina                        │
│  Producto: Pan francés                                 │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Cantidad pedida:              30 und            │  │
│  │  Ya devuelta o en trámite:      3 und            │  │
│  │  Disponible para devolver:     27 und            │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  Cantidad a devolver *                                 │
│  ┌──────────────────┐                                  │
│  │ 3            und │                                  │
│  └──────────────────┘                                  │
│                                                        │
│  Motivo *                                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Producto en mal estado                        ▾  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  Descripción                                           │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Tres panes llegaron aplastados en el transporte  │  │
│  └──────────────────────────────────────────────────┘  │
│                                            48 / 255    │
│                                                        │
│  ℹ La devolución quedará pendiente de aprobación.      │
│    El stock volverá al inventario solo al aprobarse.   │
│                                                        │
│              [ Cancelar ]  [ Registrar ]               │
└────────────────────────────────────────────────────────┘
```

**Los tres números del recuadro** son lo que hace comprensible el límite (R-03). Sin
"ya devuelta o en trámite", el `422` parece arbitrario.

La nota final explica R-01: registrar no repone.

### 🔑 Diálogo de aprobación

```
┌────────────────────────────────────────────────────────┐
│  ✓  Aprobar devolución                                 │
├────────────────────────────────────────────────────────┤
│  Pedido #47 · Pan francés · 3 unidades                 │
│  Motivo: Producto en mal estado                        │
│                                                        │
│  Al aprobar, el stock volverá al inventario:           │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Lote PF-2026-0917    +3 und    22 → 25 und      │  │
│  │  Vence el 21/09/2026 (en 3 días)                 │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  🔒 Esta acción no se puede deshacer.                  │
│                                                        │
│       [ Cancelar ]  [ Confirmar aprobación ]           │
└────────────────────────────────────────────────────────┘
```

Muestra **a qué lote vuelve** y su vencimiento (R-02). Si el lote está vencido, se advierte
que la mercancía no volverá como stock vendible.

---

## 2. Flujo

```
Vendedor          Frontend            Backend         InventarioPtService
   │                 │                   │                    │
   │ "Devolver" en   │ GET detalle + devoluciones previas     │
   │ la línea        ├──────────────────►│                    │
   │                 │◄──────────────────┤ disponible: 27     │
   │ Completa        │ POST /devoluciones│                    │
   ├────────────────►├──────────────────►│ ¿pedido devolvible?│
   │                 │                   │ ¿acumulado ≤ pedido?
   │                 │                   │ INSERT (PENDIENTE) │
   │                 │◄──────────────────┤ 201                │
   │                 │                   │  ← stock SIN cambios (R-01)
   │                 │                   │                    │
   │ ... revisión física de la mercancía ...                  │
   │                 │                   │                    │
   │ "Aprobar"       │ PATCH /devoluciones/15/estado          │
   ├────────────────►├──────────────────►│                    │
   │                 │                   │ ── BEGIN ────────► │
   │                 │                   │ ¿transición válida?│
   │                 │                   │ consulta el kardex:│
   │                 │                   │ ¿de qué lotes salió?
   │                 │                   ├───────────────────►│ repone en orden inverso
   │                 │                   │                    │ INSERT movimiento
   │                 │                   │ UPDATE estado      │
   │                 │                   │ bitácora APROBAR   │
   │                 │                   │ ── COMMIT ───────► │
   │                 │◄──────────────────┤ 200 + lotesRepuestos
   │ "Stock repuesto"│                   │                    │
   │◄────────────────┤                   │                    │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Bandeja | `MatTable` + `MatSort` + `MatPaginator` |
| Estado | `MatChip` — ámbar pendiente, verde aprobada, gris rechazada |
| Resumen de cantidades | `MatCard` con los tres números |
| Motivo | `MatSelect` con opción "Otro" + `MatInput` |
| Descripción | `MatInput` textarea con contador |
| Diálogos | `MatDialog` (aprobar / rechazar) |
| Rango de fechas | `MatDateRangePicker` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Cantidad | requerida, > 0, ≤ disponible | "Solo puede devolver 27 unidades" |
| Motivo | requerido, máx. 50 | "Indique el motivo de la devolución" |
| Descripción | opcional, máx. 255 | "Máximo 255 caracteres" |

⚠️ El `max` del cliente es orientativo: otra devolución pendiente puede registrarse mientras
tanto. El `422` del servidor es la autoridad (R-03).

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Bandeja vacía | ✅ "No hay devoluciones pendientes de revisión" |
| Vacío con filtro | "Ninguna devolución coincide con los filtros" |
| Sin cantidad disponible | ⚠️ "Esta línea ya fue devuelta por completo" — sin formulario |
| Lote de destino vencido | ⚠️ "El lote de destino venció. La mercancía no volverá como stock vendible." |
| Aprobando | Botón con spinner |
| `422` al enviar | Mensaje del servidor con el disponible actualizado |

La bandeja vacía es **buena noticia**: se muestra en verde, no con el mensaje genérico de
"sin resultados".

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
