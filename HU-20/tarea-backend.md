# HU-20 · Plan de producción — Tarea Backend

Cabecera con máquina de estados. Más simple que HU-13 y HU-16 porque **ninguna transición mueve inventario** (R-01): el movimiento vive en HU-22 y HU-23.

---

## 1. Pasos

1. `EstadoPlan` (enum) con `transicionesValidas()` (R-01), mismo patrón que
   `EstadoCompra` y `EstadoPedido`.
2. `PlanProduccion` (POJO) y `PlanProduccionRowMapper` con `JOIN usuarios` para el nombre
   del responsable; el listado cuenta las líneas con subconsulta sobre
   `detalle_plan_produccion`.
3. `PlanProduccionRepository`: `buscar`, `contar`, `buscarPorId`, `insertar`, `actualizar`,
   `cambiarEstado`, `contarDetalles(idPlan)`.
4. DTO: `CrearPlanRequest` (sin `estado`), `ActualizarPlanRequest`,
   `CambiarEstadoPlanRequest`, `PlanResponse`.
5. `PlanProduccionService.crear()`:
   - valida `fechaProduccion >= fechaPlanificacion` → `FECHAS_INVALIDAS`;
   - **asigna el usuario autenticado** si no viene `idUsuario`, y solo permite asignar otro
     si el solicitante es `ADMIN` (R-04);
   - estado inicial `PLANIFICADO`.
6. `PlanProduccionService.actualizar()`: solo si está `PLANIFICADO` → `PLAN_NO_EDITABLE`.
7. `PlanProduccionService.cambiarEstado()` `@Transactional`:
   - valida la transición;
   - **al pasar a `EN_PROCESO`, exige al menos una línea** → `PLAN_SIN_PRODUCTOS` (R-02);
   - **no toca inventario** (R-06).
8. `esEditable(idPlan)` para que HU-21 lo consulte.
9. Controlador: escritura ADMIN y PRODUCCION.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/produccion/
├── {PlanProduccion,PlanProduccionRowMapper,PlanProduccionRepository}.java
├── {PlanProduccionService,PlanProduccionController}.java
├── EstadoPlan.java
└── dto/{CrearPlanRequest,ActualizarPlanRequest,CambiarEstadoPlanRequest,PlanResponse}.java
```

---

## 3. Puntos de cuidado

- **No confiar en el `idUsuario` que envía el cliente** (R-04): un usuario de PRODUCCION
  podría asignar su plan a otra persona y distorsionar la trazabilidad. Se toma del token
  salvo que quien llama sea `ADMIN`.
- **Cancelar no revierte consumos** (R-06). Si alguien pide "deshacer todo al cancelar", la
  respuesta es que la harina ya se usó: se corrige con un ajuste de inventario, no borrando
  el consumo.
- La validación de R-02 va en el cambio de estado, no en la creación: un plan recién creado
  no tiene líneas todavía, y eso es normal.
- Ninguna transición de esta historia llama a `InventarioService`.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
