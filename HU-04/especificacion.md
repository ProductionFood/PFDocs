# HU-04 · Bitácora de auditoría — Especificación

**Fase 1** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** sistema,
> **quiero** registrar en bitácora cada acción crítica (crear, editar, eliminar),
> **para que** exista trazabilidad de las operaciones.

**Depende de:** [HU-03](../HU-03/) (se necesita saber quién actúa)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Se registra automáticamente al crear, editar o eliminar | Aspecto AOP sobre los métodos anotados con `@Auditable` |
| CA-02 | Se almacena `id_usuario`, `accion`, `tabla_afectada`, `detalle`, `fecha` | Tabla `bitacora` — sin cambios de columnas |
| CA-03 | La fecha se registra con `current_timestamp()` por defecto | Ya está en el esquema original |
| CA-04 | La bitácora es de **solo lectura** | Solo existe `GET`. No hay `POST`, `PUT` ni `DELETE` de bitácora |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/bitacora` | Consultar con filtros y paginación | ADMIN |
| `GET` | `/api/v1/bitacora/acciones` | Catálogo de acciones registradas | ADMIN |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · `id_usuario` puede ser nulo: significa "sistema"

El esquema original tenía `id_usuario NOT NULL`, lo que hacía **imposible auditar**:
- la creación del administrador semilla,
- los intentos de login fallidos (HU-03) — no hay usuario válido que registrar,
- los procesos automáticos.

Corregido en V2 (`02-CORRECCIONES-DB.md` §C-11): la columna admite `NULL`, y `NULL` se
muestra como **"Sistema"** en la interfaz.

### R-02 · Se audita después del éxito, no antes

El aspecto usa `@AfterReturning`, no `@Before`. Auditar antes de ejecutar registraría como
cambios reales operaciones que después fallaron y se revirtieron, llenando la bitácora de
eventos que nunca ocurrieron.

### R-03 · Un fallo de auditoría no debe tumbar la operación de negocio

`BitacoraService.registrar()` captura sus propias excepciones y las envía al log de la
aplicación. Que la auditoría falle es un problema; que impida registrar una venta es peor.

**Consecuencia asumida:** la bitácora puede tener huecos si la base de datos falla al
escribirla. Es el compromiso deliberado, y por eso la trazabilidad del **inventario** no
descansa en la bitácora sino en el kardex (§C-04), que sí es transaccional.

### R-04 · `detalle` está limitado a 255 caracteres
La columna es `varchar(255)`. El resumen se **recorta** antes de insertar. Sin recortar, con
`sql_mode` estricto de MySQL 8 el `INSERT` **falla** en lugar de truncar, y por R-03 el
fallo se tragaría en silencio: la acción no quedaría auditada.

### R-05 · Nunca se auditan datos sensibles
`detalle` **no** incluye contraseñas, hashes ni tokens. El resumen de una creación de
usuario registra el correo y el rol, nunca la credencial. Una bitácora con contraseñas es
peor que no tener bitácora: concentra en una tabla consultable lo que el resto del sistema
protege.

### R-06 · Acciones que se auditan

| Acción | Cuándo |
|---|---|
| `CREAR` | Alta de cualquier entidad |
| `EDITAR` | Modificación |
| `CAMBIAR_ESTADO` | Activar/desactivar, transiciones de estado |
| `ELIMINAR` | Borrado de líneas de detalle |
| `LOGIN` / `LOGIN_FALLIDO` | HU-03 |
| `RECEPCION_COMPRA` | HU-13 — entra stock |
| `APROBAR_DEVOLUCION` | HU-18 — vuelve stock |
| `REGISTRAR_CONSUMO` | HU-22 — sale stock |

Las consultas (`GET`) **no se auditan**: generarían un volumen enorme sin aportar
trazabilidad de cambios.

---

## 4. Modelo de datos

**Tabla `bitacora`** (columnas sin cambios; solo `id_usuario` pasa a nullable en V2)

| Columna | Tipo | Notas |
|---|---|---|
| `id_bitacora` | `int unsigned` PK AI | |
| `id_usuario` | `int unsigned` **NULL** | `NULL` = sistema (R-01, §C-11) |
| `accion` | `varchar(50)` NOT NULL | R-06 |
| `tabla_afectada` | `varchar(50)` NOT NULL | |
| `detalle` | `varchar(255)` NULL | Recortado (R-04) |
| `fecha` | `datetime` NOT NULL DEFAULT `CURRENT_TIMESTAMP` | CA-03 |

Índice añadido en V2: `idx_bitacora_tabla (tabla_afectada, fecha)`.

```json
// GET /api/v1/bitacora?tabla=usuarios&accion=CREAR&fechaInicio=2026-09-01&fechaFin=2026-09-18
{
  "content": [
    { "idBitacora": 145, "usuario": { "idUsuario": 1, "nombre": "Administrador Inicial" },
      "accion": "CREAR", "tablaAfectada": "usuarios",
      "detalle": "Usuario creado: maria@productionfood.local, rol VENTAS",
      "fecha": "2026-09-18T14:32:11" },
    { "idBitacora": 144, "usuario": null,
      "accion": "LOGIN_FALLIDO", "tablaAfectada": "usuarios",
      "detalle": "Intento fallido para el correo: desconocido@x.local",
      "fecha": "2026-09-18T14:30:02" }
  ],
  "page": 0, "size": 20, "totalElements": 2, "totalPages": 1
}
```

`usuario: null` se muestra como **"Sistema"**.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `ACCESO_DENEGADO_BITACORA` | 403 | Un rol distinto de ADMIN intenta consultarla |

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

### Por qué esta historia va en el Sprint 1 y no al final

La auditoría es **transversal**: se implementa como aspecto AOP y se activa anotando los
métodos de servicio de cada módulo.

Si se deja para el final hay que volver a abrir los veinte servicios ya escritos, anotarlos
uno por uno y volver a probarlos. Implementada al principio, cada historia posterior solo
agrega su `@Auditable` mientras se escribe.

Está en el Sprint 1 del `00-base/10-ROADMAP.md` por esta razón.
