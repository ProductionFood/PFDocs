# HU-20 · Plan de producción — Diseño

---

## 1. Pantallas

### Listado de planes

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Planes de producción                                            [ + Nuevo plan ] │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Estado ┌──────────────┐ Responsable ┌────────────┐ Período ┌──────────────────┐ │
│         │ Todos      ▾ │             │ Todos    ▾ │         │ 15/09 – 30/09/26 │ │
│         └──────────────┘             └────────────┘         └──────────────────┘ │
├──────────────────────────────────────────────────────────────────────────────────┤
│  N.º   PRODUCCIÓN      RESPONSABLE       PRODUCTOS   AVANCE        ESTADO        │
├──────────────────────────────────────────────────────────────────────────────────┤
│  #10   🟡 HOY 19/09    Ismael Batalla        3      ░░░░░░░░░░   ⚪ Planificado  │
│  #9    18/09/2026      Ismael Batalla        2      ███████░░░   🔵 En proceso   │
│  #8    17/09/2026      Natalia Góngora       4      ██████████   🟢 Completado   │
│  #7    16/09/2026      Ismael Batalla        1      ██░░░░░░░░   🔴 Cancelado    │
└──────────────────────────────────────────────────────────────────────────────────┘
```

La barra de AVANCE es `Σ producido / Σ planificado` de las líneas del plan (HU-21).
🟡 marca la producción del día: es lo que se busca al abrir la pantalla por la mañana.

### Detalle del plan

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ← Plan #9                                                    🔵 En proceso      │
│     Planificado: 17/09/2026  ·  Producción: 18/09/2026                           │
│     Responsable: Ismael Batalla                                                  │
│     Obs.: Producción para el pedido #47 y reposición de estante                  │
│                                              [ Cancelar ]  [ ✓ Completar plan ]  │
├──────────────────────────────────────────────────────────────────────────────────┤
│  PRODUCTOS DEL PLAN  (HU-21)                            [ + Agregar producto ]   │
├──────────────────────────────────────────────────────────────────────────────────┤
│  PRODUCTO         PLANIFICADO   PRODUCIDO   AVANCE          ⋯                    │
├──────────────────────────────────────────────────────────────────────────────────┤
│  Pan francés         90 und       60 und    ██████░░░░ 67%  ✏️                   │
│  Croissant           30 und       30 und    ██████████ 100% ✏️                   │
├──────────────────────────────────────────────────────────────────────────────────┤
│  CONSUMO DE MATERIA PRIMA  (HU-22)                                               │
│  ... (ver HU-22)                                                                 │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Formulario

```
┌────────────────────────────────────────────────┐
│  Nuevo plan de producción                 [X]  │
├────────────────────────────────────────────────┤
│  Fecha planificación *   Fecha producción *    │
│  ┌──────────────────┐   ┌──────────────────┐   │
│  │ 18/09/2026    📅 │   │ 19/09/2026    📅 │   │
│  └──────────────────┘   └──────────────────┘   │
│                          No anterior a la      │
│                          planificación         │
│                                                │
│  Responsable                                   │
│  ┌──────────────────────────────────────────┐  │
│  │ Ismael Batalla (usted)                   │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  Observaciones                                 │
│  ┌──────────────────────────────────────────┐  │
│  │ Producción para el pedido #47 y          │  │
│  │ reposición de estante                    │  │
│  └──────────────────────────────────────────┘  │
│                                     58 / 255   │
│                                                │
│              [ Cancelar ]  [ Crear plan ]      │
└────────────────────────────────────────────────┘
```

El responsable aparece como campo **de solo lectura** para PRODUCCION (R-04); solo ADMIN ve
un selector.

### Cancelación — advertencia honesta

```
┌────────────────────────────────────────────────────┐
│  ⚠  Cancelar el plan #9                            │
├────────────────────────────────────────────────────┤
│  El plan quedará cancelado y no podrá reactivarse. │
│                                                    │
│  ⚠ La materia prima ya consumida NO volverá al     │
│    inventario. Se registraron 3 consumos por un    │
│    total de 18,5 kg.                               │
│                                                    │
│    Para corregir un consumo mal registrado, use    │
│    un ajuste de inventario.                        │
│                                                    │
│      [ Volver ]  [ Cancelar el plan ]              │
└────────────────────────────────────────────────────┘
```

Esta advertencia (R-06) es lo que evita la expectativa equivocada de que cancelar "deshace
todo". La harina ya se usó.

---

## 2. Flujo

```
Productor            Frontend               Backend
    │                   │                      │
    │ Nuevo plan        │ POST /planes-produccion
    ├──────────────────►├─────────────────────►│ ¿fechas válidas?
    │                   │                      │ responsable = token  ← R-04
    │                   │                      │ INSERT (PLANIFICADO)
    │                   │◄─────────────────────┤ 201
    │                   │                      │
    │  (HU-21: agrega productos al plan)       │
    │                   │                      │
    │ "Iniciar"         │ PATCH .../estado {"EN_PROCESO"}
    ├──────────────────►├─────────────────────►│ ¿transición válida?
    │                   │                      │ ¿tiene productos?   ← R-02
    │                   │                      │ UPDATE estado
    │                   │◄─────────────────────┤ 200
    │                   │                      │
    │  (HU-22: registra consumo → HU-23 descuenta inventario)
    │                   │                      │
    │ "Completar"       │ PATCH .../estado {"COMPLETADO"}
    ├──────────────────►├─────────────────────►│ UPDATE estado
    │                   │◄─────────────────────┤ 200 (terminal)
```

**Ninguna de estas transiciones toca el inventario.** El stock se mueve en HU-22 y HU-23.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Estado | `MatChip` con color |
| Avance | `MatProgressBar` con porcentaje |
| Fechas | `MatDatepicker` con `min` |
| Responsable | `MatSelect` (solo ADMIN) o texto de solo lectura |
| Observaciones | `MatInput` textarea con contador |
| Acciones de estado | `MatRaisedButton` con `MatTooltip` explicativo |
| Cancelación | `MatDialog` con advertencia de consumos |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Fecha planificación | requerida | "Indique la fecha de planificación" |
| Fecha producción | requerida, ≥ planificación | "No puede ser anterior a la planificación" |
| Responsable | requerido (automático) | — |
| Observaciones | opcional, máx. 255 | "Máximo 255 caracteres" |

**No hay campo de estado** (R-01): nace `PLANIFICADO` y cambia por el flujo de acciones.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío | "No hay planes de producción" + "Crear el primero" |
| Vacío con filtro | "Ningún plan coincide con los filtros" |
| **Plan sin productos** | ⚠️ "Agregue productos antes de iniciar la producción"; botón "Iniciar" deshabilitado con tooltip (R-02) |
| Cancelando | Diálogo con el resumen de consumos ya registrados |
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
