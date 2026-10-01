# HU-02 · Gestión de usuarios — Especificación

**Fase 1** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** listar, editar y desactivar usuarios,
> **para que** gestione quién tiene acceso al sistema.

**Depende de:** [HU-01](../HU-01/) (registro de usuarios) — entregada, ver
`Backend/Docs/Hechos/HU-01-registro-usuarios.md`.

> **Revisión 2026-10-01.** Ajustada a lo que realmente entregó HU-01: persistencia con
> Spring Data JPA (no JdbcTemplate), validación de `estado` por petición ya implementada,
> login de usuario inactivo → `409 CREDENCIALES_INVALIDAS`, y bitácora (HU-04) aún no
> disponible. Se añaden R-06 y R-07 y se amplía R-02. Los CA no cambian.

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Listar todos los usuarios con paginación | `GET /usuarios` con `page`, `size`, `sort` — `04-CONTRATO-API.md` §4 |
| CA-02 | Filtrar por nombre o correo | Parámetro `busqueda` (`LIKE prefijo%` sobre ambas columnas). Filtros adicionales `idRol` y `activo` que usa la pantalla |
| CA-03 | Editar nombre, correo, rol y estado | Nombre, correo y rol con `PUT /usuarios/{id}`; **el estado solo con `PATCH`** (CA-04). La contraseña no se edita aquí |
| CA-04 | Desactivar/activar cambiando `estado` | `PATCH /usuarios/{id}/estado` |
| CA-05 | **No se permite eliminar**, solo desactivar | No existe endpoint `DELETE` → `405` |

