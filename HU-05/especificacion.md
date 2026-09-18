# HU-05 · Gestión de clientes — Especificación

**Fase 2** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** crear, consultar, editar y desactivar clientes (nombre, contacto, teléfono, estado),
> **para que** lleve el registro de quiénes compran.

**Depende de:** [HU-03](../HU-03/), [HU-04](../HU-04/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Nombre obligatorio, máx. 100 | `@NotBlank` + `@Size(max=100)` |
| CA-02 | Contacto y teléfono opcionales | Sin `@NotBlank`; `@Size` para el largo |
| CA-03 | Estado por defecto activo (`1`) | `DEFAULT 1` ya presente en el esquema original |
| CA-04 | Listar con paginación y filtrar por nombre | `GET /clientes?nombre=&page=&size=&sort=` |
| CA-05 | **No se permite eliminar**, solo desactivar | No existe endpoint `DELETE` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/clientes` | Listar con filtro y paginación | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/clientes/{id}` | Obtener un cliente | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/clientes` | Registrar | ADMIN, VENTAS |
| `PUT` | `/api/v1/clientes/{id}` | Editar | ADMIN, VENTAS |
| `PATCH` | `/api/v1/clientes/{id}/estado` | Activar o desactivar | ADMIN, VENTAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Nunca se elimina (CA-05)
`pedidos.id_cliente` es una FK sin `ON DELETE CASCADE`: borrar un cliente con pedidos
fallaría por integridad referencial, y forzarlo destruiría el historial de ventas.
La desactivación (`estado = 0`) conserva todo.

### R-02 · Un cliente inactivo no puede recibir pedidos nuevos
HU-16 CA-01 exige que el cliente exista **y esté activo**. Desactivar es la forma de
retirar a un cliente sin perder sus pedidos anteriores.

Los pedidos ya creados de un cliente que se desactiva **no se cancelan**: siguen su curso
normal. Desactivar impide nuevas ventas, no anula compromisos adquiridos.

### R-03 · Se permiten nombres duplicados
No hay `UNIQUE` sobre `nombre`, y es deliberado: dos clientes pueden llamarse igual
("Tienda La Esquina" hay varias). La interfaz muestra un aviso no bloqueante cuando detecta
un nombre idéntico, para que quien registra decida.

### R-04 · El teléfono se guarda tal como se escribe
`varchar(20)` sin validación de formato. Un teléfono puede ser fijo, celular, con
indicativo o con extensión. Imponer un formato rígido obliga a falsear datos cuando el
número real no encaja. Se valida solo la longitud.

---

## 4. Modelo de datos

**Tabla `clientes`** (sin cambios de esquema; V2 añade el índice `idx_clientes_nombre`)

| Columna | Tipo | Notas |
|---|---|---|
| `id_cliente` | `int unsigned` PK AI | |
| `nombre` | `varchar(100)` NOT NULL | CA-01 |
| `contacto` | `varchar(100)` NULL | CA-02 |
| `telefono` | `varchar(20)` NULL | CA-02, R-04 |
| `estado` | `tinyint(1)` NOT NULL DEFAULT 1 | CA-03 |

```json
// POST /api/v1/clientes
{ "nombre": "Tienda La Esquina", "contacto": "Pedro Ruiz", "telefono": "300 555 1234" }

// 201 Created · Location: /api/v1/clientes/7
{ "idCliente": 7, "nombre": "Tienda La Esquina",
  "contacto": "Pedro Ruiz", "telefono": "300 555 1234", "activo": true }
```

**Campos ordenables:** `nombre`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `CLIENTE_CON_PEDIDOS_ACTIVOS` | 409 | Se intenta desactivar un cliente con pedidos en curso (advertencia configurable) |

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

### Esta historia es la plantilla de todos los CRUD del proyecto

HU-06 (proveedores), HU-07 (unidades), HU-08 (materias primas) y HU-11 (productos) tienen
la misma estructura. El `00-base/10-ROADMAP.md` indica implementar **esta primero y
completa** —backend, frontend y pruebas— y usarla como molde.

Hacer los cinco en paralelo desde cero multiplica por cinco las decisiones de diseño y
produce cinco estilos distintos que después hay que unificar.
