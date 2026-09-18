# HU-16 · Creación de pedidos — Diseño

---

## 1. Pantallas

### Listado de pedidos

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Pedidos                                                      [ + Nuevo pedido ]  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Cliente ┌──────────┐ Estado ┌────────────────┐ Período ┌──────────────────────┐ │
│          │ Todos  ▾ │        │ Todos        ▾ │         │ 01/09 – 30/09/2026   │ │
│          └──────────┘        └────────────────┘         └──────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────────────┤
│  N.º   CLIENTE                PEDIDO      ENTREGA      ÍTEMS  TOTAL      ESTADO  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  #48   Panadería El Trigo     18/09/26  🔴 17/09/26      3  $ 45.600  🟠 Pendien│
│  #47   Tienda La Esquina      18/09/26  🟡 HOY           2  $ 12.400  🔵 En prep│
│  #46   Supermercado Central   17/09/26     20/09/26      5  $ 98.000  🔵 En prep│
│  #45   Tienda La Esquina      16/09/26     16/09/26      2  $  8.400  🟢 Entrega│
└──────────────────────────────────────────────────────────────────────────────────┘
```

🔴 entrega vencida · 🟡 entrega hoy. Es la información que se busca al abrir la pantalla
por la mañana: qué hay que sacar hoy y qué ya se retrasó.

### Formulario de pedido

```
┌────────────────────────────────────────────────┐
│  Nuevo pedido                             [X]  │
├────────────────────────────────────────────────┤
│  Cliente *                                     │
│  ┌──────────────────────────────────────────┐  │
│  │ Tienda La Esquina                     ▾  │  │
│  └──────────────────────────────────────────┘  │
│  Solo se muestran clientes activos             │
│                                                │
│  Fecha de entrega                              │
│  ┌──────────────────────┐                      │
│  │ 20/09/2026        📅 │                      │
│  └──────────────────────┘                      │
│  Opcional. No puede ser anterior a hoy.        │
│                                                │
│  ℹ La fecha del pedido se registra             │
│    automáticamente: 18/09/2026                 │
│                                                │
│              [ Cancelar ]  [ Crear pedido ]    │
└────────────────────────────────────────────────┘
```

**No hay campo de fecha de pedido ni de estado**: los pone el servidor (R-01). La nota
informativa muestra cuál será, sin ofrecerla como editable.

### 🔑 Diálogo de cancelación — devuelve stock

```
┌──────────────────────────────────────────────────────────┐
│  ⚠  Cancelar el pedido #47                               │
├──────────────────────────────────────────────────────────┤
│  Al cancelar, los productos volverán al inventario:      │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Pan francés      24 und → lote PF-2026-0917        │  │
│  │ Croissant         6 und → lote CR-2026-0916        │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  El pedido quedará cancelado y no podrá reactivarse.     │
│                                                          │
│         [ Volver ]  [ Cancelar el pedido ]               │
└──────────────────────────────────────────────────────────┘
```

Mostrar a qué lote vuelve cada producto (R-04) hace visible que la operación mueve
inventario. Sin esa información, cancelar parece un cambio de etiqueta.

---

## 2. Flujo

```
Vendedor            Frontend               Backend           InventarioPtService
   │                   │                      │                      │
   │ Nuevo pedido      │ GET /clientes?activo=true                   │
   ├──────────────────►├─────────────────────►│                      │
   │ Selecciona, crea  │ POST /pedidos        │                      │
   ├──────────────────►├─────────────────────►│ ¿cliente activo?     │
   │                   │                      │ INSERT (sin fecha:   │
   │                   │                      │  actúa el DEFAULT)   │
   │                   │◄─────────────────────┤ 201                  │
   │                   │                      │                      │
   │  (HU-17: agrega productos → sale stock)  │                      │
   │                   │                      │                      │
   │ "Cancelar"        │ PATCH /pedidos/47/estado {"CANCELADO"}      │
   ├──────────────────►├─────────────────────►│                      │
   │                   │                      │ ── BEGIN ──────────► │
   │                   │                      │ ¿transición válida?  │
   │                   │                      │ por cada línea:      │
   │                   │                      ├─────────────────────►│ devuelve al lote
   │                   │                      │                      │ INSERT movimiento
   │                   │                      │ UPDATE estado        │
   │                   │                      │ ── COMMIT ─────────► │
   │                   │◄─────────────────────┤ 200                  │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Estado | `MatChip` con color |
| Alerta de entrega | `MatIcon` con color según proximidad |
| Cliente | `MatSelect` (solo activos) |
| Fecha de entrega | `MatDatepicker` con `min` |
| Diálogo de cancelación | `MatDialog` con tabla de devolución |
| Acciones de estado | `MatRaisedButton` / `MatMenu` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Cliente | requerido, **activo** | "Seleccione un cliente" |
| Fecha de entrega | opcional, ≥ fecha del pedido | "No puede ser anterior a hoy" |

**No se validan `fechaPedido` ni `estado`**: no están en el formulario (R-01).

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío | "No hay pedidos registrados" + "Crear el primero" |
| Vacío con filtro | "Ningún pedido coincide con los filtros" |
| Pedido sin productos | ⚠️ "Agregue productos a este pedido" |
| Sin clientes activos | Enlace a HU-05 para registrar uno |
| Cancelando | Botón con spinner; diálogo bloqueado |
| Error | Mensaje del servidor + "Reintentar" |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
