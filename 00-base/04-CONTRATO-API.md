# 04 — Contrato de API

Reglas transversales a todos los endpoints. Cada `especificacion.md` de HU define su
contrato específico; **este documento define lo que no se repite en cada uno**.

---

## 1. Base y versionado

```
http://localhost:8080/api/v1
```

- Todos los recursos cuelgan de `/api/v1`.
- Producción/JSON en `application/json; charset=UTF-8`.
- Swagger UI en `/swagger-ui.html`, OpenAPI en `/v3/api-docs`.

---

## 2. Autenticación

Todos los endpoints requieren JWT salvo `POST /api/v1/auth/login`.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

**Claims del token** (HU-03 CA-05):

```json
{
  "sub": "12",
  "correo": "maria@productionfood.local",
  "nombre": "María Gómez",
  "rol": "VENTAS",
  "iat": 1757980800,
  "exp": 1757984400
}
```

- `sub` es el `id_usuario` como string (es el estándar de JWT).
- Vigencia: **1 hora**. No hay refresh token en este alcance; al expirar se vuelve a
  iniciar sesión.
- El secreto de firma viene de `application.properties` vía variable de entorno.
  **Nunca se versiona en el repositorio.**

---

## 3. Métodos y códigos de estado

| Método | Uso | Éxito | Cuerpo de respuesta |
|---|---|---|---|
| `GET` (colección) | Listar | `200` | `PageResponse<T>` |
| `GET` (recurso) | Obtener uno | `200` | `T` |
| `POST` | Crear | `201` + cabecera `Location` | El recurso creado |
| `PUT` | Reemplazar completo | `200` | El recurso actualizado |
| `PATCH` | Modificar parcial (típicamente estado) | `200` | El recurso actualizado |
| `DELETE` | Eliminar línea de detalle | `204` | Vacío |

**No se usa `DELETE` sobre entidades principales.** Las HU son explícitas: clientes,
proveedores y usuarios se **desactivan** (`estado = 0`), no se borran. `DELETE` queda
reservado a líneas de detalle (ítems de receta, de compra, de pedido).

### Códigos de error

| Código | Cuándo | Ejemplo típico |
|---|---|---|
| `400` | Petición malformada o validación sintáctica fallida | Falta un campo obligatorio, formato inválido |
| `401` | Sin token, token inválido o expirado | Sesión vencida |
| `403` | Autenticado pero sin permiso, o usuario inactivo | `VENTAS` intentando crear un producto |
| `404` | El recurso no existe | `GET /clientes/9999` |
| `409` | Conflicto con el estado actual | Correo duplicado, stock insuficiente, transición de estado inválida |
| `422` | Regla de negocio violada con datos sintácticamente válidos | Devolver más cantidad de la pedida |
| `500` | Error no controlado | — |

**`409` vs `422`:** `409` es conflicto con algo que ya existe en el sistema (duplicado,
estado incompatible). `422` es una regla de negocio que los datos no satisfacen aunque el
formato sea correcto. Ante la duda, `409`. Lo que importa es ser consistente: cada
`especificacion.md` fija el código de cada caso.

---

## 4. Listados: paginación, filtros y ordenamiento

### Parámetros

| Parámetro | Tipo | Por defecto | Restricción |
|---|---|---|---|
| `page` | int | `0` | `>= 0` |
| `size` | int | `20` | `1..100` — por encima de 100 se devuelve `400` |
| `sort` | string | Según el recurso | `campo,asc` o `campo,desc` |

`size` se acota por arriba deliberadamente: sin ese límite, un `?size=999999` convierte
cualquier listado en un ataque de denegación trivial.

### Respuesta de colección

```json
{
  "content": [ { "...": "..." } ],
  "page": 0,
  "size": 20,
  "totalElements": 143,
  "totalPages": 8,
  "first": true,
  "last": false
}
```

### Ordenamiento seguro

El valor de `sort` **nunca se concatena crudo al SQL**. Cada repositorio declara su lista
blanca:

```java
private static final Map<String, String> CAMPOS_ORDEN = Map.of(
    "nombre", "c.nombre",
    "id",     "c.id_cliente"
);
// Un campo fuera del mapa → 400 CAMPO_ORDEN_INVALIDO
```

