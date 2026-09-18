# HU-21 · Productos en plan de producción — Tarea Backend

Detalle del plan con dos operaciones separadas: planificar y registrar producción (R-01). Calcula el consumo teórico estimado a partir de la receta de HU-12.

---

## 1. Pasos

1. `DetallePlanProduccion` (POJO) y `DetallePlanRowMapper` con `JOIN productos`,
   `unidades_medida` y `LEFT JOIN recetas` (para `tieneReceta`).
2. `DetallePlanRepository`: `listarPorPlan`, `insertar`, `actualizarPlanificada`,
   `actualizarProducida`, `eliminar`, `existeProducto(idPlan, idProducto)`,
   `tieneConsumos(idDetallePlan)`.
3. **Cálculo del consumo teórico** (`CalculadoraConsumoService`, reutilizable por HU-22):
   ```
   consumoTeorico = (cantidadPlanificada / receta.cantidadProducir) × detalleReceta.cantidad
   ```
   Con `BigDecimal`, escala 3 y `RoundingMode.HALF_UP` (`detalle_receta.cantidad` es
   `decimal(10,3)`, HU-12 R-06).
4. DTO: `AgregarDetallePlanRequest`, `ActualizarPlanificadaRequest`,
   `RegistrarProducidaRequest`, `DetallePlanResponse` (con `avance` y
   `consumoTeoricoEstimado`).
5. `DetallePlanService`:
   - `agregar()`: plan existe y **está `PLANIFICADO`**; producto existe; no duplicado
     → `PRODUCTO_DUPLICADO`;
   - `actualizarPlanificada()`: solo con plan `PLANIFICADO` → `PLAN_NO_EDITABLE` (R-02);
   - **`registrarProducida()`**: solo con plan `EN_PROCESO` → `PLAN_NO_EN_PROCESO` (R-03);
     **no valida un tope superior** (R-04);
   - `eliminar()`: si `tieneConsumos()` → `409 DETALLE_CON_CONSUMOS` (R-08).
6. `avance = (producida / planificada) × 100`, con `planificada > 0` garantizado por el
   `CHECK`.
7. `@Auditable` en las cuatro escrituras.
8. Controlador: escritura ADMIN y PRODUCCION.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/produccion/
├── {DetallePlanProduccion,DetallePlanRowMapper,DetallePlanRepository}.java
├── {DetallePlanService,DetallePlanController}.java
├── CalculadoraConsumoService.java          ← la reutiliza HU-22
└── dto/{AgregarDetallePlanRequest,ActualizarPlanificadaRequest,
        RegistrarProducidaRequest,DetallePlanResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **Endpoints separados para planificar y producir** (R-01). Un `PUT` que acepte ambos
  campos permite reescribir el objetivo después de conocer el resultado, y la eficiencia de
  HU-26 pierde todo significado.
- 🔴 **`CalculadoraConsumoService` se escribe una vez aquí y la usa HU-22.** Si el cálculo
  se duplica, las dos versiones divergen y el consumo teórico estimado no coincidirá con el
  que HU-22 registre.
- **No limitar la cantidad producida** (R-04): producir de más es un dato real.
- **Registrar producción no crea el lote** (R-07). Es el hueco conocido del flujo; conviene
  que la interfaz lo recuerde.
- `BigDecimal` con escala 3 en el consumo teórico (HU-12 R-06).
- Verificar consumos antes de eliminar una línea (R-08).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
