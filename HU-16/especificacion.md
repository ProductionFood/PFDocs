# HU-16 · Creación de pedidos — Especificación

**Fase 6** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** vendedor,
> **quiero** crear un pedido para un cliente con fecha de entrega y estado,
> **para que** gestione las ventas.

**Depende de:** [HU-05](../HU-05/) (clientes)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | `id_cliente` debe existir **y estar activo** | Validado contra `ClienteRepository.existeActivo()` |
| CA-02 | `fecha_pedido` se registra automáticamente | `DEFAULT (curdate())` **añadido en V2** §C-07 |
| CA-03 | `fecha_entrega` es opcional | Nullable; si se informa, no puede ser anterior al pedido |
| CA-04 | `estado` obligatorio | Canónicos: `PENDIENTE`, `EN_PREPARACION`, `ENTREGADO`, `CANCELADO` — `CHECK` en V2 §C-12 |
| CA-05 | Se retorna el pedido creado con su `id_pedido` | `201` + `Location` + `PedidoResponse` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/pedidos` | Listar con filtros y paginación | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `GET` | `/api/v1/pedidos/{id}` | Obtener pedido con su detalle | ADMIN, VENTAS, PRODUCCION, CONSULTA |
| `POST` | `/api/v1/pedidos` | Crear un pedido | ADMIN, VENTAS |
| `PUT` | `/api/v1/pedidos/{id}` | Editar cabecera (solo si está `PENDIENTE`) | ADMIN, VENTAS |
| `PATCH` | `/api/v1/pedidos/{id}/estado` | Cambiar estado | ADMIN, VENTAS |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · La fecha del pedido la pone el servidor

CA-02 dice "automáticamente". V2 añadió `DEFAULT (curdate())` (§C-07), y **`curdate()` lo
evalúa el servidor de base de datos**, no la aplicación ni el navegador.

⚠️ Esto hace crítica la zona horaria: una instancia RDS está en UTC por defecto, y Colombia
es UTC-5. Un pedido registrado a las 7:30 p.m. se guardaría con la fecha del día siguiente,
y HU-24 CA-01 ("pedidos del día") dejaría de cuadrar con lo que ve el usuario — solo después
de las 7 p.m., así que las pruebas de la mañana no lo detectan.

La configuración obligatoria está en `00-base/11-DESPLIEGUE-AWS-RDS.md` §3.

### R-02 · Máquina de estados

```
PENDIENTE ──▶ EN_PREPARACION ──▶ ENTREGADO
    │                │
    └────────────────┴──────────▶ CANCELADO
```

`ENTREGADO` es terminal. Un pedido entregado no se cancela: si el cliente devuelve algo,
eso es una devolución (HU-18), que sí repone stock y deja registro.

Transición no permitida → `409 TRANSICION_INVALIDA`.

### R-03 · 🔑 Cuándo sale el stock del inventario

**Decisión de diseño que las historias no fijaban.** El descuento ocurre **al agregar el
producto al pedido** (HU-17 CA-04 verifica stock suficiente en ese momento), no al
entregarlo.

Razón: si se descontara al entregar, dos vendedores podrían comprometer el mismo pan con
dos clientes distintos y el conflicto aparecería al momento de la entrega, frente al
cliente. Descontar al agregar reserva la mercancía.

**Consecuencia:** cancelar un pedido **devuelve el stock** a los lotes de los que salió.
Lo gestiona HU-17; aquí solo se documenta porque el cambio de estado a `CANCELADO` es el
disparador.

### R-04 · Cancelar un pedido devuelve el stock
Al pasar a `CANCELADO` desde `PENDIENTE` o `EN_PREPARACION`, cada línea del detalle genera
un movimiento `ENTRADA_DEVOLUCION` en `movimientos_inventario_pt` sobre el lote original.
Transaccional.

Si el stock no volviera, cancelar un pedido haría desaparecer mercancía del sistema que
sigue en el estante.

### R-05 · La cabecera solo se edita mientras está `PENDIENTE`
Igual que en compras (HU-13 R-04). Una vez en preparación, cambiar el cliente del pedido
no tiene sentido operativo.

### R-06 · La fecha de entrega puede ser futura, la del pedido no
`fecha_entrega` es un compromiso a futuro: se permite cualquier fecha igual o posterior a
la del pedido. `fecha_pedido` la pone el servidor con la fecha de hoy.

### R-07 · Un cliente puede desactivarse después de tener pedidos
La validación de cliente activo (CA-01) es **solo al crear**. Los pedidos existentes de un
cliente desactivado siguen su curso hasta entregarse: son compromisos adquiridos
(HU-05 R-02).

---

## 4. Modelo de datos

**Tabla `pedidos`** (V2 añade el `DEFAULT` de fecha, el `CHECK` de estado y el índice)

| Columna | Tipo | Notas |
|---|---|---|
| `id_pedido` | `int unsigned` PK AI | |
| `id_cliente` | `int unsigned` NOT NULL FK | CA-01 |
| `fecha_pedido` | `date` NOT NULL **DEFAULT (curdate())** | CA-02 · V2 §C-07 · R-01 |
| `fecha_entrega` | `date` NULL | CA-03, R-06 |
| `estado` | `varchar(30)` NOT NULL | CA-04 · `CHECK` en V2 §C-12 |

```json
// POST /api/v1/pedidos
{ "idCliente": 7, "fechaEntrega": "2026-09-20" }

// 201 Created · Location: /api/v1/pedidos/45
{ "idPedido": 45,
  "cliente": { "idCliente": 7, "nombre": "Tienda La Esquina" },
  "fechaPedido": "2026-09-18", "fechaEntrega": "2026-09-20",
  "estado": "PENDIENTE", "totalPedido": 0.00, "cantidadItems": 0,
  "editable": true }

// PATCH /api/v1/pedidos/45/estado
{ "estado": "EN_PREPARACION" }
```

El cuerpo del `POST` **no acepta `fechaPedido` ni `estado`**: la fecha la pone la base
(R-01) y el estado inicial siempre es `PENDIENTE`.

**Campos ordenables:** `fechaPedido`, `fechaEntrega`, `estado`, `id`.

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `CLIENTE_INEXISTENTE` | 409 | El `idCliente` no existe |
| `CLIENTE_INACTIVO` | 409 | El cliente está desactivado (solo al crear) |
| `TRANSICION_INVALIDA` | 409 | Transición de estado no permitida (R-02) |
| `PEDIDO_NO_EDITABLE` | 409 | Se intenta modificar un pedido que no está `PENDIENTE` (R-05) |
| `FECHA_ENTREGA_INVALIDA` | 400 | `fechaEntrega` anterior a `fechaPedido` (R-06) |

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

### El pedido es el espejo de la compra

La estructura es la misma que HU-13: cabecera con estado, detalle en historia aparte
(HU-17), máquina de estados, y una transición que mueve inventario.

La diferencia está en **cuándo** se mueve el stock:

| | Compra (HU-13) | Pedido (HU-16/17) |
|---|---|---|
| Momento | Al **recibir** (cambio de estado) | Al **agregar la línea** (R-03) |
| Dirección | Entrada | Salida |
| Reversible | No (`RECIBIDA` es terminal) | Sí (cancelar devuelve stock, R-04) |

La asimetría es deliberada: la mercancía comprada no existe hasta que llega, pero la
mercancía vendida hay que apartarla en cuanto se compromete.
