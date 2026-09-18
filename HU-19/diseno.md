# HU-19 · Lotes de producto terminado — Diseño

---

## 1. Pantallas

### Lotes en estante

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│  Lotes de producto terminado                                       [ + Nuevo lote ] │
├────────────────────────────────────────────────────────────────────────────────────┤
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐                             │
│  │      12       │ │       3       │ │       1       │                             │
│  │ lotes con     │ │ vencen en     │ │ vencido       │                             │
│  │ stock  🟢     │ │ ≤ 3 días 🟠   │ │ con stock 🔴  │                             │
│  └───────────────┘ └───────────────┘ └───────────────┘                             │
├────────────────────────────────────────────────────────────────────────────────────┤
│  Producto ┌────────────┐ ☑ Solo con stock  ☐ Incluir vencidos                      │
│           │ Todos    ▾ │                                                            │
│           └────────────┘                                                            │
├────────────────────────────────────────────────────────────────────────────────────┤
│  CÓDIGO          PRODUCTO       PRODUCCIÓN  VENCE        INGRESÓ  DISPONIBLE  EST. │
├────────────────────────────────────────────────────────────────────────────────────┤
│  CR-2026-0914    Croissant      14/09/26    16/09/26      30 und     8 und   🔴    │
│  PF-2026-0915    Pan francés    15/09/26    19/09/26     120 und     0 und   🟠    │
│  PF-2026-0917    Pan francés    17/09/26    21/09/26      60 und    40 und   🟠 3d │
│  PF-2026-0918    Pan francés    18/09/26    22/09/26      25 und    25 und   🟡 4d │
│  TZ-2026-0916    Torta zanah…   16/09/26    23/09/26       5 und     3 und   🟡 5d │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**INGRESÓ y DISPONIBLE son columnas distintas** (R-03): la diferencia es lo vendido.
`PF-2026-0915` ingresó con 120 y está agotado; `CR-2026-0914` está vencido y **aún tiene 8
unidades en el estante** — hay que retirarlas físicamente.

La tarjeta roja "vencido con stock" es la que exige acción inmediata.

### Formulario

```
┌──────────────────────────────────────────────────────┐
│  Nuevo lote de producto terminado               [X]  │
├──────────────────────────────────────────────────────┤
│  Código de lote *                                    │
│  ┌────────────────────────────────────────────────┐  │
│  │ PF-2026-0918                                   │  │
│  └────────────────────────────────────────────────┘  │
│  Sugerencia: prefijo del producto + fecha            │
│                                                      │
│  Producto *                                          │
│  ┌────────────────────────────────────────────────┐  │
│  │ Pan francés (und)                           ▾  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Fecha producción *      Fecha vencimiento *         │
│  ┌──────────────────┐   ┌──────────────────┐        │
│  │ 18/09/2026    📅 │   │ 22/09/2026    📅 │        │
│  └──────────────────┘   └──────────────────┘        │
│                          No puede ser anterior a hoy │
│                                                      │
│  Cantidad producida *                                │
│  ┌──────────────────┐                                │
│  │ 25           und │                                │
│  └──────────────────┘                                │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ ℹ Este lote quedará disponible para la venta.  │  │
│  │   Stock de Pan francés:  77 und → 102 und      │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│              [ Cancelar ]  [ Registrar lote ]        │
└──────────────────────────────────────────────────────┘
```

El recuadro final conecta esta historia con HU-17: lo que se registra aquí es lo que se
puede vender allí.

### Kardex de un lote

```
┌───────────────────────────────────────────────────────────────────────────┐
│  ← Lote PF-2026-0917 · Pan francés          Vence 21/09/2026 (en 3 días)  │
│     Ingresó: 60 und    ·    Disponible: 40 und    ·    Vendido: 20 und    │
├───────────────────────────────────────────────────────────────────────────┤
│  FECHA         MOVIMIENTO            CANT.    ANTES   DESPUÉS   ORIGEN    │
├───────────────────────────────────────────────────────────────────────────┤
│  18/09 11:20 ↗ Entrada devolución     3 und    37       40    Devol. #15 │
│  18/09 09:45 ↘ Salida venta          18 und    55       37    Pedido #47 │
│  17/09 16:10 ↘ Salida venta           5 und    60       55    Pedido #46 │
│  17/09 08:00 ↗ Entrada producción    60 und     0       60    Lote       │
└───────────────────────────────────────────────────────────────────────────┘
```

Esta pantalla es la trazabilidad completa del lote: **qué clientes recibieron producto de
él**. Es la información que se necesita ante un retiro sanitario.

---

## 2. Flujo

```
Productor           Frontend             Backend          InventarioPtService
   │                   │                    │                     │
   │ Completa el form  │                    │                     │
   ├──────────────────►│ POST /lotes-producto                     │
   │                   ├───────────────────►│                     │
   │                   │                    │ ── BEGIN ─────────► │
   │                   │                    │ ¿código único?      │
   │                   │                    │ ¿producto existe?   │
   │                   │                    │ ¿fechas válidas?    │
   │                   │                    │ ¿no está vencido?   │
   │                   │                    │ INSERT lotes        │
   │                   │                    │ INSERT inventario_pt│  ← CA-05
   │                   │                    ├────────────────────►│
   │                   │                    │                     │ INSERT movimiento
   │                   │                    │                     │ ENTRADA_PRODUCCION
   │                   │                    │ bitácora            │
   │                   │                    │ ── COMMIT ────────► │
   │                   │◄───────────────────┤ 201                 │
   │ "77 → 102 und"    │                    │                     │
   │◄──────────────────┤                    │                     │
   │                                                              │
   └── El producto ya se puede vender en HU-17 ───────────────────┘
```

**Las tres inserciones son una sola operación** (R-02): un lote sin inventario no existe
para el resto del sistema.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas de resumen | `MatCard` con conteos por estado |
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Semáforo | `MatChip` con color y días restantes |
| Producto | `MatSelect` |
| Fechas | `MatDatepicker` con `min` |
| Impacto en stock | `MatCard` destacada, en vivo |
| Kardex | `MatTable` con iconos de dirección |
| Filtros | `MatCheckbox` + `MatSelect` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Código de lote | requerido, máx. 30, único | "El código es obligatorio" |
| Producto | requerido | "Seleccione un producto" |
| Fecha producción | requerida | "Indique la fecha de producción" |
| Fecha vencimiento | requerida, ≥ producción, **≥ hoy** | "No puede registrar un lote ya vencido" |
| Cantidad | requerida, > 0 | "Debe ser mayor que cero" |

A diferencia de los lotes de materia prima (HU-09 R-05), **aquí sí se bloquea el
vencimiento pasado** (R-05): un lote de producto terminado que nace vencido no se puede
vender, así que registrarlo no tiene utilidad.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | Tarjetas en esqueleto + `MatProgressBar` |
| Vacío | "No hay lotes registrados" + "Registrar el primero. Sin lotes no se puede vender." |
| Vacío con filtro | "Ningún lote coincide con los filtros" |
| **Vencidos con stock** | 🔴 Destacado: "N lotes vencidos aún tienen unidades en estante" |
| Vencimiento pasado en el formulario | Error bloqueante (R-05) |
| Guardado | Confirmación con el stock resultante del producto |
| Error | Mensaje del servidor + "Reintentar" |

El estado vacío dice **la consecuencia** —sin lotes no hay nada que vender—, que es lo que
conecta esta historia con HU-17.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
