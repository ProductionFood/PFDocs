# HU-10 · Consulta de inventario — Diseño

---

## 1. Pantallas

### Inventario de materias primas

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  Inventario de materias primas                                                 │
├────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                         │
│  │      8       │  │      2       │  │      1       │                         │
│  │ materias     │  │ bajo mínimo  │  │ agotadas     │                         │
│  │ primas       │  │     🟠       │  │     🔴       │                         │
│  └──────────────┘  └──────────────┘  └──────────────┘                         │
├────────────────────────────────────────────────────────────────────────────────┤
│  Materia prima ┌──────────────┐  ☑ Solo stock bajo   ☐ Incluir inactivas      │
│                │ Todas      ▾ │                                                │
│                └──────────────┘                                                │
├────────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA     DISPONIBLE   MÍNIMO    NIVEL           FALTA   ACTUALIZADO  │
├────────────────────────────────────────────────────────────────────────────────┤
│  🔴 Levadura         0,00 g     500,00 g  [          ]  500,00 g  ayer 16:40   │
│  🟠 Harina de trigo 12,50 kg     50,00 kg [██        ]   37,50 kg  hoy 09:12   │
│  🟢 Azúcar          45,00 kg     30,00 kg [██████████]        —    12/09 08:30 │
│  🟢 Mantequilla     35,00 kg     20,00 kg [██████████]        —    10/09 11:15 │
└────────────────────────────────────────────────────────────────────────────────┘
```

La barra muestra `disponible / mínimo`. Un vistazo basta para saber qué pedir y cuánto:
la columna "FALTA" ya trae la resta hecha.

**Sin botones de acción**: la pantalla es de consulta (R-01).

### Kardex de una materia prima

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ← Harina de trigo · Movimientos                     Saldo actual: 12,50 kg  │
├──────────────────────────────────────────────────────────────────────────────┤
│  FECHA            MOVIMIENTO           CANT.      ANTES    DESPUÉS   USUARIO │
├──────────────────────────────────────────────────────────────────────────────┤
│  18/09 09:12   ↘ Salida producción   100,00 kg   112,50     12,50   Ismael B.│
│                   Consumo #31                                                │
│  12/09 08:05   ↗ Entrada lote        100,00 kg    12,50    112,50   Carlos C.│
│                   Lote H-2026-0912                                           │
│  05/09 14:22   ↗ Entrada compra       50,00 kg     0,00     12,50   Carlos C.│
│                   Compra #8                                          ⚠        │
└──────────────────────────────────────────────────────────────────────────────┘
```

↗ verde entradas · ↘ rojo salidas. El documento de origen es un enlace a la compra, el lote
o el consumo que generó el movimiento.

Mostrar el saldo antes y después de cada movimiento es lo que permite encontrar **dónde**
se descuadró cuando el conteo físico no coincide.

---

## 2. Flujo

```
Usuario              Frontend                Backend              Base de datos
   │                    │                       │                       │
   │ Abre inventario    │ GET /inventario       │                       │
   ├───────────────────►├──────────────────────►│ SELECT FROM           │
   │                    │                       │ v_inventario_materia_prima
   │                    │                       ├──────────────────────►│
   │                    │◄──────────────────────┤ 200 PageResponse      │
   │  Ve el semáforo    │                       │                       │
   │◄───────────────────┤                       │                       │
   │                    │                       │                       │
   │ Clic en "Harina"   │ GET /inventario/5/movimientos                 │
   ├───────────────────►├──────────────────────►│ SELECT FROM           │
   │                    │                       │ movimientos_inventario_mp
   │                    │◄──────────────────────┤ 200 kardex            │
   │  Ve el historial   │                       │                       │
   │◄───────────────────┤                       │                       │
```

**Esta pantalla nunca escribe.** El inventario cambia únicamente desde HU-09 (lotes), HU-13
(recepción de compra) y HU-23 (consumo de producción).

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas de resumen | `MatCard` con conteos |
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Nivel de stock | `MatProgressBar` con color por `estadoStock` |
| Semáforo | `MatIcon` con color |
| Filtros | `MatSelect` + `MatCheckbox` |
| Kardex | `MatTable` con icono de dirección por movimiento |
| Enlaces al origen | `RouterLink` al documento correspondiente |

---

## 4. Validaciones en el formulario

Esta pantalla **no tiene formularios de datos** (R-01). Solo filtros:

| Filtro | Regla |
|---|---|
| Materia prima | Opcional |
| Solo stock bajo | Booleano |
| Incluir inactivas | Booleano, desactivado por defecto |

En el kardex, el rango de fechas por defecto son los últimos 30 días: el historial crece
rápido y quien lo consulta suele buscar algo reciente.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` + tarjetas en esqueleto |
| Vacío | "No hay materias primas registradas" + enlace a HU-08 |
| Vacío con filtro de stock bajo | ✅ "Todas las materias primas están por encima del mínimo" |
| Kardex vacío | "Esta materia prima no registra movimientos" |
| Error | Mensaje + "Reintentar" |

El vacío del filtro de stock bajo es una **buena noticia**, no una ausencia de datos: se
muestra en verde y con ese texto, no con el mensaje genérico de "sin resultados".

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
