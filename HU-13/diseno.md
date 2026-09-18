# HU-13 · Registro de compras — Diseño

---

## 1. Pantallas

### Listado de compras

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Compras                                                  [ + Nueva compra ]  │
├──────────────────────────────────────────────────────────────────────────────┤
│  Proveedor ┌──────────┐  Estado ┌────────────┐  Desde ┌────────┐ Hasta ┌────┐│
│            │ Todos  ▾ │         │ Todos    ▾ │        │01/09/26│       │18/…││
│            └──────────┘         └────────────┘        └────────┘       └────┘│
├──────────────────────────────────────────────────────────────────────────────┤
│  N.º    PROVEEDOR                 FECHA       ÍTEMS   TOTAL          ESTADO  │
├──────────────────────────────────────────────────────────────────────────────┤
│  #12    Harinas del Caribe        18/09/2026    2   $ 480.000,00   🟠 Pendien│
│  #11    Distribuidora Lácteos     15/09/2026    4   $ 1.240.000,00 🟢 Recibid│
│  #10    Insumos del Sinú          12/09/2026    1   $  85.000,00   ⚪ Cancela│
└──────────────────────────────────────────────────────────────────────────────┘
```

### 🔑 Diálogo de recepción — la pantalla más importante de la historia

```
┌────────────────────────────────────────────────────────────┐
│  ⚠  Confirmar recepción de la compra #12                   │
├────────────────────────────────────────────────────────────┤
│  Al recibir esta compra, el inventario quedará así:        │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Harina de trigo    + 100,00 kg    12,50 → 112,50 kg  │  │
│  │ Azúcar             +  50,00 kg    45,00 →  95,00 kg  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  🔒 Esta acción no se puede deshacer.                      │
│     Una vez recibida, la compra no vuelve a estado         │
│     pendiente. Para corregir un error será necesario       │
│     registrar un ajuste de inventario.                     │
│                                                            │
│              [ Cancelar ]  [ Confirmar recepción ]         │
└────────────────────────────────────────────────────────────┘
```

Este diálogo hace tres cosas que importan:

1. **Muestra el efecto antes de aplicarlo**, línea por línea.
2. **Advierte que es irreversible** (R-02), con el motivo.
3. Convierte una acción de menú en una decisión consciente.

Es la operación que más altera el estado del sistema; un clic distraído no debería poder
dispararla.

### Detalle de compra

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ← Compra #12                Harinas del Caribe S.A.S.        🟠 Pendiente   │
│     Fecha: 18/09/2026                                                        │
│                                        [ Cancelar compra ]  [ ✓ Recibir ]    │
├──────────────────────────────────────────────────────────────────────────────┤
│  ÍTEMS DE LA COMPRA                                      [ + Agregar ítem ]  │
├──────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA        CANTIDAD      PRECIO UNIT.      SUBTOTAL          ⋯    │
├──────────────────────────────────────────────────────────────────────────────┤
│  Harina de trigo      100,00 kg     $ 3.200,00     $ 320.000,00      ✏️ 🗑️  │
│  Azúcar                50,00 kg     $ 3.200,00     $ 160.000,00      ✏️ 🗑️  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                      TOTAL:        $ 480.000,00              │
└──────────────────────────────────────────────────────────────────────────────┘
```

Cuando la compra está `RECIBIDA` o `CANCELADA`, los botones de edición del detalle
**desaparecen** (R-04), no se muestran deshabilitados: es más claro.

---

## 2. Flujo

```
Comprador           Frontend              Backend            InventarioService
    │                  │                     │                      │
    │ "Recibir"        │                     │                      │
    ├─────────────────►│ (abre diálogo con   │                      │
    │                  │  el impacto previsto)│                     │
    │ Confirma         │                     │                      │
    ├─────────────────►│ PATCH /compras/12/estado {"RECIBIDA"}      │
    │                  ├────────────────────►│                      │
    │                  │                     │ ── BEGIN ──────────► │
    │                  │                     │ ¿transición válida?  │
    │                  │                     │ ¿tiene detalle?      │
    │                  │                     │                      │
    │                  │                     │ por cada línea:      │
    │                  │                     ├─────────────────────►│ FOR UPDATE
    │                  │                     │                      │ UPDATE inventario
    │                  │                     │                      │ INSERT movimiento
    │                  │                     │◄─────────────────────┤
    │                  │                     │ UPDATE compras.estado│
    │                  │                     │ bitácora RECEPCION   │
    │                  │                     │ ── COMMIT ─────────► │
    │                  │◄────────────────────┤ 200 + movimientos    │
    │ "Stock actualizado"                    │                      │
    │◄─────────────────┤                     │                      │
```

**Si falla cualquier línea, se revierte todo** (R-01): no hay recepciones a medias.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla de compras | `MatTable` + `MatSort` + `MatPaginator` |
| Estado | `MatChip` con color |
| Filtros de fecha | `MatDateRangePicker` |
| Proveedor / estado | `MatSelect` |
| Diálogo de recepción | `MatDialog` con tabla de impacto y aviso destacado |
| Acciones de estado | `MatRaisedButton` (Recibir) · `MatButton` (Cancelar) |
| Total | Pipe `currency:'COP'` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Proveedor | requerido, activo | "Seleccione un proveedor" |
| Fecha | requerida, **no futura** | "La fecha no puede ser posterior a hoy" |
| Estado | requerido, valor canónico | — |

Al crear, el estado es siempre `PENDIENTE`: no se ofrece como opción editable. Los cambios
posteriores pasan por el flujo de estados, no por el formulario.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío | "No hay compras registradas" + "Registrar la primera" |
| Vacío con filtro | "Ninguna compra coincide con los filtros" |
| Compra sin ítems | ⚠️ "Agregue al menos un ítem antes de recibir" (R-03) |
| Recibiendo | Botón con spinner; diálogo bloqueado |
| Recibida | Confirmación con los saldos resultantes |
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
