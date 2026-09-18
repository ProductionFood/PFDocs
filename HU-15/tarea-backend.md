# HU-15 · Historial de compras — Tarea Backend

Solo lectura. Amplía el listado de HU-13 con filtros combinables y agrega dos endpoints de agregación.

---

## 1. Pasos

1. Ampliar `CompraRepository.buscar()` con `idProveedor`, `fechaInicio`, `fechaFin`,
   `estado` — todos opcionales, combinados con `AND` (R-02).
2. `CompraRepository.resumenPorPeriodo(fechaInicio, fechaFin)`:
   agrupa por estado con `SUM(cantidad * precio_unitario)`, excluyendo `CANCELADA` del
   gasto (R-04).
3. `CompraRepository.gastoPorProveedor(fechaInicio, fechaFin)`:
   `GROUP BY proveedor` sobre compras `RECIBIDA`, ordenado por total descendente.
4. Carga del detalle con `incluirDetalle=true`:
   - **una sola consulta** con `JOIN` a `detalle_compra` y agrupación en memoria por
     `id_compra`. Nunca una consulta por compra (R-06);
   - forzar `size <= 20` cuando el parámetro esté activo.
5. Validar `fechaInicio <= fechaFin` → `400 RANGO_FECHAS_INVALIDO` (R-03).
6. DTO: `ResumenComprasResponse`, `GastoProveedorResponse`.
7. `@Transactional(readOnly = true)` en todos los métodos.
8. Sin endpoints de escritura (R-01).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/compra/
├── CompraRepository.java       (+ buscar ampliado, resumenPorPeriodo, gastoPorProveedor)
├── CompraService.java          (+ métodos de consulta)
├── CompraController.java       (+ /resumen, /por-proveedor)
└── dto/{ResumenComprasResponse,GastoProveedorResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **Evitar el N+1 en `incluirDetalle`** (R-06): un `JOIN` con agrupación, no un bucle de
  consultas. Con 200 compras, la versión ingenua hace 201 viajes a la base y supera de largo
  los 2 s del RNF-02.
- **Las canceladas fuera del gasto** (R-04), y las pendientes en una cifra aparte.
- `BETWEEN` funciona correctamente porque `fecha` es `DATE` (R-03). Si algún día pasara a
  `DATETIME`, habría que usar `< fechaFin + 1 día`.
- El total se recalcula siempre desde el detalle (R-05); no cachear ni denormalizar.
- Verificar con `EXPLAIN` que el filtro por fechas usa `idx_compras_fecha`.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
