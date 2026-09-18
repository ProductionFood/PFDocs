# HU-12 · Gestión de recetas — Diseño

---

## 1. Pantallas

### Editor de receta

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ← Receta: Masa de pan francés                              ●Activa   [⋮]    │
├──────────────────────────────────────────────────────────────────────────────┤
│  Producto                     Nombre de la receta                            │
│  ┌────────────────────────┐   ┌────────────────────────────────────────────┐ │
│  │ Pan francés         ▾  │   │ Masa de pan francés                        │ │
│  └────────────────────────┘   └────────────────────────────────────────────┘ │
│                                                                              │
│  Cantidad que produce *        Unidad *                                      │
│  ┌──────────────────┐         ┌──────────────────┐                          │
│  │ 30,00            │         │ Unidad (und)  ▾  │                          │
│  └──────────────────┘         └──────────────────┘                          │
│  ℹ Cantidad que rinde UNA TANDA completa de esta receta.                     │
│    Ejemplo: si con estos ingredientes salen 30 panes, escriba 30.            │
├──────────────────────────────────────────────────────────────────────────────┤
│  INGREDIENTES                                          [ + Agregar ]         │
├──────────────────────────────────────────────────────────────────────────────┤
│  MATERIA PRIMA        CANTIDAD      POR UNIDAD     COSTO EST.          ⋯     │
├──────────────────────────────────────────────────────────────────────────────┤
│  Harina de trigo      5,000 kg      0,167 kg/und   $ 16.000,00      ✏️ 🗑️   │
│  Levadura             0,050 kg      0,002 kg/und   $      9,00      ✏️ 🗑️   │
│  Sal                  0,100 kg      0,003 kg/und   $     85,00      ✏️ 🗑️   │
├──────────────────────────────────────────────────────────────────────────────┤
│                       COSTO TOTAL ESTIMADO POR TANDA:      $ 16.094,00      │
│                       COSTO ESTIMADO POR UNIDAD:           $    536,47      │
│                       Precio de venta actual:              $  1.200,00      │
│                       Margen estimado:                            55,3 %    │
└──────────────────────────────────────────────────────────────────────────────┘
```

**La columna "POR UNIDAD" es la defensa contra el error de R-02.** Si alguien escribe `1`
en "cantidad que produce" pensando que es por unidad, esa columna mostrará *5,000 kg/und* de
harina por pan — un absurdo evidente que se corrige al instante, en lugar de descubrirse
tres sprints después en HU-23.

El margen estimado es informativo: usa el costo de referencia de HU-08, no el costo real.

### Agregar ingrediente

```
┌────────────────────────────────────────────┐
│  Agregar ingrediente                  [X]  │
├────────────────────────────────────────────┤
│  Materia prima *                           │
│  ┌──────────────────────────────────────┐  │
│  │ Mantequilla (kg)                  ▾  │  │
│  └──────────────────────────────────────┘  │
│  Solo se muestran las que aún no están     │
│  en esta receta                            │
│                                            │
│  Cantidad *                                │
│  ┌──────────────────┐                      │
│  │ 0,250        kg  │                      │
│  └──────────────────┘                      │
│  Admite hasta 3 decimales                  │
│                                            │
│  Costo estimado: $ 4.625,00                │
│                                            │
│              [ Cancelar ]  [ Agregar ]     │
└────────────────────────────────────────────┘
```

El selector **excluye los ingredientes ya presentes** (CA-05): el duplicado no llega ni a
intentarse.

---

## 2. Flujo

```
Usuario              Frontend                 Backend
   │                    │                        │
   │ Nueva receta       │ GET /productos?sinReceta=true
   ├───────────────────►├───────────────────────►│
   │                    │◄───────────────────────┤ productos disponibles
   │ Cabecera + 3 ingr. │                        │
   ├───────────────────►│ POST /recetas          │
   │                    ├───────────────────────►│
   │                    │                        │ ── BEGIN ──────────────►
   │                    │                        │ ¿producto existe y activo?
   │                    │                        │ ¿ya tiene receta?  → CA-01
   │                    │                        │ ¿cantidadProducir > 0?
   │                    │                        │ ¿MP repetidas en la lista?
   │                    │                        │ INSERT recetas
   │                    │                        │ INSERT detalle_receta ×3
   │                    │                        │   (uk_detalle_receta protege)
   │                    │                        │ bitácora
   │                    │                        │ ── COMMIT ─────────────►
   │                    │◄───────────────────────┤ 201 con costos calculados
   │  Ve la receta      │                        │
   │◄───────────────────┤                        │
```

**Todo en una transacción**: si falla el tercer ingrediente, no queda una receta a medias.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Cabecera | `MatFormField` + `MatSelect` |
| Ayuda de rendimiento | `MatHint` destacado con ejemplo |
| Tabla de ingredientes | `MatTable` con acciones por fila |
| Agregar ingrediente | `MatDialog` |
| Cantidad | `MatInput[type=number]` con `step="0.001"` |
| Resumen de costos | `MatCard` al pie de la tabla |
| Quitar | `MatIconButton` + `ConfirmarDialogComponent` |
| Estado | `MatChip` |

---

## 4. Validaciones en el formulario

**Cabecera**

| Campo | Reglas | Mensaje |
|---|---|---|
| Producto | requerido, sin receta previa | "Seleccione un producto" |
| Nombre | requerido, máx. 100 | "El nombre es obligatorio" |
| Cantidad a producir | requerida, **> 0** | "Debe ser mayor que cero" |
| Unidad | requerida | "Seleccione una unidad" |

**Ingrediente**

| Campo | Reglas | Mensaje |
|---|---|---|
| Materia prima | requerida, no repetida | "Seleccione una materia prima" |
| Cantidad | requerida, > 0, **3 decimales** | "Debe ser mayor que cero" |

El `step` del input de cantidad debe ser `0.001` (R-06). Con `0.01` no se puede registrar
0,005 kg de sal, y en repostería esa cifra es real.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Receta sin ingredientes | ⚠️ "Esta receta no tiene ingredientes. No se podrá usar para planificar producción." |
| Sin productos disponibles | "Todos los productos ya tienen receta" o enlace a HU-11 |
| Error | Mensaje + "Reintentar" |
| Guardando | Botón con spinner |

El aviso de receta sin ingredientes dice **la consecuencia** (R-05), no solo que falta algo:
es lo que hace que alguien la complete en lugar de dejarla así.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
