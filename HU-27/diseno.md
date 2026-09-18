# HU-27 · Alerta de stock bajo — Diseño

---

## 1. Pantallas

### Alertas de stock

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│  Alertas de stock                                              ☐ Incluir inactivas│
├───────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐ ┌───────────────────────┐               │
│  │        3        │ │        1        │ │   $ 359.388           │               │
│  │  en alerta      │ │  agotada  🔴    │ │  costo estimado de    │               │
│  │                 │ │                 │ │  reposición           │               │
│  └─────────────────┘ └─────────────────┘ └───────────────────────┘               │
├───────────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA      DISPONIBLE   MÍNIMO    NIVEL           FALTA      REPOSICIÓN  │
├───────────────────────────────────────────────────────────────────────────────────┤
│  🔴 Levadura           0,00 kg    5,00 kg  [          ]   5,00 kg    $   1.080    │
│     AGOTADA · bloquea la producción                                               │
│  🟠 Mantequilla        4,50 kg   20,00 kg  [██        ]  15,50 kg    $ 344.100    │
│  🟠 Harina de trigo   46,30 kg   50,00 kg  [█████████ ]   3,70 kg    $  14.208    │
├───────────────────────────────────────────────────────────────────────────────────┤
│                                            [ Ver sugerencia de compra → ]         │
└───────────────────────────────────────────────────────────────────────────────────┘
```

**Las agotadas van primero aunque les falte menos** (R-03): la levadura bloquea la
producción hoy; a la mantequilla le falta más pero todavía permite trabajar.

La leyenda "bloquea la producción" bajo las agotadas explica por qué encabezan la lista.

### Sugerencia de compra

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ← Sugerencia de compra                                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  Agrupada por el proveedor de la última compra de cada insumo.          │
│  Las cantidades incluyen un margen del 20% sobre el faltante.           │
├─────────────────────────────────────────────────────────────────────────┤
│  📦 Harinas del Caribe S.A.S.                                           │
│     Harina de trigo    falta 3,70 kg  →  comprar 4,44 kg   $  14.208    │
│                                            Total: $ 14.208              │
│                                          [ Crear compra a este prov. ]  │
├─────────────────────────────────────────────────────────────────────────┤
│  📦 Distribuidora Lácteos                                               │
│     Mantequilla       falta 15,50 kg  →  comprar 18,60 kg  $ 344.100    │
│                                            Total: $ 344.100             │
│                                          [ Crear compra a este prov. ]  │
├─────────────────────────────────────────────────────────────────────────┤
│  ⚠ Levadura no tiene compras previas registradas.                       │
│     Seleccione un proveedor manualmente.       [ Crear compra ]         │
└─────────────────────────────────────────────────────────────────────────┘
```

**"Crear compra" prellena el formulario de HU-13** con el proveedor y los ítems sugeridos.
Es lo que cierra el ciclo: de detectar el problema a resolverlo sin volver a teclear nada.

El margen del 20% (R-06) evita el efecto de comprar justo el faltante y volver a estar en
alerta al primer consumo.

### Indicador global

```
┌────────────────────────────────────────────────────────────────────────┐
│ ☰  ProductionFood         🔴 3 alertas    Carlos C. · COMPRAS     [⏻]  │
└────────────────────────────────────────────────────────────────────────┘
```

Visible desde cualquier pantalla. Rojo si hay alguna agotada, ámbar si solo hay bajas.

### Sin alertas — una buena noticia

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│                            ✅                                 │
│                                                               │
│        Todas las materias primas están por encima             │
│                  de su stock mínimo                           │
│                                                               │
│              Última verificación: 18/09/2026 10:42            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

En verde y con un mensaje afirmativo. El mensaje genérico de "no se encontraron resultados"
haría dudar de si la consulta funcionó.

---

## 2. Flujo

```
Usuario              Frontend                  Backend
   │                    │                         │
   │ (cualquier pantalla)│ GET /alertas/resumen   │
   │                    ├────────────────────────►│ conteo ligero
   │                    │◄────────────────────────┤ {total:3, agotadas:1}
   │  Ve 🔴 3 alertas    │                         │
   │◄───────────────────┤                         │
   │                    │                         │
   │ Clic en el badge   │ GET /inventario/alertas │
   ├───────────────────►├────────────────────────►│ SELECT FROM
   │                    │                         │ v_inventario_materia_prima
   │                    │                         │ WHERE estado_stock IN (...)
   │                    │                         │ ORDER BY FIELD(...), faltante DESC
   │                    │◄────────────────────────┤ 200
   │  Ve la lista       │                         │
   │◄───────────────────┤                         │
   │                    │                         │
   │ "Sugerencia"       │ GET .../sugerencia-compra
   ├───────────────────►├────────────────────────►│ agrupa por proveedor
   │                    │◄────────────────────────┤ de la última compra
   │                    │                         │
   │ "Crear compra"     │ → HU-13 con el formulario prellenado
   ├───────────────────►│                         │
   │                                              │
   └─── El ciclo se cierra: la recepción sumará stock y la alerta desaparecerá
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas de resumen | `MatCard` |
| Tabla | `MatTable` ordenada por urgencia |
| Nivel de stock | `MatProgressBar` con color |
| Nivel | `MatChip` — rojo `AGOTADO`, ámbar `BAJO` |
| Indicador global | `MatBadge` sobre `MatIcon` en `MatToolbar` |
| Sugerencia | `MatExpansionPanel` por proveedor |
| Crear compra | `MatRaisedButton` con navegación y estado prellenado |
| Estado vacío | `MatIcon` `check_circle` verde |

---

## 4. Validaciones en el formulario

Pantalla de consulta: **sin formularios de datos**. Un solo filtro:

| Filtro | Regla |
|---|---|
| Incluir inactivas | Booleano, **desactivado por defecto** (R-05) |

El `stock_minimo` que define cuándo alertar se configura en HU-08, no aquí.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` + tarjetas en esqueleto |
| **Sin alertas** | ✅ Verde: "Todas las materias primas están por encima del mínimo" |
| Con alertas | Tabla ordenada, agotadas primero |
| Sin compras previas de un insumo | ⚠️ En la sugerencia, con opción de elegir proveedor |
| Error | Mensaje + "Reintentar" |

**El estado vacío es el caso deseable**, y debe comunicarlo: verde, afirmativo y con la hora
de la verificación. Un "sin resultados" genérico transmite lo contrario.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
