# HU-11 · Gestión de productos — Tarea Backend

CRUD gemelo de HU-08. **No** crea fila de inventario: el stock de producto terminado es por lote (HU-19), no por producto.

---

## 1. Pasos

1. `Producto` (POJO) y `ProductoRowMapper` con `JOIN unidades_medida`.
2. `ProductoRepository`:
   - `buscar`, `contar`, `buscarPorId`, `insertar`, `actualizar`, `cambiarEstado`;
   - el `SELECT` del listado incluye `LEFT JOIN recetas` (para `tieneReceta`) y
     `LEFT JOIN v_stock_producto_terminado` (para `stockDisponible`);
   - lista blanca: `{nombre, precio, id}`.
3. DTO con `BigDecimal` en `precio`.
4. `ProductoService.crear()`: valida unidad existente → `UNIDAD_INEXISTENTE`.
   **No crea fila de inventario** (a diferencia de HU-08 R-01).
5. `ProductoService.cambiarEstado()`: al desactivar, consultar
   `v_stock_producto_terminado` y devolver el stock como advertencia (R-03).
6. Método `existeActivo(id)` para HU-12, HU-17, HU-19 y HU-21.
7. Controlador: lectura amplia, escritura solo ADMIN.
8. `@Auditable` en escrituras. Sin `DELETE` (R-05).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/producto/
├── {Producto,ProductoRowMapper,ProductoRepository,ProductoService,ProductoController}.java
└── dto/{CrearProductoRequest,ActualizarProductoRequest,ProductoResponse}.java
```

---

## 3. Puntos de cuidado

- **A diferencia de HU-08, aquí NO se crea fila de inventario.** El stock de producto
  terminado entra con los lotes de HU-19.
- `BigDecimal` en el precio, nunca `double`.
- `precio = 0` es válido (R-06): el `CHECK` es `>= 0`.
- **Cambiar el precio no debe tocar `detalle_pedido`** (R-01). Si alguien propone
  "actualizar los pedidos al nuevo precio", la respuesta está en R-01.
- La unidad del producto es la de **venta**, distinta de la de la receta (R-04).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
