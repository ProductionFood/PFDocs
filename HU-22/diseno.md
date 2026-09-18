# HU-22 · Registro de consumo de materia prima — Diseño

---

## 1. Pantallas

### Tabla de consumos (dentro del plan)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  CONSUMO DE MATERIA PRIMA · Pan francés (90 und planificadas)  [ Registrar consumo ] │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA      TEÓRICO     REAL       DESVIACIÓN      EFICIENCIA   REGISTRO    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  Harina de trigo   15,000 kg  16,200 kg   🔴 +8,0 %          92,6 %     10:22      │
│  Levadura           0,150 kg   0,150 kg   🟢  0,0 %         100,0 %     10:23      │
│  Sal                0,300 kg   0,000 kg   🔵 −100,0 %          —  ⓘ     10:23      │
│  Azúcar  ⚠️ fuera   0,000 kg   0,500 kg       —               0,0 %     10:25      │
│          de receta                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**La columna DESVIACIÓN es la que se lee** (R-04): `+8,0%` dice "se gastó un 8% más".
La eficiencia de 92,6% para el mismo dato es correcta pero menos directa.

El `—` con ⓘ en la fila de Sal es `eficiencia: null` (R-03): no se puede calcular porque no
se consumió nada. El tooltip lo explica. **Nunca debe aparecer `NaN` ni `Infinity`.**

El azúcar está marcado **fuera de receta** (R-08): se consumió algo que la receta no
contempla, dato útil porque indica que la receta no refleja el proceso real.

### 🔑 Registrar consumo

```
┌──────────────────────────────────────────────────────────────────┐
│  Registrar consumo · Pan francés                            [X]  │
├──────────────────────────────────────────────────────────────────┤
│  Planificado: 90 und   ·   Receta rinde 30 und   ·   Factor: ×3  │
│                                                                  │
│  MATERIA PRIMA      TEÓRICO      REAL CONSUMIDO    DISPONIBLE    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Harina de trigo   15,000 kg   ┌────────────┐    112,50 kg ✅│  │
│  │                               │ 16,200     │               │  │
│  │                               └────────────┘               │  │
│  │ Levadura           0,150 kg   ┌────────────┐      0,00 kg 🔴│  │
│  │                               │  0,150     │  ⚠ sin stock  │  │
│  │                               └────────────┘               │  │
│  │ Sal                0,300 kg   ┌────────────┐      8,20 kg ✅│  │
│  │                               │  0,300     │               │  │
│  │                               └────────────┘               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  [ + Agregar ingrediente fuera de receta ]                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ ⚠ Al registrar, se descontará del inventario:              │  │
│  │                                                            │  │
│  │   Harina de trigo   −16,20 kg   112,50 → 96,30 kg          │  │
│  │   Sal                −0,30 kg     8,20 →  7,90 kg          │  │
│  │                                                            │  │
│  │   🔴 Levadura: no hay stock suficiente (0,00 disponible)   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│              [ Cancelar ]  [ Registrar consumo ]                 │
└──────────────────────────────────────────────────────────────────┘
```

**Los campos vienen prellenados con el teórico**: en el caso normal se consume lo previsto y
solo hay que corregir lo que difiere.

El recuadro de impacto (R-05) deja claro que esto **descuenta inventario**, no es un apunte.
Y advierte del stock insuficiente antes de intentar la operación.

### Corregir un consumo

```
┌────────────────────────────────────────────────┐
│  Corregir consumo · Harina de trigo       [X]  │
├────────────────────────────────────────────────┤
│  Registrado actualmente:  16,200 kg            │
│                                                │
│  Cantidad real corregida *                     │
│  ┌──────────────────┐                          │
│  │ 18,000       kg  │                          │
│  └──────────────────┘                          │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │ Se descontarán 1,800 kg adicionales      │  │
│  │ Inventario:  96,30 → 94,50 kg            │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│              [ Cancelar ]  [ Corregir ]        │
└────────────────────────────────────────────────┘
```

Indica el **ajuste**, no solo el valor final (R-07): es lo que hace comprensible qué va a
pasar con el inventario.

---

## 2. Flujo

```
Productor          Frontend            Backend         InventarioService
   │                  │                   │                    │
   │ "Registrar"      │ GET .../consumo-teorico                │
   ├─────────────────►├──────────────────►│ calcula desde la receta ← R-01
   │                  │◄──────────────────┤ teóricos + disponibles
   │ Ajusta los reales│                   │                    │
   │ Confirma         │ POST .../consumos │                    │
   ├─────────────────►├──────────────────►│                    │
   │                  │                   │ ── BEGIN ────────► │
   │                  │                   │ ¿plan EN_PROCESO?  │
   │                  │                   │ ¿duplicado?  ← R-06│
   │                  │                   │ calcula teórica ← R-01
   │                  │                   ├───────────────────►│ FOR UPDATE
   │                  │                   │                    │ ¿stock suficiente?
   │                  │                   │                    │ UPDATE inventario
   │                  │                   │                    │ INSERT movimiento
   │                  │                   │◄───────────────────┤   SALIDA_PRODUCCION
   │                  │                   │ INSERT consumo     │
   │                  │                   │ bitácora           │
   │                  │                   │ ── COMMIT ───────► │
   │                  │◄──────────────────┤ 201 + saldos       │
   │ "96,30 kg"       │                   │                    │
   │◄─────────────────┤                   │                    │
```

**HU-22 y HU-23 son la misma transacción.** Si el descuento falla, el consumo no se
registra.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla de consumos | `MatTable` |
| Desviación | `MatChip` con color por signo |
| Eficiencia nula | Texto `—` + `MatIcon` `info` con `MatTooltip` |
| Fuera de receta | `MatChip` ámbar |
| Diálogo de registro | `MatDialog` con tabla editable |
| Cantidades | `MatInput[type=number]` con `step="0.001"` |
| Impacto en inventario | `MatCard` destacada, en vivo |
| Disponibilidad | `MatIcon` ✅/🔴 por ingrediente |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Materia prima | requerida, no duplicada en la línea | "Seleccione una materia prima" |
| Cantidad real | requerida, ≥ 0, **3 decimales** | "No puede ser negativa" |

**Cero es válido** (R-03): significa que no se consumió ese ingrediente, y es un dato
legítimo. El `step` debe ser `0.001` (R-09).

⚠️ **`cantidadTeorica` no es un campo del formulario** (R-01): la calcula el servidor.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando teóricos | `MatProgressBar` en el diálogo |
| Producto sin receta | ⚠️ "Sin receta: no hay consumo teórico. Registre los ingredientes manualmente." |
| Stock insuficiente | 🔴 Junto al ingrediente, con el disponible; botón deshabilitado |
| **Eficiencia nula** | `—` con tooltip "No se puede calcular: no se registró consumo" |
| Consumo fuera de receta | ⚠️ Marcado en la tabla |
| Plan no `EN_PROCESO` | 🔒 Modo lectura |
| `409 STOCK_INSUFICIENTE` | Mensaje con el disponible actual |
| Registrando | Botón con spinner |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
