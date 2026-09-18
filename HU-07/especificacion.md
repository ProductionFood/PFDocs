# HU-07 · Unidades de medida — Especificación

**Fase 2** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** CRUD de unidades de medida (nombre, abreviatura),
> **para que** estandarice las cantidades en todo el sistema.

**Depende de:** [HU-03](../HU-03/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Nombre y abreviatura obligatorios | `@NotBlank` en ambos |
| CA-02 | La abreviatura debe ser **única** | `UNIQUE KEY abreviatura` ya existe en el esquema; validación previa en el servicio |
| CA-03 | Listar todas las unidades disponibles | `GET /unidades-medida` — **sin paginación** |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/unidades-medida` | Listar todas | Todos los roles autenticados |
| `GET` | `/api/v1/unidades-medida/{id}` | Obtener una | Todos los roles autenticados |
| `POST` | `/api/v1/unidades-medida` | Crear | ADMIN |
| `PUT` | `/api/v1/unidades-medida/{id}` | Editar | ADMIN |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Sin paginación, y es deliberado (CA-03)
Una panadería maneja entre 5 y 15 unidades de medida. Paginar un catálogo de ese tamaño
añade complejidad al cliente sin beneficio, y además **este endpoint alimenta los
desplegables** de HU-08, HU-11 y HU-12: un selector paginado es una mala experiencia.

Devuelve la lista completa ordenada por nombre. Es la única excepción a la regla de
paginación de `04-CONTRATO-API.md` §4, y por eso se declara explícitamente.

### R-02 · La abreviatura se normaliza antes de comparar
Se recorta y se guarda tal como se escribe, respetando mayúsculas y minúsculas: `kg` y `KG`
se muestran distinto.

**Pero la collation `utf8mb4_0900_ai_ci` es *case-insensitive*:** el `UNIQUE` considera
`kg` y `KG` **la misma abreviatura**, y la segunda es rechazada. Es el comportamiento
correcto —tener ambas sería confuso— pero conviene saberlo, porque el mensaje de error
("la abreviatura ya existe") puede desconcertar a quien acaba de escribir algo que en
pantalla se ve diferente.

### R-03 · Lo que impide eliminar una unidad
`materias_primas.id_unidad`, `productos.id_unidad` y `recetas.id_unidad` la referencian.
Borrar una unidad en uso rompería la integridad referencial.

**Esta tabla no tiene columna `estado`**, así que tampoco se puede desactivar. Por eso
**no se implementa ningún `DELETE`**: una unidad creada por error se corrige editándola.

Se documenta como limitación consciente. Agregar una columna `estado` sería lo correcto si
el catálogo creciera, pero no lo justifica un conjunto de quince filas.

### R-04 · No se cambia el significado de una unidad en uso
Editar "Kilogramo/kg" para convertirlo en "Litro/L" **reinterpretaría silenciosamente todos
los registros existentes**: las materias primas que decían 50 kg pasarían a decir 50 L, sin
que nada cambie en sus datos.

El servicio verifica si la unidad está en uso antes de permitir editar la abreviatura, y
la interfaz advierte: *"Esta unidad se usa en 12 materias primas y 5 productos."*

### R-05 · Catálogo inicial
`V3__datos_semilla.sql` crea nueve unidades: kg, g, L, mL, und, doc, paq, bol, bdj.
Cubren el caso de una panadería sin necesidad de configuración previa.

---

## 4. Modelo de datos

**Tabla `unidades_medida`** (sin cambios)

| Columna | Tipo | Notas |
|---|---|---|
| `id_unidad` | `int unsigned` PK AI | |
| `nombre` | `varchar(50)` NOT NULL | CA-01 |
| `abreviatura` | `varchar(50)` NOT NULL **UNIQUE** | CA-01, CA-02 |

**Sin columna `estado`** — de ahí R-03.

```json
// GET /api/v1/unidades-medida
[
  { "idUnidad": 9, "nombre": "Bandeja",   "abreviatura": "bdj" },
  { "idUnidad": 8, "nombre": "Bolsa",     "abreviatura": "bol" },
  { "idUnidad": 6, "nombre": "Docena",    "abreviatura": "doc" },
  { "idUnidad": 2, "nombre": "Gramo",     "abreviatura": "g"   },
  { "idUnidad": 1, "nombre": "Kilogramo", "abreviatura": "kg"  },
  { "idUnidad": 3, "nombre": "Litro",     "abreviatura": "L"   }
]
```

**Respuesta como array plano**, no `PageResponse` (R-01).

```json
// POST /api/v1/unidades-medida
{ "nombre": "Media docena", "abreviatura": "1/2doc" }
```

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `ABREVIATURA_DUPLICADA` | 409 | Ya existe una unidad con esa abreviatura (ignorando mayúsculas) |
| `UNIDAD_EN_USO` | 409 | Se intenta cambiar la abreviatura de una unidad referenciada |

Los transversales (`VALIDACION_FALLIDA`, `NO_AUTENTICADO`, `SIN_PERMISO`,
`RECURSO_NO_ENCONTRADO`) están en `00-base/04-CONTRATO-API.md` §5.

---

## 6. Fuera de alcance

Lo que **no** cubre esta historia y no debe implementarse aquí:
se limita estrictamente a los criterios de aceptación listados arriba.
Cualquier funcionalidad adicional se propone como historia nueva, no se agrega en silencio.

---

## 7. Definición de Terminado

Se aplica la DoD de `00-base/01-CONVENCIONES.md` §7.

## 8. Notas

### Por qué esta historia va primero en la Fase 2

HU-08 CA-02 y HU-11 CA-04 exigen que `id_unidad` exista. Sin unidades registradas **no se
puede crear ni una materia prima ni un producto**.

El `00-base/10-ROADMAP.md` la pone antes que HU-08 y HU-11 por esta razón. La semilla V3
la deja resuelta desde el primer arranque, pero la pantalla debe existir para poder agregar
unidades que el negocio necesite.
