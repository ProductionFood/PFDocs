# HU-12 · Gestión de recetas — Tarea Backend

Primera historia con **cabecera y detalle**. El patrón (subrecurso anidado, transacción que crea ambos, restricción de unicidad en el detalle) se repite en HU-14, HU-17 y HU-21.

---

## 1. Pasos

1. `Receta` y `DetalleReceta` (POJO), con sus `RowMapper`.
   El de detalle hace `JOIN materias_primas` y `unidades_medida`.
2. `RecetaRepository`:
   - `buscarPorId`, `buscarPorProducto`, `existePorProducto(idProducto)`;
   - `insertar`, `actualizar`, `cambiarEstado`;
   - `buscar(filtros, page)`, `contar(...)`.
3. `DetalleRecetaRepository`:
   - `listarPorReceta`, `insertar`, `actualizarCantidad`, `eliminar`;
   - `existeMateriaPrima(idReceta, idMateriaPrima)` → CA-05.
4. DTO: `CrearRecetaRequest` (con lista de detalles), `ActualizarRecetaRequest`,
   `AgregarDetalleRequest`, `RecetaResponse`, `DetalleRecetaResponse`.
5. `RecetaService.crear()` **`@Transactional`**:
   - valida producto existente y activo → `PRODUCTO_INEXISTENTE`;
   - valida que no tenga receta → `PRODUCTO_YA_TIENE_RECETA` (CA-01);
   - valida `cantidadProducir > 0` → `CANTIDAD_PRODUCIR_INVALIDA` (R-03);
   - **valida que no haya materias primas repetidas en la lista recibida** antes de
     insertar (el `UNIQUE` lo impediría, pero con un error menos claro);
   - inserta cabecera y detalles; si algo falla, se revierte todo.
6. `RecetaService.agregarDetalle()`: valida duplicado → `MATERIA_PRIMA_DUPLICADA`;
   captura `DuplicateKeyException` por concurrencia (R-01).
7. Cálculo de costo estimado con `BigDecimal` y `RoundingMode.HALF_UP`:
   `costoTotal = Σ (cantidad × costo_unitario)`, `costoPorUnidad = costoTotal / cantidadProducir`.
8. `RecetaController` con los nueve endpoints y `@PreAuthorize`.
9. **`DELETE` permitido solo sobre el detalle** (R-07), nunca sobre la receta.
10. `@Auditable` en todas las escrituras, incluido el borrado de detalle.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/receta/
├── {Receta,DetalleReceta}.java
├── {RecetaRowMapper,DetalleRecetaRowMapper}.java
├── {RecetaRepository,DetalleRecetaRepository}.java
├── {RecetaService,RecetaController}.java
└── dto/{CrearRecetaRequest,ActualizarRecetaRequest,AgregarDetalleRequest,
        RecetaResponse,DetalleRecetaResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **El `UNIQUE` de V2 es la garantía de CA-05** (R-01). Validar solo en el servicio deja
  pasar duplicados en peticiones simultáneas, y un ingrediente duplicado hace que HU-23
  descuente el doble de inventario.
- 🔴 **Documentar en el código qué significa `cantidad_producir`** (R-02): es el rendimiento
  de una tanda, no de una unidad. El cálculo de HU-22 depende de interpretarlo bien.
- Validar también los duplicados **dentro de la lista recibida** en `POST /recetas`: el
  `UNIQUE` los rechazaría, pero con un mensaje menos útil.
- `cantidad` del detalle es `decimal(10,3)`, no `(10,2)` (R-06). Usar `BigDecimal` con la
  escala correcta.
- `cantidadProducir > 0` estrictamente (R-03): es divisor.
- `DELETE` solo sobre el detalle (R-07).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