CA-03 menciona el estado: se cumple entre los dos endpoints. Separarlos permite que las
reglas de desactivación (R-02) vivan en un solo sitio y que un `PUT` de edición nunca
desactive a nadie por accidente.

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/usuarios` | Listar con filtros y paginación | ADMIN |
| `GET` | `/api/v1/usuarios/{id}` | Obtener un usuario | ADMIN |
| `PUT` | `/api/v1/usuarios/{id}` | Editar nombre, correo y rol | ADMIN |
| `PATCH` | `/api/v1/usuarios/{id}/estado` | Activar o desactivar | ADMIN |

### Parámetros de `GET /usuarios`

| Parámetro | Tipo | Por defecto | Regla |
|---|---|---|---|
| `busqueda` | string | — | Prefijo sobre `nombre` **o** `correo`; `%` y `_` se tratan como literales |
| `idRol` | int | — | Solo usuarios de ese rol |
| `activo` | boolean | — | `true` / `false`; omitido = todos |
| `page` | int | `0` | `>= 0`; no numérico → `400 PARAMETRO_INVALIDO` |
| `size` | int | `20` | `1..100`; fuera de rango → `400 PARAMETRO_INVALIDO` (no se recorta en silencio) |
| `sort` | string | `nombre,asc` | Lista blanca `nombre`, `correo`, `id`; dirección omitida = `asc` |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Nunca se elimina un usuario (CA-05)
No hay endpoint `DELETE`, y no es solo una decisión de interfaz: `usuarios` está
referenciada por `planes_produccion.id_usuario`, `bitacora.id_usuario` y las dos tablas de
kardex. Borrar un usuario rompería la trazabilidad de todo lo que hizo.

Un `DELETE` responde `405 METODO_NO_PERMITIDO` con el formato de error estándar, no `500`.

### R-02 · El sistema nunca se queda sin administrador activo
Si no queda ningún `ADMIN` activo, **nadie puede volver a entrar a corregirlo** y solo se
recupera con acceso directo a la base. Hay **dos vías** para llegar ahí, y ambas se cierran:

| Vía | Comprobación | Error |
|---|---|---|
| `PATCH /estado` sobre sí mismo | `id == idUsuario del token` | `409 AUTO_DESACTIVACION` |
| `PATCH /estado` sobre el último `ADMIN` activo | `ADMIN` activos `<= 1` | `409 ULTIMO_ADMIN` |
| `PUT` que cambia **su propio** rol | `id == idUsuario del token` y el rol cambia | `409 AUTO_DEGRADACION` |
| `PUT` que quita `ADMIN` al último `ADMIN` activo | rol actual `ADMIN`, nuevo ≠ `ADMIN`, activos `<= 1` | `409 ULTIMO_ADMIN` |

La comprobación vive en **un único método del servicio** (`validarQuedaAdmin`) que usan los
dos endpoints. El conteo se hace con bloqueo (`PESSIMISTIC_WRITE`) dentro de la misma
transacción que el cambio: sin él, dos administradores que se desactivan mutuamente a la
vez pasan ambos la validación.

### R-03 · La contraseña no se edita en este endpoint
`PUT /usuarios/{id}` no acepta `password` (ni `activo`): si llegan en el cuerpo se
**ignoran**. Cambiar contraseña es una operación distinta, con reglas propias, y queda
fuera de alcance.

### R-04 · El correo sigue siendo único al editar
- Se normaliza igual que en HU-01: `trim` + minúsculas; el nombre con `trim`.
- La verificación de unicidad **excluye al propio usuario**
  (`existsByCorreoAndIdUsuarioNot`). Sin esa exclusión, guardar sin cambiar el correo
  devuelve `CORREO_DUPLICADO` contra sí mismo.
- La restricción `UNIQUE` de la base es la garantía final: si salta en el `flush`, se
  traduce también a `409 CORREO_DUPLICADO`.

### R-05 · Efecto inmediato de la desactivación — *ya implementado en HU-01*
El `JwtAuthenticationFilter` consulta `estado` en cada petición (PR #8): un usuario
desactivado recibe `403 USUARIO_INACTIVO` en su siguiente llamada aunque su token siga
vigente. HU-02 no lo reimplementa; lo cubre con pruebas de regresión (CP-14, SEC-05).

Al intentar **iniciar sesión** desactivado, la respuesta es `409 CREDENCIALES_INVALIDAS`
(decisión de HU-01: no revelar si falló el correo, la contraseña o el estado).

### R-06 · Los permisos salen de la base, no del token *(nueva)*
Hoy el filtro valida `estado` contra la base pero construye rol, nombre y correo desde los
*claims* del JWT. Con HU-02 el rol pasa a ser editable, así que un `ADMIN` degradado
conservaría permisos de administrador hasta una hora.

El filtro arma `UsuarioAutenticado` y las *authorities* con el usuario **leído de la base**
en esa misma petición (rol cargado con `JOIN FETCH`). Los *claims* solo identifican al
usuario (`sub`). Mismo principio que R-05, aplicado al rol: el cambio tiene efecto en la
siguiente petición.

### R-07 · Auditoría diferida a HU-04 *(nueva)*
HU-04 (bitácora) aún no existe y el roadmap la plantea como aspecto AOP transversal. HU-02
**no escribe en bitácora**: cuando HU-04 se implemente, cubrirá `EDITAR` y
`CAMBIAR_ESTADO` de usuarios sin tocar este servicio. Queda como pendiente explícito en
Hechos.

---

## 4. Modelo de datos

Misma tabla `usuarios` de HU-01. **Sin migración nueva**: los índices que necesita la
búsqueda ya existen (`correo` UNIQUE, `idx_usuarios_nombre` de `V2`).

```json
// GET /api/v1/usuarios?busqueda=mar&activo=true&page=0&size=20&sort=nombre,asc
{
  "content": [
    { "idUsuario": 12, "nombre": "María Gómez", "correo": "maria@productionfood.local",
      "activo": true, "rol": { "idRol": 4, "nombre": "VENTAS" } }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1,
  "first": true, "last": true
}

// PUT /api/v1/usuarios/12  → 200 con el UsuarioResponse actualizado
{ "nombre": "María Gómez Ruiz", "correo": "maria.gomez@productionfood.local", "idRol": 4 }

// PATCH /api/v1/usuarios/12/estado  → 200 con el UsuarioResponse actualizado
{ "activo": false }
```

Respuesta: el mismo `UsuarioResponse` de HU-01 (`idUsuario`, `nombre`, `correo`,
`activo`, `rol{idRol,nombre}`). La contraseña nunca aparece.

**Campos ordenables** (lista blanca → propiedad JPA): `nombre → nombre`,
`correo → correo`, `id → idUsuario`. Cualquier otro → `400 CAMPO_ORDEN_INVALIDO`.

### Validaciones de los cuerpos

| DTO | Campo | Regla |
|---|---|---|
| `ActualizarUsuarioRequest` | `nombre` | obligatorio, máx. 100 |
| | `correo` | obligatorio, formato correo, máx. 100 |
| | `idRol` | obligatorio |
| `CambiarEstadoRequest` | `activo` | obligatorio (`@NotNull`) |

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `AUTO_DESACTIVACION` | 409 | Un administrador intenta desactivarse a sí mismo |
| `AUTO_DEGRADACION` | 409 | Un administrador intenta cambiar su propio rol |
| `ULTIMO_ADMIN` | 409 | Se desactivaría o degradaría al último administrador activo |
| `CORREO_DUPLICADO` | 409 | El correo ya pertenece a otro usuario |
| `ROL_INEXISTENTE` | 409 | El `idRol` no existe (mismo código que HU-01) |

Transversales (`VALIDACION_FALLIDA`, `PARAMETRO_INVALIDO`, `CAMPO_ORDEN_INVALIDO`,
`NO_AUTENTICADO`, `SIN_PERMISO`, `USUARIO_INACTIVO`, `RECURSO_NO_ENCONTRADO`) en
`00-base/04-CONTRATO-API.md` §5. Esta HU añade al manejador global
`METODO_NO_PERMITIDO` (405), que hoy cae en `500 ERROR_INTERNO`.

---

## 6. Fuera de alcance

- Cambio y recuperación de contraseña.
- Escritura en bitácora (R-07 → HU-04).
- Revocación de tokens / logout server-side (no hace falta: R-05 y R-06 dan efecto
  inmediato consultando la base).
- Cambiar el código de login de usuario inactivo (409 vs 403 de HU-03 CA-03): se decide
  en HU-03, no aquí.

Cualquier funcionalidad adicional se propone como historia nueva, no se agrega en silencio.

---

## 7. Definición de Terminado

Se aplica la DoD de `00-base/01-CONVENCIONES.md` §7, con dos ajustes de esta revisión:
la auditoría en bitácora no aplica (R-07) y el plan de trabajo por PR está en
`tarea-backend.md` §1.

## 8. Notas

### Sobre la búsqueda por prefijo

CA-02 pide filtrar por nombre **o** correo. La implementación usa
`(u.nombre LIKE :q ESCAPE '\' OR u.correo LIKE :q ESCAPE '\')` con patrón
`texto%`. Sin `LOWER()`: anularía el índice, y la colación ya ignora mayúsculas.

Buscar "gomez" **no** encontrará a "María Gómez" — el apellido no está al inicio del campo
`nombre`. Es el compromiso consciente de `04-CONTRATO-API.md` §4. La colación
`utf8mb4_0900_ai_ci` ya hace la comparación insensible a mayúsculas y tildes.

Si en pruebas resulta incómodo, la alternativa correcta es un índice `FULLTEXT`, no cambiar
a `%texto%`.
