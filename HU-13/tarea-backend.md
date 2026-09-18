# HU-13 · Registro de compras — Tarea Backend

La operación más delicada de la fase: **recibir una compra suma stock**. Usa el `InventarioService` creado en HU-09. Requiere una máquina de estados explícita.

---

## 1. Pasos

1. `EstadoCompra` (enum): `PENDIENTE`, `RECIBIDA`, `CANCELADA`, con
   `transicionesValidas()` — un único lugar donde vive la máquina de estados (R-02).
2. `Compra` (POJO) y `CompraRowMapper` con `JOIN proveedores`; el `SELECT` del listado
   calcula `totalCompra` con una subconsulta
   `SUM(cantidad * precio_unitario)` sobre `detalle_compra`.
3. `CompraRepository`: `buscar`, `contar`, `buscarPorId`, `insertar`, `actualizar`,
   `cambiarEstado`, `contarDetalles(idCompra)`.
4. DTO: `CrearCompraRequest`, `ActualizarCompraRequest`, `CambiarEstadoCompraRequest`,
   `CompraResponse`, `RecepcionResponse`.
5. `CompraService.crear()`: valida proveedor existente y activo, valida fecha no futura.
6. `CompraService.actualizar()`: solo si `estado == PENDIENTE` → `COMPRA_NO_EDITABLE` (R-04).
7. 🔑 **`CompraService.cambiarEstado()` `@Transactional`** — el núcleo de la historia:
   - valida la transición con el enum → `TRANSICION_INVALIDA`;
   - si el nuevo estado es `RECIBIDA`:
     - verifica que haya detalle → `COMPRA_SIN_DETALLE` (R-03);
     - **por cada línea, `inventarioService.registrarEntrada(...)`** con tipo
       `ENTRADA_COMPRA` y origen `detalle_compra`/`idDetalle`;
     - recoge los saldos anterior y posterior para la respuesta;
   - actualiza el estado;
   - `@Auditable(accion="RECEPCION_COMPRA")`.
8. Exponer `esEditable(idCompra)` para que HU-14 lo consulte antes de tocar el detalle.
9. Controlador con `@PreAuthorize`: escritura y recepción solo ADMIN y COMPRAS.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/compra/
├── {Compra,CompraRowMapper,CompraRepository,CompraService,CompraController}.java
├── EstadoCompra.java                 ← máquina de estados en un solo lugar
└── dto/{CrearCompraRequest,ActualizarCompraRequest,CambiarEstadoCompraRequest,
        CompraResponse,RecepcionResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **Toda la recepción en UNA transacción** (R-01). Si la tercera línea falla, las dos
  primeras no deben haber entrado. Una recepción parcial produce un inventario que nadie
  puede explicar ni corregir con confianza.
- 🔴 **`RECIBIDA` es terminal** (R-02). Si alguien pide "poder deshacer la recepción", la
  respuesta está en R-02: se corrige con un ajuste de inventario, que queda registrado.
- **Las entradas pasan por `InventarioService`**, nunca con un `UPDATE inventario` directo.
  Cada línea genera su movimiento de kardex.
- La máquina de estados vive **en el enum**, no dispersa en `if` por el servicio: así una
  transición nueva se agrega en un solo sitio.
- Validar proveedor activo solo al crear (R-06).
- El `CHECK` de estado ya está en la base; validar antes da un mensaje mucho más útil.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Probado: recepción con fallo en una línea no deja ninguna entrada aplicada.**
- [ ] **Probado: `RECIBIDA → PENDIENTE` devuelve `409`.**
