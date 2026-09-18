# HU-17 · Agregar productos al pedido — Diseño

---

## 1. Pantallas

### Detalle del pedido

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│  PRODUCTOS DEL PEDIDO #47                                    [ + Agregar producto ] │
├────────────────────────────────────────────────────────────────────────────────────┤
│  PRODUCTO       CANTIDAD   PRECIO UNIT.   SUBTOTAL     LOTES CONSUMIDOS        ⋯   │
├────────────────────────────────────────────────────────────────────────────────────┤
│  Pan francés    30 und     $ 1.200,00   $ 36.000,00   PF-…0915 (12) 🟠          ✏️🗑️│
│                                                        PF-…0917 (18)               │
│  Croissant       6 und     $ 2.500,00   $ 15.000,00   CR-…0916 (6)              ✏️🗑️│
├────────────────────────────────────────────────────────────────────────────────────┤
│                                          TOTAL:       $ 51.000,00                  │
└────────────────────────────────────────────────────────────────────────────────────┘
```

La columna **LOTES CONSUMIDOS** es la trazabilidad (R-10): permite saber después qué lote
fue a qué cliente, dato indispensable ante un retiro sanitario. El 🟠 marca un lote próximo
a vencer.

### 🔑 Agregar producto — con previsualización FEFO

```
┌──────────────────────────────────────────────────────────┐
│  Agregar producto al pedido                         [X]  │
├──────────────────────────────────────────────────────────┤
│  Producto *                                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Pan francés — 77 disponibles                    ▾  │  │
│  └────────────────────────────────────────────────────┘  │
│  Solo se muestran productos con stock disponible         │
│                                                          │
│  Cantidad *                    Precio unitario           │
│  ┌──────────────────┐         $ 1.200,00                 │
│  │ 30           und │         (precio actual)            │
│  └──────────────────┘                                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Se tomará de estos lotes (el que vence primero):  │  │
│  │                                                    │  │
│  │  🟠 PF-2026-0915    12 und    vence en 1 día       │  │
│  │     PF-2026-0917    18 und    vence en 3 días      │  │
│  │                                                    │  │
│  │  Quedarán 47 unidades disponibles                  │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│                     Subtotal:          $ 36.000,00       │
│                                                          │
│              [ Cancelar ]  [ Agregar ]                   │
└──────────────────────────────────────────────────────────┘
```

**El recuadro de lotes se recalcula al cambiar la cantidad.** Hace tres cosas útiles:

1. Enseña que el sistema aplica FEFO y qué mercancía va a salir físicamente.
2. Avisa si se está despachando producto próximo a vencer.
3. Muestra el stock resultante, evitando la sorpresa del `409`.

### Sin stock suficiente

```
┌──────────────────────────────────────────────────────────┐
│  Cantidad *                                              │
│  ┌──────────────────┐                                    │
│  │ 100          und │  ⚠                                 │
│  └──────────────────┘                                    │
│  ⚠ Solo hay 77 unidades disponibles.                     │
│    Faltan 23 unidades para completar el pedido.          │
│                                                          │
│              [ Cancelar ]  [ Agregar ]  ← deshabilitado  │
└──────────────────────────────────────────────────────────┘
```

El mensaje dice cuánto falta, no solo que no alcanza.

---

## 2. Flujo

```
Vendedor          Frontend            Backend          InventarioPtService
   │                 │                   │                     │
   │ Elige producto  │ GET /productos/3/stock                  │
   ├────────────────►├──────────────────►│                     │
   │                 │◄──────────────────┤ 77 und, 3 lotes     │
   │ Escribe 30      │ (previsualiza FEFO en el cliente)       │
   │ Confirma        │ POST /pedidos/47/detalles               │
   ├────────────────►├──────────────────►│                     │
   │                 │                   │ ── BEGIN ─────────► │
   │                 │                   │ ¿pedido editable?   │
   │                 │                   │ ¿producto activo?   │
   │                 │                   │ ¿duplicado?         │
   │                 │                   ├────────────────────►│ SELECT v_lotes_fefo
   │                 │                   │                     │ ORDER BY vencimiento ← R-02
   │                 │                   │                     │ SELECT ... FOR UPDATE
   │                 │                   │                     │ ¿stock total suficiente?
   │                 │                   │                     │ descuenta lote 1 (12)
   │                 │                   │                     │ descuenta lote 2 (18)
   │                 │                   │                     │ INSERT 2 movimientos
   │                 │                   │◄────────────────────┤ lotes consumidos
   │                 │                   │ copia precio ← R-04 │
   │                 │                   │ INSERT detalle_pedido
   │                 │                   │ bitácora            │
   │                 │                   │ ── COMMIT ────────► │
   │                 │◄──────────────────┤ 201 + lotesConsumidos
   │ Ve la línea     │                   │                     │
   │◄────────────────┤                   │                     │
```

**Si falla cualquier paso, ningún lote queda descontado** (CP-22).

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla del detalle | `MatTable` con fila de totales |
| Lotes consumidos | `MatChip` compactos con `MatTooltip` |
| Agregar | `MatDialog` |
| Producto | `MatSelect` con stock en la etiqueta |
| Previsualización FEFO | `MatCard` destacada, recalculada en vivo |
| Alerta de vencimiento | `MatIcon` ámbar + días restantes |
| Cantidad | `MatInput[type=number]` con `max` = disponible |
| Importes | Pipe `currency:'COP'` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Producto | requerido, activo, con stock, no repetido | "Seleccione un producto" |
| Cantidad | requerida, > 0, ≤ disponible | "Solo hay 77 unidades disponibles" |

⚠️ **El `max` del cliente es orientativo.** Entre la consulta de stock y el envío, otro
vendedor puede haber tomado unidades. El `409 STOCK_INSUFICIENTE` del servidor es la
autoridad (R-05), y hay que manejarlo mostrando el disponible actualizado.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando stock | Spinner junto al selector |
| Sin productos con stock | "No hay productos disponibles en inventario" + enlace a HU-19 |
| Cantidad excede el stock | ⚠️ con el disponible y el faltante; botón deshabilitado |
| Lote próximo a vencer | 🟠 con los días restantes en la previsualización |
| Pedido no editable | 🔒 Tabla en modo lectura, sin controles (R-09) |
| `409` al enviar | "El stock cambió mientras completaba el formulario. Disponible ahora: N" |
| Error | Mensaje del servidor + "Reintentar" |

El mensaje del `409` explica **por qué** cambió: es lo que evita que el vendedor crea que el
sistema falló.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
