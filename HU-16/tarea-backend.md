# HU-16 · Creación de pedidos — Tarea Backend

Cabecera del pedido con máquina de estados. La cancelación devuelve stock, así que **el cambio de estado es transaccional** aunque parezca una simple actualización.

---

## 1. Pasos

1. `EstadoPedido` (enum) con `transicionesValidas()` (R-02), igual patrón que
   `EstadoCompra`.
2. `Pedido` (POJO) y `PedidoRowMapper` con `JOIN clientes`; el listado calcula
   `totalPedido` con subconsulta `SUM(cantidad * precio_unitario)` sobre `detalle_pedido`
   — posible gracias a la columna añadida en V2 §C-01.
3. `PedidoRepository`: `buscar`, `contar`, `buscarPorId`, `insertar`, `actualizar`,
   `cambiarEstado`.
4. **El `INSERT` omite `fecha_pedido`** para que actúe el `DEFAULT (curdate())` (R-01).
   No enviar `LocalDate.now()` desde Java: la fecha debe ser la del servidor de base de
   datos, una sola fuente de verdad.
5. DTO: `CrearPedidoRequest` (solo `idCliente` y `fechaEntrega`), `ActualizarPedidoRequest`,
   `CambiarEstadoPedidoRequest`, `PedidoResponse`.
6. `PedidoService.crear()`: valida cliente existente y **activo** → `CLIENTE_INACTIVO`;
   valida `fechaEntrega >= hoy` si viene informada.
7. 🔑 **`PedidoService.cambiarEstado()` `@Transactional`**:
   - valida la transición con el enum;
   - **si el nuevo estado es `CANCELADO`**, por cada línea del detalle llama a
     `InventarioPtService.registrarEntrada(idLote, cantidad, ENTRADA_DEVOLUCION, "pedidos", idPedido)`
     para devolver el stock al lote de origen (R-04);
   - actualiza el estado y audita.
8. `esEditable(idPedido)` para HU-17.
9. Controlador: escritura ADMIN y VENTAS.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/pedido/
├── {Pedido,PedidoRowMapper,PedidoRepository,PedidoService,PedidoController}.java
├── EstadoPedido.java
└── dto/{CrearPedidoRequest,ActualizarPedidoRequest,CambiarEstadoPedidoRequest,
        PedidoResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **No enviar la fecha desde Java** (R-01): omitir la columna en el `INSERT` y dejar que
  el `DEFAULT (curdate())` la ponga. Si la aplicación y la base están en zonas horarias
  distintas, tener dos fuentes de fecha produce inconsistencias que solo aparecen de noche.
- 🔴 **Cancelar devuelve stock** (R-04) y es transaccional. Es fácil tratarlo como un simple
  `UPDATE estado` y olvidar la devolución: el resultado es mercancía que desaparece del
  sistema pero sigue en el estante.
- `ENTREGADO` es terminal (R-02): una devolución posterior se registra en HU-18.
- Validar cliente activo solo al crear (R-07).
- La cancelación debe registrar el movimiento **sobre el lote original** de cada línea, no
  sobre un lote cualquiera del producto.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Probado: cancelar un pedido con líneas devuelve el stock a los lotes de origen.**
