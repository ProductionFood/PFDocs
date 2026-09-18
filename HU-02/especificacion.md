# HU-02 · Gestión de usuarios — Especificación

**Fase 1** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** listar, editar y desactivar usuarios,
> **para que** gestione quién tiene acceso al sistema.

**Depende de:** [HU-01](../HU-01/) (registro de usuarios)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Listar todos los usuarios con paginación | `GET /usuarios` con `page`, `size`, `sort` — `04-CONTRATO-API.md` §4 |
| CA-02 | Filtrar por nombre o correo | Parámetro `busqueda` que aplica `LIKE prefijo%` sobre ambas columnas |
| CA-03 | Editar nombre, correo, rol y estado | `PUT /usuarios/{id}` — **la contraseña no se edita aquí** |
| CA-04 | Desactivar/activar cambiando `estado` | `PATCH /usuarios/{id}/estado` |
| CA-05 | **No se permite eliminar**, solo desactivar | No existe endpoint `DELETE` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/usuarios` | Listar con filtro y paginación | ADMIN |
| `GET` | `/api/v1/usuarios/{id}` | Obtener un usuario | ADMIN |
| `PUT` | `/api/v1/usuarios/{id}` | Editar nombre, correo y rol | ADMIN |
| `PATCH` | `/api/v1/usuarios/{id}/estado` | Activar o desactivar | ADMIN |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Nunca se elimina un usuario (CA-05)
No hay endpoint `DELETE`, y no es solo una decisión de interfaz: `usuarios` está
referenciada por `planes_produccion.id_usuario`, `bitacora.id_usuario` y las dos tablas de
kardex. Borrar un usuario rompería la trazabilidad de todo lo que hizo — que es
precisamente lo que la bitácora de HU-04 existe para conservar.

### R-02 · Un administrador no puede desactivarse a sí mismo
Si el único administrador activo se desactiva, **nadie puede volver a entrar a reactivarlo**.
El sistema queda inutilizable y solo se recupera con acceso directo a la base de datos.

Dos comprobaciones en el servicio:
- El `idUsuario` del token no puede ser el mismo que se desactiva → `409 AUTO_DESACTIVACION`.
- No se puede desactivar al último `ADMIN` activo → `409 ULTIMO_ADMIN`.

### R-03 · La contraseña no se edita en este endpoint
`PUT /usuarios/{id}` no acepta `password`. Cambiar contraseña es una operación distinta,
con reglas propias (contraseña actual, política de complejidad), y mezclarla con la edición
de datos hace que un `PUT` parcial la sobrescriba sin querer. Queda fuera de alcance.

### R-04 · El correo sigue siendo único al editar
La verificación de unicidad **excluye al propio usuario**:
`SELECT ... WHERE correo = ? AND id_usuario <> ?`. Sin esa exclusión, guardar un usuario sin
cambiarle el correo devuelve `CORREO_DUPLICADO` contra sí mismo.

### R-05 · Efecto inmediato de la desactivación
El token ya emitido **sigue siendo criptográficamente válido hasta que expira** (máx. 1 hora).
Por eso la validación de `estado` se hace **en cada petición**, no solo al iniciar sesión:
un usuario desactivado recibe `403 USUARIO_INACTIVO` en su siguiente llamada, no una hora
después.

---

## 4. Modelo de datos

Misma tabla `usuarios` de HU-01. Sin cambios de esquema.

```json
// GET /api/v1/usuarios?busqueda=mar&page=0&size=20&sort=nombre,asc
{
  "content": [
    { "idUsuario": 12, "nombre": "María Gómez", "correo": "maria@productionfood.local",
      "activo": true, "rol": { "idRol": 4, "nombre": "VENTAS" } }
  ],
  "page": 0, "size": 20, "totalElements": 1, "totalPages": 1,
  "first": true, "last": true
}

// PUT /api/v1/usuarios/12
{ "nombre": "María Gómez Ruiz", "correo": "maria.gomez@productionfood.local", "idRol": 4 }

// PATCH /api/v1/usuarios/12/estado
{ "activo": false }
```

**Campos ordenables** (lista blanca): `nombre`, `correo`, `id`.
Cualquier otro valor en `sort` → `400 CAMPO_ORDEN_INVALIDO`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `AUTO_DESACTIVACION` | 409 | Un administrador intenta desactivarse a sí mismo |
| `ULTIMO_ADMIN` | 409 | Se intenta desactivar al último administrador activo |
| `CORREO_DUPLICADO` | 409 | El correo ya pertenece a otro usuario |
| `ROL_INEXISTENTE` | 409 | El `idRol` no existe |

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

### Sobre la búsqueda por prefijo

CA-02 pide filtrar por nombre **o** correo. La implementación usa
`WHERE (nombre LIKE :q OR correo LIKE :q)` con patrón `texto%`.

Buscar "gomez" **no** encontrará a "María Gómez" — el apellido no está al inicio del campo
`nombre`. Es el compromiso consciente de `04-CONTRATO-API.md` §4: `%texto%` recorrería la
tabla completa en cada pulsación de tecla. Para el volumen de usuarios de una pyme la
diferencia es irrelevante, pero la convención se mantiene uniforme en todo el sistema.

Si en pruebas resulta incómodo, la alternativa correcta es un índice `FULLTEXT`, no cambiar
a `%texto%`.
