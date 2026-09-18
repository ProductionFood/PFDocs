# HU-09 · Lotes de materia prima — Diseño

---

## 1. Pantallas

### Listado de lotes

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  Lotes de materia prima                                        [ + Nuevo lote ] │
├────────────────────────────────────────────────────────────────────────────────┤
│  Materia prima ┌────────────────┐  ☐ Solo vencidos  ☐ Vencen en 7 días        │
│                │ Todas        ▾ │                                              │
│                └────────────────┘                                              │
├────────────────────────────────────────────────────────────────────────────────┤
│  CÓDIGO          MATERIA PRIMA     PRODUCCIÓN   VENCIMIENTO   CANTIDAD  ESTADO │
├────────────────────────────────────────────────────────────────────────────────┤
│  H-2026-0912     Harina de trigo   10/09/2026   10/03/2027    100,00 kg  🟢    │
│  LV-2026-0905    Levadura          05/09/2026   25/09/2026    500,00 g   🟠 7d │
│  MQ-2026-0820    Mantequilla       20/08/2026   15/09/2026     20,00 kg  🔴    │
│  AZ-2026-0901    Azúcar            01/09/2026   01/09/2027     50,00 kg  🟢    │
└────────────────────────────────────────────────────────────────────────────────┘
```

🔴 vencido · 🟠 vence en ≤ 7 días · ⚪ ≤ 30 días · 🟢 vigente

Los lotes vencidos **se muestran** (R-05): siguen físicamente en la bodega hasta que alguien
decida darlos de baja.

### Formulario — con el impacto en inventario visible

```
┌──────────────────────────────────────────────────────┐
│  Nuevo lote de materia prima                    [X]  │
├──────────────────────────────────────────────────────┤
│  Código de lote *                                    │
│  ┌────────────────────────────────────────────────┐  │
│  │ H-2026-0912                                    │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Materia prima *                                     │
│  ┌────────────────────────────────────────────────┐  │
│  │ Harina de trigo (kg)                        ▾  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Fecha producción *      Fecha vencimiento *         │
│  ┌──────────────────┐   ┌──────────────────┐        │
│  │ 10/09/2026    📅 │   │ 10/03/2027    📅 │        │
│  └──────────────────┘   └──────────────────┘        │
│                                                      │
│  Cantidad *                                          │
│  ┌──────────────────┐                                │
│  │ 100,00       kg  │                                │
│  └──────────────────┘                                │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │ ℹ Impacto en el inventario                   │    │
│  │   Harina de trigo:  12,50 kg  →  112,50 kg   │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│              [ Cancelar ]  [ Registrar lote ]        │
└──────────────────────────────────────────────────────┘
```

**El recuadro de impacto se actualiza en vivo.** Es lo que evita que alguien registre un
lote de una compra que ya sumó, sin darse cuenta (R-02).

---

## 2. Flujo

```
Usuario            Frontend              Backend              InventarioService
   │                  │                     │                        │
   │ Completa el form │                     │                        │
   ├─────────────────►│ POST /lotes-materia-prima                    │
   │                  ├────────────────────►│                        │
   │                  │                     │ ── BEGIN ───────────►  │
   │                  │                     │ ¿código único?         │
   │                  │                     │ ¿MP existe?            │
   │                  │                     │ ¿fechas válidas?       │
   │                  │                     │ INSERT lotes           │
   │                  │                     │                        │
   │                  │                     │ ¿idDetalleCompra?      │
   │                  │                     │   SÍ → no suma  ← R-02 │
   │                  │                     │   NO → registrarEntrada│
   │                  │                     ├───────────────────────►│
   │                  │                     │                        │ SELECT ... FOR UPDATE
   │                  │                     │                        │ UPDATE inventario
   │                  │                     │                        │ INSERT movimiento
   │                  │                     │◄───────────────────────┤
   │                  │                     │ bitácora               │
   │                  │                     │ ── COMMIT ──────────►  │
   │                  │◄────────────────────┤ 201 + saldos           │
   │ "12,50 → 112,50" │                     │                        │
   │◄─────────────────┤                     │                        │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Semáforo de vencimiento | `MatChip` con color + días restantes |
| Fechas | `MatDatepicker` (el de vencimiento con `min` atado a producción) |
| Materia prima | `MatSelect` |
| Cantidad | `MatInput[type=number]` + `MatSuffix` con la unidad |
| Impacto en inventario | `MatCard` destacada, actualizada en vivo |
| Filtros booleanos | `MatCheckbox` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Código de lote | requerido, máx. 30 | "El código es obligatorio" |
| Materia prima | requerida | "Seleccione una materia prima" |
| Fecha producción | requerida | "Indique la fecha de producción" |
| Fecha vencimiento | requerida, ≥ producción | "No puede ser anterior a la producción" |
| Cantidad | requerida, > 0 | "Debe ser mayor que cero" |

**La unicidad del código solo la verifica el backend.** Una fecha de vencimiento en el
pasado genera advertencia, no error (R-05).

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío sin filtro | "No hay lotes registrados" + "Registrar el primero" |
| Vacío con filtro | "Ningún lote coincide con los filtros" |
| Vencimiento pasado | Advertencia ámbar en el formulario, sin bloquear |
| Error | Mensaje + "Reintentar" |
| Guardado | Confirmación con el saldo anterior y posterior |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
