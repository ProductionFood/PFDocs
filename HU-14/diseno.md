# HU-14 · Detalle de compra — Diseño

---

## 1. Pantallas

### Tabla de ítems (dentro de la pantalla de HU-13)

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ÍTEMS DE LA COMPRA #12                                     [ + Agregar ítem ] │
├────────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA      CANTIDAD     PRECIO UNIT.    vs. REF.   SUBTOTAL       ⋯   │
├────────────────────────────────────────────────────────────────────────────────┤
│  Harina de trigo    100,00 kg    $ 3.200,00        =       $ 320.000,00  ✏️🗑️ │
│  Azúcar              50,00 kg    $ 3.500,00     ↑ +$300    $ 175.000,00  ✏️🗑️ │
│  Levadura             2,00 kg    $   150,00     ↓ -$30     $     300,00  ✏️🗑️ │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                  TOTAL:    $ 495.300,00       │
│                                                  3 ítems                       │
└────────────────────────────────────────────────────────────────────────────────┘
```

La columna **vs. REF.** compara con el costo de referencia de HU-08 (R-04):
↑ rojo más caro · ↓ verde más barato · `=` igual. Hace visible de inmediato si el proveedor
subió precios, sin tener que recordar cuánto costaba antes.

**El TOTAL es el que devuelve el backend** (R-03), no una suma del navegador.

### Agregar ítem

```
┌────────────────────────────────────────────────────┐
│  Agregar ítem a la compra                     [X]  │
├────────────────────────────────────────────────────┤
│  Materia prima *                                   │
│  ┌──────────────────────────────────────────────┐  │
│  │ Mantequilla (kg)                          ▾  │  │
│  └──────────────────────────────────────────────┘  │
│  Solo se muestran las que aún no están en esta     │
│  compra. Para cambiar una cantidad, edite la fila. │
│                                                    │
│  Cantidad *              Precio unitario *         │
│  ┌──────────────────┐   ┌──────────────────────┐   │
│  │ 20,00        kg  │   │ $ 18.500,00          │   │
│  └──────────────────┘   └──────────────────────┘   │
│                          Costo de referencia:      │
│                          $ 18.500,00               │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │  Subtotal:                    $ 370.000,00   │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│              [ Cancelar ]  [ Agregar ]             │
└────────────────────────────────────────────────────┘
```

El precio viene **prellenado con el costo de referencia** y es editable. Ahorra tecleo en el
caso normal y hace evidente la desviación cuando el proveedor cobra otra cosa.

El selector excluye las materias primas ya presentes (R-02) y lo explica: sin esa nota,
quien no encuentra la harina en la lista piensa que falta en el catálogo.

### Compra ya recibida — modo lectura

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ÍTEMS DE LA COMPRA #11                          🔒 Compra recibida            │
├────────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA      CANTIDAD     PRECIO UNIT.    SUBTOTAL                      │
├────────────────────────────────────────────────────────────────────────────────┤
│  Leche entera       200,00 L     $ 2.800,00     $ 560.000,00                   │
│  Mantequilla         40,00 kg    $ 17.000,00    $ 680.000,00                   │
├────────────────────────────────────────────────────────────────────────────────┤
│                                   TOTAL:        $ 1.240.000,00                 │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Sin botón de agregar y sin acciones por fila** (R-01). Los controles se ocultan, no se
deshabilitan: un botón gris invita a preguntarse por qué no funciona.

---

## 2. Flujo

```
Comprador            Frontend                  Backend
    │                   │                         │
    │ "+ Agregar ítem"  │                         │
    ├──────────────────►│ GET /materias-primas    │
    │                   │ (filtra las ya usadas)  │
    │                   │◄────────────────────────┤
    │ Selecciona MP     │                         │
    ├──────────────────►│ (prellena precio con    │
    │                   │  el costo de referencia)│
    │ Confirma          │ POST /compras/12/detalles
    ├──────────────────►├────────────────────────►│ ¿compra existe?
    │                   │                         │ ¿está PENDIENTE?   ← R-01
    │                   │                         │ ¿MP existe y activa?
    │                   │                         │ ¿ya está en la compra? ← R-02
    │                   │                         │ INSERT detalle_compra
    │                   │                         │ recalcula total    ← R-03
    │                   │                         │ bitácora
    │                   │◄────────────────────────┤ 201 + subtotal + total
    │ Tabla actualizada │                         │
    │◄──────────────────┤                         │
```

**El total siempre viene del backend.** El frontend lo muestra, no lo calcula.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla de ítems | `MatTable` con fila de totales (`matFooterRow`) |
| Agregar | `MatDialog` |
| Materia prima | `MatSelect` filtrado |
| Cantidad / precio | `MatInput[type=number]` con `step="0.01"` |
| Comparativo de costo | `MatIcon` ↑↓ con color + `MatTooltip` |
| Subtotal y total | Pipe `currency:'COP':'symbol-narrow':'1.2-2'` |
| Eliminar | `MatIconButton` + `ConfirmarDialogComponent` |
| Modo lectura | `MatIcon` `lock` en la cabecera |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Materia prima | requerida, activa, no repetida | "Seleccione una materia prima" |
| Cantidad | requerida, **> 0**, 2 decimales | "Debe ser mayor que cero" |
| Precio unitario | requerido, **≥ 0**, 2 decimales | "No puede ser negativo" |

La asimetría entre ambos campos es deliberada (R-05): una línea de cero unidades no tiene
sentido, pero una de precio cero sí — es una bonificación o una muestra.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` sobre la tabla |
| Sin ítems, compra editable | ⚠️ "Esta compra no tiene ítems. Agregue al menos uno antes de recibirla." |
| Sin ítems, compra cerrada | "Esta compra no registra ítems" |
| Compra no editable | 🔒 Modo lectura, sin controles de escritura |
| Guardando | Fila con spinner; el resto de la tabla sigue usable |
| Error | Mensaje del servidor junto al campo, o `MatSnackBar` |

El aviso de "sin ítems" enlaza con HU-13 R-03: una compra vacía no se puede recibir, y
conviene saberlo antes de intentarlo.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
