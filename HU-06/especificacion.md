# HU-06 · Gestión de proveedores — Especificación

**Fase 2** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** CRUD de proveedores (nombre, contacto, teléfono, correo, estado),
> **para que** gestione quiénes nos suministran materia prima.

**Depende de:** [HU-05](../HU-05/) (mismo patrón de CRUD)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Nombre obligatorio, máx. 100 | `@NotBlank` + `@Size(max=100)` |
| CA-02 | Contacto, teléfono y correo opcionales | Sin `@NotBlank`; `@Email` solo si viene informado |
| CA-03 | Estado por defecto activo (`1`) | `DEFAULT 1` ya presente en el esquema |
| CA-04 | Listar con paginación y filtrar por nombre | `GET /proveedores?nombre=&page=&size=` |
| CA-05 | **No se permite eliminar**, solo desactivar | No existe endpoint `DELETE` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/proveedores` | Listar con filtro y paginación | ADMIN, COMPRAS, CONSULTA |
| `GET` | `/api/v1/proveedores/{id}` | Obtener uno | ADMIN, COMPRAS, CONSULTA |
| `POST` | `/api/v1/proveedores` | Registrar | ADMIN, COMPRAS |
| `PUT` | `/api/v1/proveedores/{id}` | Editar | ADMIN, COMPRAS |
| `PATCH` | `/api/v1/proveedores/{id}/estado` | Activar o desactivar | ADMIN, COMPRAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Nunca se elimina (CA-05)
`compras.id_proveedor` es FK sin cascada. Borrar un proveedor con compras rompería el
historial de gastos que HU-15 consulta.

### R-02 · Un proveedor inactivo no admite compras nuevas
HU-13 CA-01 valida que el proveedor exista. Se añade la verificación de que esté **activo**:
desactivar sirve para dejar de comprarle sin perder el historial.

Las compras ya registradas siguen su curso, incluidas las `PENDIENTE` de recibir. Desactivar
un proveedor no cancela lo que ya se le pidió.

### R-03 · El correo es opcional pero, si viene, debe ser válido
`@Email` en Bean Validation acepta `null` y cadena vacía sin protestar; solo valida cuando
hay contenido. Es el comportamiento correcto para un campo opcional. **No hay `UNIQUE`**:
dos sucursales del mismo proveedor pueden compartir correo.

### R-04 · Diferencia con clientes
Proveedores tiene `correo` y clientes no. Es la única diferencia estructural entre ambas
tablas; el resto del CRUD es idéntico a HU-05 y se implementa copiando ese patrón.

---

## 4. Modelo de datos

**Tabla `proveedores`** (sin cambios; V2 añade `idx_proveedores_nombre`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_proveedor` | `int unsigned` PK AI | |
| `nombre` | `varchar(100)` NOT NULL | CA-01 |
| `contacto` | `varchar(100)` NULL | CA-02 |
| `telefono` | `varchar(20)` NULL | CA-02 |
| `correo` | `varchar(100)` NULL | CA-02, R-03 |
| `estado` | `tinyint(1)` NOT NULL DEFAULT 1 | CA-03 |

```json
// POST /api/v1/proveedores
{ "nombre": "Harinas del Caribe S.A.S.", "contacto": "Jorge Mendoza",
  "telefono": "605 278 4410", "correo": "ventas@harinascaribe.com" }

// 201 Created
{ "idProveedor": 3, "nombre": "Harinas del Caribe S.A.S.", "contacto": "Jorge Mendoza",
  "telefono": "605 278 4410", "correo": "ventas@harinascaribe.com", "activo": true }
```

**Campos ordenables:** `nombre`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `PROVEEDOR_INACTIVO` | 409 | Se intenta registrar una compra a un proveedor desactivado (validado en HU-13) |

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

### Sobre el alcance

El documento de Entrega 1 excluía explícitamente la gestión de proveedores
(§3.2 Limitaciones). Se incluye porque **sin compras no entra materia prima al inventario**
y tres historias más quedarían sin sentido. Justificación completa en
`00-base/08-DESVIACIONES-PDF.md` §D-01.
