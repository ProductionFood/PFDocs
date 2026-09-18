# HU-08 · Gestión de materias primas — Diseño

---

## 1. Pantallas

### Listado

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Materias primas                                        [ + Nueva materia prima ]│
├─────────────────────────────────────────────────────────────────────────────────┤
│  🔍 ┌──────────────────────┐  Estado ┌───────────┐                              │
│     │ Buscar por nombre    │         │ Activas ▾ │                              │
│     └──────────────────────┘         └───────────┘                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│  NOMBRE ▲          UNIDAD   STOCK MÍN.   DISPONIBLE   COSTO UNIT.   ESTADO  ⋯   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Azúcar            kg        30,00        45,00       $ 2.800,00   ●Activa  ⋮   │
│  Harina de trigo   kg        50,00     ⚠  12,50       $ 3.200,00   ●Activa  ⋮   │
│  Levadura          g        500,00        0,00     🔴 $   180,00   ●Activa  ⋮   │
│  Mantequilla       kg        20,00        35,00       $ 18.500,00  ●Activa  ⋮   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                          Elementos por página: 20 ▾   1–4 de 4   ◀  ▶          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

- ⚠ ámbar: `disponible < stock mínimo`
- 🔴 rojo: `disponible = 0` (agotada)

Mostrar el disponible junto al mínimo convierte el listado en una herramienta de trabajo:
se ve de un vistazo qué hay que comprar, sin ir a otra pantalla.

### Formulario

```
┌──────────────────────────────────────────────────┐
│  Nueva materia prima                        [X]  │
├──────────────────────────────────────────────────┤
│  Nombre *                                        │
│  ┌────────────────────────────────────────────┐  │
│  │ Harina de trigo                            │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  Unidad de medida *                              │
│  ┌────────────────────────────────────────────┐  │
│  │ Kilogramo (kg)                          ▾  │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  Stock mínimo *            Costo unitario *      │
│  ┌──────────────────┐     ┌──────────────────┐   │
│  │ 50,00        kg  │     │ $ 3.200,00   /kg │   │
│  └──────────────────┘     └──────────────────┘   │
│  Se alertará por debajo     Valor de referencia  │
│  de esta cantidad           para costear recetas │
│                                                  │
│              [ Cancelar ]  [ Guardar ]           │
└──────────────────────────────────────────────────┘
```

El sufijo de unidad se actualiza al cambiar el selector. Sin él, es fácil teclear `50`
pensando en kilos cuando la unidad configurada es gramos.

El texto bajo "costo unitario" aclara R-03: no es el precio de compra, es una referencia.

---

## 2. Flujo

```
Administrador        Frontend                  Backend                  Base de datos
     │                  │                         │                          │
     │ Nueva MP         │                         │                          │
     ├─────────────────►│ GET /unidades-medida    │                          │
     │                  │ (desde caché de HU-07)  │                          │
     │ Completa, guarda │                         │                          │
     ├─────────────────►│ POST /materias-primas   │                          │
     │                  ├────────────────────────►│                          │
     │                  │                         │ ── BEGIN ──────────────► │
     │                  │                         │ ¿existe la unidad?       │
     │                  │                         │ INSERT materias_primas   │
     │                  │                         │ INSERT inventario (0)  ← R-01
     │                  │                         │ bitácora                 │
     │                  │                         │ ── COMMIT ─────────────► │
     │                  │◄────────────────────────┤ 201                      │
     │ "Registrada"     │                         │                          │
     │◄─────────────────┤                         │                          │
```

**Las dos inserciones van en la misma transacción.** Si falla la del inventario, tampoco se
crea la materia prima: es preferible que el alta falle a que quede una materia prima
inservible para el resto del sistema.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Alerta de stock | `MatIcon` `warning` / `error` con color |
| Unidad | `MatSelect` (catálogo cacheado de HU-07) |
| Campos numéricos | `MatInput[type=number]` + `MatSuffix` con la unidad |
| Moneda | `MatPrefix` `$` + pipe `currency:'COP'` |
| Ayuda contextual | `MatHint` bajo cada campo numérico |
| Formulario | `MatDialog` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Nombre | requerido, máx. 100 | "El nombre es obligatorio" |
| Unidad | requerida | "Seleccione una unidad de medida" |
| Stock mínimo | requerido, ≥ 0, máx. 2 decimales | "Debe ser 0 o mayor" |
| Costo unitario | requerido, ≥ 0, máx. 2 decimales | "Debe ser 0 o mayor" |

**Cero es válido** en ambos campos numéricos (R-06). El validador debe ser `min(0)`, no
`min(0.01)`.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío sin filtro | "No hay materias primas registradas" + "Registrar la primera" |
| Vacío con filtro | "Ninguna coincide con «xxx»" |
| **Sin unidades disponibles** | "Primero debe registrar unidades de medida" + enlace a HU-07 |
| Error | Mensaje + "Reintentar" |

El caso "sin unidades" importa: sin ellas el formulario no puede completarse, y el usuario
necesita saber a dónde ir en lugar de quedarse ante un desplegable vacío.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
