# HU-17 · Agregar productos al pedido — Tarea Backend

**La historia más delicada del proyecto junto con HU-23.** Aquí nace `InventarioPtService`, el equivalente para producto terminado del `InventarioService` de HU-09. Requiere FEFO, bloqueo de filas y ajuste por diferencia.

---

## 1. Pasos

### `InventarioPtService` (se crea aquí; lo usan HU-16, HU-18 y HU-19)
1. `registrarSalidaFefo(idProducto, cantidad, tablaOrigen, idOrigen)`:
   - consulta `v_lotes_disponibles_fefo` **con `ORDER BY fecha_vencimiento ASC, id_lote ASC`
     en la propia consulta** (R-02: la vista no ordena por sí sola);
   - `SELECT ... FOR UPDATE` sobre las filas de `inventario_producto_terminado` implicadas;
   - valida stock total suficiente → `STOCK_INSUFICIENTE`;
   - recorre los lotes descontando hasta cubrir la cantidad;
   - por cada lote consumido: `UPDATE` del saldo + `INSERT` en `movimientos_inventario_pt`
     con tipo `SALIDA_VENTA`;
   - devuelve la lista de lotes consumidos.
2. `registrarEntrada(idLote, cantidad, tipo, tablaOrigen, idOrigen)` — para devoluciones.
3. **Regla de oro:** ningún otro componente ejecuta `UPDATE inventario_producto_terminado`.

### Módulo `detalle-pedido`
4. `DetallePedido` (POJO) y `DetallePedidoRowMapper` con `JOIN productos`.
5. `DetallePedidoRepository`: `listarPorPedido`, `insertar`, `actualizarCantidad`,
   `eliminar`, `existeProducto(idPedido, idProducto)`, `calcularTotal(idPedido)`.
6. DTO: `AgregarDetallePedidoRequest`, `ActualizarCantidadRequest`,
   `DetallePedidoResponse` (con `lotesConsumidos`), `StockProductoResponse`.
7. 🔑 **`DetallePedidoService.agregar()` `@Transactional`**:
   - pedido existe y es editable → `PEDIDO_NO_EDITABLE`;
   - producto existe y activo;
   - no duplicado → `PRODUCTO_DUPLICADO`;
   - **copia `productos.precio` en `precio_unitario`** (R-04);
   - `registrarSalidaFefo(...)`;
   - `INSERT` en `detalle_pedido`.
8. `actualizarCantidad()` con **ajuste por diferencia** (R-07):
   - `nueva > actual` → salida FEFO por la diferencia;
   - `nueva < actual` → entrada a los lotes de origen por la diferencia,
     consultando el kardex para saber a cuáles volver;
   - **no** devolver todo y volver a descontar.
9. `eliminar()`: devuelve la cantidad completa a los lotes de origen (R-08).
10. Consulta de stock desde `v_stock_producto_terminado`.
11. `@Auditable` en las tres escrituras.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── inventario/
│   ├── InventarioPtService.java              ← 🔑 FEFO y bloqueo
│   ├── InventarioPtRepository.java
│   └── MovimientoInventarioPtRepository.java
└── pedido/
    ├── {DetallePedido,DetallePedidoRowMapper,DetallePedidoRepository}.java
    ├── {DetallePedidoService,DetallePedidoController}.java
    └── dto/{AgregarDetallePedidoRequest,ActualizarCantidadRequest,
            DetallePedidoResponse,StockProductoResponse,LoteConsumidoResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **El `ORDER BY` va en la consulta, no en la vista** (R-02). MySQL ignora el `ORDER BY`
  interno de una vista **sin dar ningún aviso**: la vista se crea, devuelve filas, y el orden
  es arbitrario. FEFO deja de aplicarse y nadie se entera hasta que caduca producto en el
  estante.
- 🔴 **`SELECT ... FOR UPDATE` antes de validar el stock** (R-05). Validar y descontar en
  operaciones separadas permite que dos pedidos simultáneos vendan el mismo último pan.
- 🔴 **Copiar el precio, no referenciarlo** (R-04). Si el `SELECT` del detalle lee
  `productos.precio` en lugar de `detalle_pedido.precio_unitario`, se pierde el histórico y
  la corrección §C-01 queda anulada.
- 🔴 **Ajuste por diferencia al editar** (R-07), no devolver-y-redescontar: el kardex debe
  reflejar lo que realmente pasó.
- Excluir lotes vencidos del stock vendible (R-03).
- El `CHECK (cantidad_disponible >= 0)` es la última red; si salta, el bloqueo falló.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Pruebas unitarias de `registrarSalidaFefo`: un lote, varios lotes, stock exacto,
      stock insuficiente.**
- [ ] **Probado bajo concurrencia: dos peticiones simultáneas por el último ítem.**
