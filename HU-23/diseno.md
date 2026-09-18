# HU-23 · Inventario automático por producción — Diseño

---

## 1. Pantallas

### Esta historia no tiene pantallas propias

Su interfaz es la de HU-22 más la presentación de las alertas. Lo que aporta visualmente es
hacer **evidente el efecto sobre el inventario**.

### Impacto antes de confirmar (en el diálogo de HU-22)

```
┌────────────────────────────────────────────────────────────┐
│  ⚠ Al registrar, se descontará del inventario:             │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Harina de trigo   −16,20 kg   112,50 →  96,30 kg  ✅ │  │
│  │ Levadura           −0,15 kg     2,00 →   1,85 kg  🟠 │  │
│  │ Sal                −0,30 kg     8,20 →   7,90 kg  ✅ │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  🟠 Levadura quedará por debajo del mínimo (5,00 kg)       │
│                                                            │
│              [ Cancelar ]  [ Registrar consumo ]           │
└────────────────────────────────────────────────────────────┘
```

La advertencia de stock bajo aparece **antes**, pero **no bloquea** (R-04): la materia prima
ya se usó físicamente.

### Resultado y alertas

```
┌────────────────────────────────────────────────────────────┐
│  ✅ Consumo registrado                                     │
│                                                            │
│  Inventario actualizado:                                   │
│    Harina de trigo   112,50 →  96,30 kg                    │
│    Levadura            2,00 →   1,85 kg                    │
│    Sal                 8,20 →   7,90 kg                    │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 🟠 ALERTA DE STOCK                                   │  │
│  │                                                      │  │
│  │ Levadura quedó en 1,85 kg (mínimo 5,00 kg)           │  │
│  │ Faltan 3,15 kg para alcanzar el nivel mínimo.        │  │
│  │                                                      │  │
│  │  [ Ver alertas de stock ]  [ Registrar compra ]      │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

Las acciones sugeridas conectan con HU-27 y HU-13: la alerta no es solo información, es el
punto donde empieza la reposición.

### Stock insuficiente

```
┌────────────────────────────────────────────────────────────┐
│  🔴 No se pudo registrar el consumo                        │
│                                                            │
│  No hay suficiente Levadura:                               │
│      Disponible:  0,15 kg                                  │
│      Requerido:   2,00 kg                                  │
│      Faltan:      1,85 kg                                  │
│                                                            │
│  ℹ Ningún consumo de esta operación fue registrado.        │
│                                                            │
│  Si la materia prima sí estaba disponible físicamente,     │
│  el inventario del sistema está desactualizado. Registre   │
│  un ajuste de inventario antes de continuar.               │
│                                                            │
│                      [ Entendido ]                         │
└────────────────────────────────────────────────────────────┘
```

Dos cosas importan en este mensaje:

1. **"Ningún consumo fue registrado"** confirma el rollback (R-02): el usuario sabe que no
   quedó nada a medias.
2. **La sugerencia del ajuste** (R-05) le da una salida cuando la materia prima sí estaba
   allí. Sin ella, el productor queda bloqueado ante un sistema que le dice que no hay algo
   que tiene delante.

### Indicador global de alertas

```
┌────────────────────────────────────────────────────────────────────────┐
│ ☰  ProductionFood            🟠 2 alertas    Ismael B. · PRODUCCION [⏻]│
└────────────────────────────────────────────────────────────────────────┘
```

Visible desde cualquier pantalla para ADMIN, PRODUCCION y COMPRAS. Al pulsarlo lleva a
HU-27.

---

## 2. Flujo

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TRANSACCIÓN ÚNICA  (CA-04)                           │
│                                                                         │
│   POST /detalles-plan/21/consumos  con 3 ingredientes                   │
│                            │                                            │
│                       ── BEGIN ──                                       │
│                            │                                            │
│    ┌───────────────────────┼───────────────────────┐                    │
│    │  Por cada ingrediente:                        │                    │
│    │                                               │                    │
│    │  1. SELECT ... FOR UPDATE   ← bloqueo (R-03)  │                    │
│    │  2. ¿stock suficiente?  ──NO──► 409 ──────────┼──► ROLLBACK        │
│    │  3. redondeo HALF_UP 2 dec  ← (R-08)          │      completo      │
│    │  4. UPDATE inventario                         │      (R-02)        │
│    │     └─ fecha_actualizacion automática (R-06)  │                    │
│    │  5. INSERT movimientos_inventario_mp          │                    │
│    │  6. INSERT consumo_materia_prima              │                    │
│    │                                               │                    │
│    └───────────────────────┼───────────────────────┘                    │
│                            │                                            │
│              consultar alertas de stock (R-04)                          │
│                            │                                            │
│                      ── COMMIT ──                                       │
│                            │                                            │
│              201 + saldos + alertas                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Un fallo en cualquier punto revierte todo.** Es la garantía que hace que el inventario
signifique algo: no existe el estado intermedio de "dos ingredientes descontados y el
consumo sin registrar".

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Impacto previsto | `MatCard` en el diálogo de HU-22 |
| Saldos resultantes | `MatList` en la confirmación |
| Alerta de stock | `MatCard` ámbar/roja con acciones |
| Indicador global | `MatBadge` sobre `MatIcon` en `MatToolbar` |
| Error de stock | `MatDialog` con las tres cifras y la sugerencia |
| Nivel de alerta | `MatChip` — ámbar `BAJO`, rojo `AGOTADO` |

---

## 4. Validaciones en el formulario

Esta historia **no aporta formularios propios**. Las validaciones son las de HU-22.

Lo que sí define es el **contenido de los mensajes**:

| Situación | Qué debe decir |
|---|---|
| Stock insuficiente | Disponible, requerido, faltante, y que nada se registró |
| Alerta de stock bajo | Cantidad actual, mínimo, faltante, y qué hacer |
| Inventario no inicializado | Que es un problema de datos, no del usuario |

Un mensaje que solo diga "stock insuficiente" deja al productor sin saber cuánto falta ni
qué puede hacer.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Antes de confirmar | Impacto previsto con semáforo por ingrediente |
| Registrando | Botón con spinner; diálogo bloqueado |
| Registrado sin alertas | ✅ Confirmación con los saldos resultantes |
| Registrado con alertas | ✅ + 🟠 tarjeta de alerta con acciones sugeridas |
| Stock insuficiente | 🔴 Diálogo con cifras, confirmación de rollback y salida sugerida |
| Inventario no inicializado | 🔴 "Esta materia prima no tiene inventario inicializado. Contacte al administrador." |

El caso de inventario no inicializado (R-01 de HU-08) no es culpa de quien registra: el
mensaje debe dejarlo claro y dirigirlo a quien puede resolverlo.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