### Filtros de texto

Los filtros por nombre son **búsqueda por prefijo**: `LIKE 'texto%'`.

Dos razones: es lo que el usuario espera al teclear en un buscador, y es lo único que
puede aprovechar un índice B-tree. `LIKE '%texto%'` fuerza recorrido completo de la tabla
en cada pulsación (ver `02-CORRECCIONES-DB.md` §C-15).

Los caracteres `%` y `_` del valor recibido se escapan antes de armar el patrón; si no,
un usuario que escriba `%` obtiene todos los registros.

### Filtros de fecha

`fechaInicio` y `fechaFin` en formato `YYYY-MM-DD`, **ambos inclusivos**. Si
`fechaInicio > fechaFin` → `400 RANGO_FECHAS_INVALIDO`.

---

## 5. Formato de error

Único para toda la API:

```json
{
  "timestamp": "2026-09-18T14:32:11",
  "status": 409,
  "error": "Conflict",
  "code": "CORREO_DUPLICADO",
  "message": "Ya existe un usuario registrado con el correo indicado.",
  "path": "/api/v1/usuarios",
  "fieldErrors": []
}
```

Con errores de validación:

```json
{
  "timestamp": "2026-09-18T14:35:02",
  "status": 400,
  "error": "Bad Request",
  "code": "VALIDACION_FALLIDA",
  "message": "Los datos enviados no son válidos.",
  "path": "/api/v1/usuarios",
  "fieldErrors": [
    { "field": "correo", "message": "Debe ser un correo electrónico válido" },
    { "field": "nombre", "message": "No puede estar vacío" }
  ]
}
```

Reglas:

- `code` es estable y en inglés SCREAMING_SNAKE: es para el código del frontend.
- `message` es para la persona: español, claro, sin jerga técnica.
- **`message` nunca expone detalles internos.** Nada de nombres de tabla, SQL, ni
  *stack traces*. Un `DataIntegrityViolationException` se traduce a un mensaje de dominio;
  el detalle técnico va al log del servidor, no a la respuesta.

### Códigos de error transversales

| `code` | HTTP | Significado |
|---|---|---|
| `VALIDACION_FALLIDA` | 400 | Bean Validation rechazó el cuerpo |
| `PARAMETRO_INVALIDO` | 400 | Parámetro de consulta mal formado |
| `CAMPO_ORDEN_INVALIDO` | 400 | `sort` fuera de la lista blanca |
| `RANGO_FECHAS_INVALIDO` | 400 | `fechaInicio` posterior a `fechaFin` |
| `NO_AUTENTICADO` | 401 | Token ausente, inválido o expirado |
| `SIN_PERMISO` | 403 | Rol insuficiente |
| `USUARIO_INACTIVO` | 403 | Usuario con `estado = 0` |
| `RECURSO_NO_ENCONTRADO` | 404 | El id no existe |
| `REFERENCIA_INEXISTENTE` | 409 | Una FK del cuerpo apunta a algo que no existe |
| `ERROR_INTERNO` | 500 | No controlado |

Los códigos específicos de cada HU están en su `especificacion.md`.

---

## 6. Convenciones de datos

| Tipo | Formato JSON | Ejemplo |
|---|---|---|
| Fecha | `YYYY-MM-DD` (ISO-8601) | `"2026-09-18"` |
| Fecha y hora | `YYYY-MM-DDTHH:mm:ss` | `"2026-09-18T14:32:11"` |
| Decimal | Número JSON, sin separador de miles | `12500.50` |
| Booleano de estado | **`boolean`**, no `0`/`1` | `"activo": true` |
| Identificador | Número entero | `42` |
| Enumerado | String SCREAMING_SNAKE | `"EN_PREPARACION"` |

Notas:

- **Decimales.** En Java se usa `BigDecimal`, nunca `double` ni `float`. Un `double` no
  representa exactamente `0.1`; sumar precios con `double` produce centavos fantasma que
  aparecen como descuadres al cuadrar caja. En TypeScript se recibe como `number` y se
  formatea al mostrar; los cálculos de totales son autoridad del backend.
