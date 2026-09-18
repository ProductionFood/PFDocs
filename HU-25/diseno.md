# HU-25 · Consulta de pedidos — Diseño

---

## 1. Pantallas

### Consulta de pedidos con resumen

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Consulta de pedidos                                                             │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Cliente ┌──────────┐ Estado ┌────────────┐ Período ┌─────────────────────────┐  │
│          │ Todos  ▾ │        │ Todos    ▾ │         │ 01/09/2026 – 30/09/2026 │  │
│          └──────────┘        └────────────┘         └─────────────────────────┘  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐        │
│  │ VENTA NETA    │ │ COMPROMETIDO  │ │ DEVOLUCIONES  │ │ TICKET PROM.  │        │
│  │ $ 1.778.000   │ │ $  320.000    │ │ $   62.000    │ │ $   76.667    │        │
│  │ 24 entregados │ │ 6 en curso    │ │ 3,4 % 🟠      │ │               │        │
│  └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘        │
├──────────────────────────────────────────────────────────────────────────────────┤
│  VENTAS POR CLIENTE                    │  PRODUCTOS MÁS VENDIDOS                 │
│  Supermercado Cen. ███████████ $842.000│  Pan francés      1.240 und  $1.488.000 │
│  Panadería El Trigo ██████     $521.000│  Croissant          320 und  $  800.000 │
│  Tienda La Esquina  ████       $415.000│  Torta de zanah.     28 und  $  896.000 │
├──────────────────────────────────────────────────────────────────────────────────┤
│  N.º   CLIENTE              PEDIDO     ENTREGA    BRUTO      DEVOL.   NETO  EST. │
├──────────────────────────────────────────────────────────────────────────────────┤
│ ▸ #48  Panadería El Trigo   18/09/26  17/09/26  $ 45.600      —    $45.600 🟠   │
│ ▾ #47  Tienda La Esquina    18/09/26  HOY       $ 51.000  $ 3.600  $47.400 🔵   │
│     └─ Pan francés    30 und × $1.200 = $36.000   (3 devueltos)                  │
│     └─ Croissant       6 und × $2.500 = $15.000                                  │
│ ▸ #45  Tienda La Esquina    16/09/26  16/09/26  $  8.400      —    $ 8.400 🟢   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Las tres columnas de importe** (bruto, devoluciones, neto) hacen visible el porcentaje de
devolución (R-04), que es un indicador de calidad. Un 3,4% de devoluciones merece atención
aunque la venta neta sea buena.

El ranking de productos va **ordenado por unidades** (R-07): la torta de zanahoria factura
más que el croissant, pero se venden 28 unidades frente a 320. Son dos preguntas distintas
y la columna de importe permite responder ambas.

---

## 2. Flujo

```
Usuario              Frontend                    Backend
   │                    │                           │
   │ Abre consulta      │ GET /pedidos?mes-actual   │
   ├───────────────────►├──────────────────────────►│
   │                    │ GET /pedidos/resumen      │
   │                    ├──────────────────────────►│
   │                    │ GET /pedidos/por-cliente  │
   │                    ├──────────────────────────►│
   │                    │ GET /pedidos/productos-mas-vendidos
   │                    ├──────────────────────────►│
   │                    │◄──────────────────────────┤ (4 respuestas)
   │  Ve el panel       │                           │
   │◄───────────────────┤                           │
   │                    │                           │
   │ Expande #47        │ GET /pedidos/47/detalles  │  ← bajo demanda
   ├───────────────────►├──────────────────────────►│
   │                    │◄──────────────────────────┤
```

Las cuatro consultas iniciales son independientes y se lanzan en paralelo.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas | `MatCard` con cifras grandes |
| Ventas por cliente | Barras horizontales |
| Productos más vendidos | `MatTable` compacta |
| Tabla expandible | `MatTable` con `multiTemplateDataRows` |
| Rango de fechas | `MatDateRangePicker` |
| Estado | `MatChip` con color |
| Importes | Pipe `currency:'COP'` |

Antes de escribir los gráficos, cargar la guía de visualización de datos del proyecto: la
paleta debe coincidir con la de HU-24 y HU-26.

---

## 4. Validaciones en el formulario

Pantalla de consulta: **sin formularios de datos** (R-01). Solo filtros:

| Filtro | Regla |
|---|---|
| Cliente | Opcional |
| Estado | Opcional, valor canónico |
| Rango de fechas | `fechaInicio <= fechaFin` |

Por defecto, **mes en curso**.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | Tarjetas en esqueleto |
| Vacío en el período | "No hay pedidos en el período seleccionado" |
| Vacío con filtros | "Ningún pedido coincide" + "Limpiar filtros" |
| Sin devoluciones | `—` en la columna, no `$ 0` |
| Error en un bloque | Ese bloque falla; los demás siguen |

Mostrar `—` en lugar de `$ 0` en devoluciones distingue "no hubo" de "hubo por cero".

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
