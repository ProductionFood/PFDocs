# HU-07 · Unidades de medida — Diseño

---

## 1. Pantallas

### Catálogo de unidades

```
┌──────────────────────────────────────────────────────────────┐
│  Unidades de medida                       [ + Nueva unidad ]  │
├──────────────────────────────────────────────────────────────┤
│  NOMBRE ▲            ABREVIATURA        EN USO         ⋯     │
├──────────────────────────────────────────────────────────────┤
│  Bandeja             bdj                3 registros    ⋮     │
│  Bolsa               bol                1 registro     ⋮     │
│  Docena              doc                4 registros    ⋮     │
│  Gramo               g                  8 registros    ⋮     │
│  Kilogramo           kg                 17 registros   ⋮     │
│  Litro               L                  5 registros    ⋮     │
│  Mililitro           mL                 2 registros    ⋮     │
│  Paquete             paq                — sin uso      ⋮     │
│  Unidad              und                12 registros   ⋮     │
└──────────────────────────────────────────────────────────────┘
```

**Sin paginador** (R-01): el catálogo completo cabe en pantalla.
El menú `⋮` solo ofrece **Editar** — no hay eliminar (R-03).

La columna "En uso" hace visible qué unidades sostienen datos reales, y por tanto cuáles
son delicadas de modificar.

### Advertencia al editar una unidad en uso

```
┌────────────────────────────────────────────────────┐
│  ⚠  Esta unidad está en uso                        │
├────────────────────────────────────────────────────┤
│  «Kilogramo (kg)» se usa en:                       │
│                                                    │
│     • 12 materias primas                           │
│     •  5 productos                                 │
│                                                    │
│  Cambiar la abreviatura modificará cómo se         │
│  interpretan esas cantidades. Un registro de       │
│  «50 kg» pasará a leerse como «50 L» sin que       │
│  su valor cambie.                                  │
│                                                    │
│            [ Cancelar ]  [ Entiendo, cambiar ]     │
└────────────────────────────────────────────────────┘
```

El texto explica la consecuencia concreta con un ejemplo. Es la diferencia entre una
advertencia que se lee y una que se cierra por reflejo.

---

## 2. Flujo

```
Administrador          Frontend                    Backend
     │                    │                           │
     │ Abre catálogo      │ GET /unidades-medida      │
     ├───────────────────►├──────────────────────────►│
     │◄───────────────────┤◄──────────────────────────┤ 200 [array]
     │                    │ (cachea con shareReplay)  │
     │                    │                           │
     │ Nueva unidad       │ POST /unidades-medida     │
     ├───────────────────►├──────────────────────────►│ ¿abreviatura única?
     │                    │                           │ INSERT
     │◄───────────────────┤◄──────────────────────────┤ 201
     │                    │ invalidarCache()  ← clave │
     │                    │                           │
     └──── La nueva unidad ya aparece en los desplegables de
           materias primas (HU-08), productos (HU-11) y recetas (HU-12)
```

**Sin `invalidarCache()`**, la unidad recién creada no aparece en los otros módulos hasta
recargar la página, y el usuario concluye que la creación falló.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` **sin** `MatPaginator` |
| Formulario | `MatDialog` |
| Campos | `MatFormField` + `MatInput` |
| Menú de fila | `MatMenu` (solo Editar) |
| Advertencia de uso | `MatDialog` con `MatIcon` `warning` |
| En uso | `MatChip` o texto atenuado si es "sin uso" |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Nombre | requerido, máx. 50 | "El nombre es obligatorio" |
| Abreviatura | requerida, máx. 50, sin espacios al inicio o final | "La abreviatura es obligatoria" |

La unicidad de la abreviatura **solo puede verificarla el backend** (CA-02). El cliente
puede avisar si coincide con alguna de la lista ya cargada, pero la autoridad es el
`409 ABREVIATURA_DUPLICADA`.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío | "No hay unidades registradas" + "Crear la primera". **Poco probable**: la semilla V3 deja nueve |
| Error | Mensaje + "Reintentar" |
| Guardando | Botón con spinner |

En los otros módulos, un desplegable de unidades vacío debe mostrar un enlace a esta
pantalla: sin unidades no se puede crear una materia prima ni un producto, y conviene que
el camino a la solución sea evidente.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