- **`estado`.** La base guarda `tinyint(1)`; la API expone `boolean`. La traducción se
  hace en el `RowMapper` y en el DTO. El frontend nunca ve un `1` cuando quiere decir "sí".
- **Zona horaria.** Las fechas se manejan como fecha local de Colombia (UTC-5). El
  servidor se configura con `-Duser.timezone=America/Bogota`. Un servidor en UTC hace que
  los pedidos registrados después de las 7 p.m. aparezcan con la fecha del día siguiente,
  y HU-24 CA-01 ("pedidos del día") deja de cuadrar con lo que ve el usuario.

---

## 7. Estados y transiciones

Los valores canónicos están en `02-CORRECCIONES-DB.md` §C-12. Las transiciones permitidas:

### Compras (HU-13)
```
PENDIENTE ──▶ RECIBIDA      (suma al inventario · irreversible)
    │
    └───────▶ CANCELADA
```
`RECIBIDA` es terminal: una vez que el stock entró, se corrige con un ajuste de
inventario, nunca deshaciendo el estado.

### Pedidos (HU-16)
```
PENDIENTE ──▶ EN_PREPARACION ──▶ ENTREGADO
    │                │
    └────────────────┴──────────▶ CANCELADO   (devuelve stock reservado)
```
`ENTREGADO` es terminal.

### Planes de producción (HU-20)
```
PLANIFICADO ──▶ EN_PROCESO ──▶ COMPLETADO
     │               │
     └───────────────┴───────▶ CANCELADO
```

### Devoluciones (HU-18)
```
PENDIENTE ──▶ APROBADA     (devuelve stock · HU-18 CA-07)
    │
    └───────▶ RECHAZADA
```

**Toda transición no listada devuelve `409 TRANSICION_INVALIDA`**, con un mensaje que
indique el estado actual y los permitidos:

```json
{
  "code": "TRANSICION_INVALIDA",
  "message": "No se puede pasar de ENTREGADO a PENDIENTE. Desde ENTREGADO no hay transiciones disponibles."
}
```

La validación vive en un único método por módulo (`validarTransicion(actual, nuevo)`),
no dispersa en `if` por el servicio.

---

## 8. Idempotencia y concurrencia

- `GET`, `PUT` y `DELETE` son idempotentes por definición.
- `POST` no lo es: dos clics en "Guardar" crean dos registros. Mitigación: el frontend
  deshabilita el botón mientras la petición está en vuelo, **y** las restricciones
  `UNIQUE` de la base son la garantía real (`02-CORRECCIONES-DB.md` §C-05, §C-13).
- **Descuentos de stock**: la comprobación de disponibilidad y el descuento ocurren en la
  **misma transacción**, con `SELECT ... FOR UPDATE` sobre la fila de inventario. Sin ese
  bloqueo, dos pedidos simultáneos del último pan disponible pasan ambos la validación y
  el stock queda negativo. El `CHECK (cantidad_disponible >= 0)` es la segunda línea de
  defensa.

---

## 9. CORS

Solo en desarrollo, y acotado:

```java
configuration.setAllowedOrigins(List.of("http://localhost:4200"));
configuration.setAllowedMethods(List.of("GET","POST","PUT","PATCH","DELETE","OPTIONS"));
configuration.setAllowedHeaders(List.of("Authorization","Content-Type"));
```

`setAllowedOrigins(List.of("*"))` combinado con credenciales no funciona y, donde
funciona, abre la API a cualquier sitio. No se usa.

---

## 10. Documentación en Swagger

Cada endpoint lleva:

```java
@Operation(summary = "Registrar un cliente",
           description = "Crea un cliente activo. El nombre es obligatorio.")
@ApiResponses({
    @ApiResponse(responseCode = "201", description = "Cliente creado"),
    @ApiResponse(responseCode = "400", description = "Datos inválidos"),
    @ApiResponse(responseCode = "403", description = "Sin permisos")
})
```

Un endpoint sin documentar no cumple la Definición de Terminado
(`01-CONVENCIONES.md` §7). Swagger es el contrato que consume el frontend y con el que QA
arma las pruebas: si está incompleto, dos personas quedan bloqueadas.
