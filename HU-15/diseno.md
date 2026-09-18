# HU-15 · Historial de compras — Diseño

---

## 1. Pantallas

### Historial con resumen

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Historial de compras                                                            │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Proveedor ┌──────────┐ Estado ┌─────────┐ Período ┌─────────────────────────┐   │
│            │ Todos  ▾ │        │ Todos ▾ │         │ 01/09/2026 – 30/09/2026 │   │
│            └──────────┘        └─────────┘         └─────────────────────────┘   │
├──────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐        │
│  │ GASTO EJECUTADO     │ │ COMPROMETIDO        │ │ COMPRAS             │        │
│  │  $ 4.850.000        │ │  $ 620.000          │ │  11                 │        │
│  │  8 recibidas        │ │  2 pendientes       │ │  1 cancelada        │        │
│  └─────────────────────┘ └─────────────────────┘ └─────────────────────┘        │
├──────────────────────────────────────────────────────────────────────────────────┤
│  GASTO POR PROVEEDOR                                                             │
│  Harinas del Caribe    ████████████████████░░░░░░░░░░░  $ 2.340.000   48,3 %     │
│  Distribuidora Lácteos ███████████████░░░░░░░░░░░░░░░░  $ 1.810.000   37,3 %     │
│  Insumos del Sinú      ██████░░░░░░░░░░░░░░░░░░░░░░░░░  $   700.000   14,4 %     │
├──────────────────────────────────────────────────────────────────────────────────┤
│  N.º   PROVEEDOR                FECHA        ÍTEMS   TOTAL           ESTADO      │
├──────────────────────────────────────────────────────────────────────────────────┤
│  ▸ #12 Harinas del Caribe       18/09/2026     2   $   480.000,00  🟠 Pendiente  │
│  ▾ #11 Distribuidora Lácteos    15/09/2026     4   $ 1.240.000,00  🟢 Recibida   │
│      └─ Leche entera      200,00 L    $ 2.800,00    $   560.000,00               │
│      └─ Mantequilla        40,00 kg   $17.000,00    $   680.000,00               │
│  ▸ #10 Insumos del Sinú         12/09/2026     1   $    85.000,00  ⚪ Cancelada  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Gasto ejecutado y comprometido van separados** (R-04): lo primero es dinero que salió, lo
segundo es dinero pedido pero no recibido. Una sola cifra que los sumara no correspondería
a ninguna realidad contable.

**El detalle se carga al expandir la fila** (R-06), no de antemano.

---

## 2. Flujo

```
Comprador            Frontend                    Backend
    │                   │                           │
    │ Abre historial    │ GET /compras?mes-actual   │
    ├──────────────────►├──────────────────────────►│
    │                   │ GET /compras/resumen      │
    │                   ├──────────────────────────►│
    │                   │ GET /compras/por-proveedor│
    │                   ├──────────────────────────►│
    │                   │◄──────────────────────────┤ (3 respuestas)
    │ Ve el panel       │                           │
    │◄──────────────────┤                           │
    │                   │                           │
    │ Expande #11       │ GET /compras/11/detalles  │
    ├──────────────────►├──────────────────────────►│  ← bajo demanda
    │                   │◄──────────────────────────┤
    │ Ve las líneas     │                           │
    │◄──────────────────┤                           │
```

Las tres consultas iniciales son independientes y pueden lanzarse en paralelo.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas de resumen | `MatCard` con cifras grandes |
| Gasto por proveedor | Barras horizontales (ver la skill de visualización de datos) |
| Tabla expandible | `MatTable` con `multiTemplateDataRows` |
| Rango de fechas | `MatDateRangePicker` |
| Filtros | `MatSelect` |
| Estado | `MatChip` con color |
| Importes | Pipe `currency:'COP'` |

---

## 4. Validaciones en el formulario

Pantalla de consulta: **sin formularios de datos** (R-01). Solo filtros:

| Filtro | Regla |
|---|---|
| Proveedor | Opcional |
| Estado | Opcional, valor canónico |
| Rango de fechas | `fechaInicio <= fechaFin`, validado antes de llamar |

Por defecto: **mes en curso**. Cargar el historial completo en la primera apertura es lento
y casi nunca es lo que se busca.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | Tarjetas en esqueleto + `MatProgressBar` |
| Vacío en el período | "No hay compras en el período seleccionado" + sugerencia de ampliar el rango |
| Vacío con filtros | "Ninguna compra coincide con los filtros" + "Limpiar filtros" |
| Detalle vacío al expandir | "Esta compra no registra ítems" |
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
