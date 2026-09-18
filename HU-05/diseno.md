# HU-05 · Gestión de clientes — Diseño

---

## 1. Pantallas

### Listado de clientes

```
┌──────────────────────────────────────────────────────────────────────┐
│  Clientes                                          [ + Nuevo cliente ]│
├──────────────────────────────────────────────────────────────────────┤
│  🔍 ┌────────────────────────┐   Estado ┌────────────┐               │
│     │ Buscar por nombre      │          │ Activos  ▾ │               │
│     └────────────────────────┘          └────────────┘               │
├──────────────────────────────────────────────────────────────────────┤
│  NOMBRE ▲                CONTACTO         TELÉFONO      ESTADO   ⋯   │
├──────────────────────────────────────────────────────────────────────┤
│  Panadería El Trigo      Ana Restrepo     310 444 2211  ●Activo  ⋮   │
│  Supermercado Central    Luis Pérez       604 285 9900  ●Activo  ⋮   │
│  Tienda La Esquina       Pedro Ruiz       300 555 1234  ●Activo  ⋮   │
├──────────────────────────────────────────────────────────────────────┤
│                      Elementos por página: 20 ▾   1–3 de 3   ◀  ▶   │
└──────────────────────────────────────────────────────────────────────┘
```

### Formulario (diálogo)

```
┌────────────────────────────────────────────┐
│  Nuevo cliente                        [X]  │
├────────────────────────────────────────────┤
│  Nombre *                                  │
│  ┌──────────────────────────────────────┐  │
│  │ Tienda La Esquina                    │  │
│  └──────────────────────────────────────┘  │
│                              18 / 100      │
│  ℹ Ya existe un cliente con este nombre    │
│                                            │
│  Persona de contacto                       │
│  ┌──────────────────────────────────────┐  │
│  │ Pedro Ruiz                           │  │
│  └──────────────────────────────────────┘  │
│                                            │
│  Teléfono                                  │
│  ┌──────────────────────────────────────┐  │
│  │ 300 555 1234                         │  │
│  └──────────────────────────────────────┘  │
│                                            │
│              [ Cancelar ]  [ Guardar ]     │
└────────────────────────────────────────────┘
```

El aviso de nombre duplicado es **informativo, no bloqueante** (R-03): puede haber dos
clientes con el mismo nombre legítimamente.

---

## 2. Flujo

```
Vendedor               Frontend                   Backend
   │                      │                          │
   │ Abre "Clientes"      │  GET /clientes?activo=true
   ├─────────────────────►├─────────────────────────►│
   │                      │◄─────────────────────────┤ 200 PageResponse
   │  Ve el listado       │                          │
   │◄─────────────────────┤                          │
   │                      │                          │
   │ "+ Nuevo cliente"    │                          │
   ├─────────────────────►│ (abre diálogo)           │
   │ Completa y guarda    │  POST /clientes          │
   ├─────────────────────►├─────────────────────────►│ valida
   │                      │                          │ INSERT (estado = 1)
   │                      │                          │ bitácora
   │                      │◄─────────────────────────┤ 201 + Location
   │ "Cliente registrado" │ (cierra diálogo,         │
   │◄─────────────────────┤  recarga listado)        │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatSort` + `MatPaginator` |
| Búsqueda | `MatFormField` + `MatInput` + icono de lupa |
| Filtro de estado | `MatSelect` |
| Estado | `MatChip` verde/gris |
| Menú de fila | `MatMenu` (Editar · Desactivar) |
| Formulario | `MatDialog` + `MatFormField` |
| Confirmación | `ConfirmarDialogComponent` (de HU-02) |
| Avisos | `MatSnackBar` |

---

## 4. Validaciones en el formulario

| Campo | Reglas | Mensaje |
|---|---|---|
| Nombre | requerido, máx. 100 | "El nombre es obligatorio" |
| Contacto | opcional, máx. 100 | "Máximo 100 caracteres" |
| Teléfono | opcional, máx. 20 | "Máximo 20 caracteres" |

**El teléfono no valida formato** (R-04): puede ser fijo, celular, con indicativo o con
extensión. Imponer un patrón obliga a falsear el dato cuando el número real no encaja.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` sobre la tabla |
| Vacío sin filtro | "No hay clientes registrados" + "Registrar el primero" |
| Vacío con filtro | "Ningún cliente coincide con «xxx»" + "Limpiar filtros" |
| Error | Mensaje + "Reintentar" |
| Guardando | Botón deshabilitado con spinner |

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
