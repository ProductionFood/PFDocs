# HU-18 · Registro de devoluciones — Tarea Backend

Aprobar una devolución repone stock, así que el cambio de estado es transaccional. Reutiliza `InventarioPtService` de HU-17 y consulta el kardex para saber a qué lote devolver.

---

## 1. Pasos

1. `EstadoDevolucion` (enum) con `transicionesValidas()` (R-04).
2. `Devolucion` (POJO) y `DevolucionRowMapper` con `JOIN detalle_pedido`, `pedidos`,
   `productos` y `clientes`.
3. `DevolucionRepository`:
   - `insertar`, `buscarPorId`, `cambiarEstado`;
   - `buscar(filtros, page)`, `contar(...)`;
   - **`sumarDevueltoPorDetalle(idDetallePedido)`** → suma de `APROBADA` + `PENDIENTE`,
     excluyendo `RECHAZADA` (R-03).
4. **El `INSERT` omite `fecha_devolucion`** para que actúe el `DEFAULT (curdate())` (R-06).
5. DTO: `CrearDevolucionRequest`, `CambiarEstadoDevolucionRequest`, `DevolucionResponse`
   (con `disponibleParaDevolver`), `AprobacionResponse` (con `lotesRepuestos`).
6. `DevolucionService.crear()`:
   - el detalle de pedido existe → `DETALLE_PEDIDO_INEXISTENTE`;
   - el pedido está `ENTREGADO` o `EN_PREPARACION` → `PEDIDO_NO_DEVOLVIBLE` (R-05);
   - `sumarDevueltoPorDetalle() + nueva <= cantidad` → `422 CANTIDAD_EXCEDE_PEDIDO` (R-03);
   - estado inicial `PENDIENTE`.
7. 🔑 **`DevolucionService.cambiarEstado()` `@Transactional`**:
   - valida la transición;
   - **si es `APROBADA`**:
     - consulta `movimientos_inventario_pt` con
       `tabla_origen = 'detalle_pedido' AND id_origen = idDetallePedido`
       y tipo `SALIDA_VENTA`, **ordenado por `id_movimiento DESC`** (orden inverso al
       consumo, R-02);
     - reparte la cantidad devuelta entre esos lotes en ese orden;
     - `InventarioPtService.registrarEntrada(idLote, cant, ENTRADA_DEVOLUCION, "devoluciones", idDevolucion)`;
   - actualiza el estado;
   - `@Auditable(accion="APROBAR_DEVOLUCION")`.
8. Controlador con `@PreAuthorize`: registrar y aprobar, ADMIN y VENTAS.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/devolucion/
├── {Devolucion,DevolucionRowMapper,DevolucionRepository}.java
├── {DevolucionService,DevolucionController}.java
├── EstadoDevolucion.java
└── dto/{CrearDevolucionRequest,CambiarEstadoDevolucionRequest,
        DevolucionResponse,AprobacionResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **El stock vuelve al aprobar, nunca al registrar** (R-01). Reponer al registrar
  devolvería al inventario producto en mal estado, listo para venderse otra vez.
- 🔴 **Contar las `PENDIENTE` en el acumulado** (R-03). Si solo se suman las aprobadas, tres
  devoluciones pendientes de 10 sobre una línea de 10 repondrían 30 al aprobarse.
- 🔴 **Consultar el kardex para saber a qué lote devolver** (R-02). Elegir un lote arbitrario
  del producto rompe la trazabilidad y puede reponer sobre un lote que ni siquiera participó
  en esa venta.
- Devolver en **orden inverso al consumo**: el último lote tomado es el de vencimiento más
  lejano.
- Omitir `fecha_devolucion` en el `INSERT` (R-06).
- Ambos estados finales son terminales (R-04).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Probado: aprobar repone exactamente en los lotes de los que salió la venta.**
- [ ] **Probado: el acumulado incluye las devoluciones pendientes.**
