# HU-25 · Consulta de pedidos — Tarea Backend

Solo lectura. Amplía el listado de HU-16 con filtros combinables y agrega tres endpoints de agregación. Mismo patrón que HU-15.

---

## 1. Pasos

1. Ampliar `PedidoRepository.buscar()` con `idCliente`, `fechaInicio`, `fechaFin`,
   `estado`, combinados con `AND`.
2. El `SELECT` calcula tres importes por pedido:
   - `totalBruto` = `SUM(dp.cantidad * dp.precio_unitario)`;
   - `devoluciones` = `SUM(d.cantidad_devuelta * dp.precio_unitario)` sobre devoluciones
     `APROBADA`;
   - `totalNeto` = bruto − devoluciones (R-04).
3. `PedidoRepository.resumenPorPeriodo()` — agrupa por estado, excluye `CANCELADO` de la
   venta (R-03), calcula ticket promedio sobre los entregados.
4. `PedidoRepository.ventasPorCliente()` — `GROUP BY cliente`, ordenado por total.
5. `PedidoRepository.productosMasVendidos()` — `GROUP BY producto`, **ordenado por unidades**
   (R-07), con el importe como columna adicional.
6. `incluirDetalle=true` con un único `JOIN` y agrupación en memoria; `size` limitado a 20
   (R-06).
7. Validar `fechaInicio <= fechaFin` → `400 RANGO_FECHAS_INVALIDO`.
8. DTO: `ResumenPedidosResponse`, `VentasClienteResponse`, `ProductoVendidoResponse`.
9. `@Transactional(readOnly = true)`; sin endpoints de escritura (R-01).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── pedido/PedidoRepository.java     (+ buscar ampliado y 3 agregaciones)
├── pedido/PedidoService.java        (+ métodos de consulta)
├── pedido/PedidoController.java     (+ /resumen, /por-cliente, /productos-mas-vendidos)
└── reporte/dto/{ResumenPedidosResponse,VentasClienteResponse,ProductoVendidoResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **Leer `detalle_pedido.precio_unitario`, nunca `productos.precio`** (R-02). Si el
  `JOIN` trae el precio del catálogo, todo el histórico de ventas se recalcula solo y el
  reporte deja de ser reproducible.
- **Descontar las devoluciones aprobadas** (R-04), no las pendientes ni las rechazadas.
- **Las cancelaciones fuera de la venta** (R-03), y lo comprometido en cifra aparte.
- **Ranking por unidades, no por importe** (R-07).
- Evitar el N+1 en `incluirDetalle` (R-06).
- Verificar con `EXPLAIN` que se usa `idx_pedidos_fecha`.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
