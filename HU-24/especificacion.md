# HU-24 · Dashboard resumen — Especificación

**Fase 8** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** un dashboard con resumen del día (pedidos del día, compras pendientes, stock bajo, producción del día),
> **para que** tenga visión general de la operación.

**Depende de:** [HU-13](../HU-13/), [HU-16](../HU-16/), [HU-20](../HU-20/)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Total de pedidos del día actual | `WHERE fecha_pedido = curdate()` |
| CA-02 | Compras con estado "Pendiente" | `WHERE estado = 'PENDIENTE'` — sin filtro de fecha |
| CA-03 | Materias primas con stock por debajo del mínimo | Vista `v_inventario_materia_prima` (V2) |
| CA-04 | Planes de producción del día actual | `WHERE fecha_produccion = curdate()` |
| CA-05 | Productos próximos a vencer (próximos 7 días) | `fecha_vencimiento BETWEEN curdate() AND curdate() + 7 días` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `GET` | `/api/v1/dashboard` | Resumen completo del día | Todos los roles autenticados |
| `GET` | `/api/v1/dashboard/pedidos-hoy` | Detalle de los pedidos del día | ADMIN, VENTAS, CONSULTA |
| `GET` | `/api/v1/dashboard/vencimientos` | Productos próximos a vencer | ADMIN, VENTAS, PRODUCCION, CONSULTA |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · ⚠️ "El día actual" depende de la zona horaria del servidor

CA-01 y CA-04 comparan contra `curdate()`, que evalúa **el servidor de base de datos**.

Una instancia RDS está en UTC por defecto; Colombia es UTC-5. Entre las 7 p.m. y la
medianoche, `curdate()` devuelve **el día siguiente**, y el dashboard mostraría "0 pedidos
hoy" mientras el vendedor acaba de registrar tres.

Configuración obligatoria en `00-base/11-DESPLIEGUE-AWS-RDS.md` §3. Es el mismo riesgo de
HU-16 R-01, y aquí se hace visible de forma inmediata.

### R-02 · Cada tarjeta responde a una pregunta operativa

| Tarjeta | Pregunta | Filtro |
|---|---|---|
| Pedidos de hoy | ¿Cuánto se vendió hoy? | `fecha_pedido = curdate()` |
| Compras pendientes | ¿Qué mercancía está por llegar? | `estado = 'PENDIENTE'`, **sin fecha** |
| Stock bajo | ¿Qué hay que comprar? | `estado_stock IN ('BAJO','AGOTADO')` |
| Producción de hoy | ¿Qué se está fabricando? | `fecha_produccion = curdate()` |
| Por vencer | ¿Qué hay que sacar ya? | Vence en ≤ 7 días **y tiene stock** |

**Compras pendientes no lleva filtro de fecha** (CA-02): una compra pendiente de hace un mes
es precisamente la que hay que revisar.

### R-03 · Los productos por vencer solo cuentan si tienen stock
Un lote vencido sin existencias no requiere acción. La consulta exige
`cantidad_disponible > 0`; si no, el dashboard mostraría alarmas sobre lotes ya agotados.

Se incluyen también **los ya vencidos con stock**, en rojo: son los que hay que retirar del
estante hoy mismo.

### R-04 · El dashboard es de solo lectura y debe ser rápido
Cinco consultas independientes. Todas deben responder en conjunto por debajo de los 2
segundos del RNF-02.

- Las cinco se ejecutan **en paralelo** desde el cliente, o en un único endpoint que las
  agrupe.
- Todas se apoyan en índices añadidos en V2 §C-15: `idx_pedidos_fecha`,
  `idx_compras_fecha`, `idx_planes_fecha`, `idx_lotes_vencimiento`.
- **Sin caché**: un dashboard operativo que muestre datos de hace cinco minutos hace tomar
  decisiones sobre información falsa.

### R-05 · El contenido se adapta al rol
Todos los roles acceden, pero ven lo que les concierne:

| Rol | Tarjetas |
|---|---|
| `ADMIN` | Las cinco |
| `VENTAS` | Pedidos de hoy, por vencer |
| `COMPRAS` | Compras pendientes, stock bajo |
| `PRODUCCION` | Producción de hoy, stock bajo, por vencer |
| `CONSULTA` | Las cinco, solo lectura |

Es filtrado de presentación; los endpoints subyacentes mantienen su propia autorización.

### R-06 · Los importes se calculan sobre el detalle
El total de pedidos del día se calcula con `SUM(cantidad * precio_unitario)` sobre
`detalle_pedido` — posible gracias a la columna añadida en V2 §C-01. Sin ella, el dashboard
no podría mostrar ninguna cifra de ventas.

---

## 4. Modelo de datos

Sin cambios de esquema. Consulta `pedidos`, `compras`, `planes_produccion`, `lotes` y las
vistas de V2.

```json
// GET /api/v1/dashboard
{
  "fecha": "2026-09-18",
  "pedidosHoy": {
    "cantidad": 7, "total": 284500.00,
    "porEstado": { "PENDIENTE": 3, "EN_PREPARACION": 2, "ENTREGADO": 2, "CANCELADO": 0 }
  },
  "comprasPendientes": {
    "cantidad": 2, "total": 620000.00,
    "masAntigua": { "idCompra": 10, "fecha": "2026-09-02", "diasPendiente": 16 }
  },
  "stockBajo": {
    "cantidad": 3, "agotadas": 1,
    "items": [
      { "nombre": "Levadura", "disponible": 0.00, "minimo": 5.00,
        "faltante": 5.00, "nivel": "AGOTADO" },
      { "nombre": "Harina de trigo", "disponible": 46.30, "minimo": 50.00,
        "faltante": 3.70, "nivel": "BAJO" }
    ]
  },
  "produccionHoy": {
    "cantidad": 2,
    "porEstado": { "PLANIFICADO": 1, "EN_PROCESO": 1, "COMPLETADO": 0 },
    "avancePromedio": 45.50
  },
  "porVencer": {
    "cantidad": 4, "vencidos": 1,
    "items": [
      { "codigoLote": "CR-2026-0914", "producto": "Croissant",
        "vence": "2026-09-16", "diasParaVencer": -2, "disponible": 8.00 },
      { "codigoLote": "PF-2026-0917", "producto": "Pan francés",
        "vence": "2026-09-21", "diasParaVencer": 3, "disponible": 40.00 }
    ]
  }
}
```

`diasParaVencer` negativo indica lote ya vencido (R-03).

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|


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

### Este dashboard es la prueba de fuego de las correcciones del esquema

Cada tarjeta depende de algo que el `db.sql` original no permitía:

| Tarjeta | Depende de |
|---|---|
| Total de pedidos de hoy | `detalle_pedido.precio_unitario` (§C-01) |
| Stock bajo | Que exista fila de inventario por materia prima (HU-08 R-01) y que las compras sumen (§C-03) |
| Por vencer con stock | `inventario_producto_terminado` poblado desde HU-19 |
| Producción de hoy | Estados canónicos (§C-12) |

Si alguna tarjeta muestra ceros persistentes, el problema no suele estar en esta historia
sino en la corrección que la sostiene.
