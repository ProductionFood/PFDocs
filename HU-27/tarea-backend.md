# HU-27 · Alerta de stock bajo — Tarea Backend

La historia más simple del proyecto: una consulta sobre la vista de V2. El valor está en el ordenamiento por urgencia y en la sugerencia de compra.

---

## 1. Pasos

1. `AlertaStockRowMapper` sobre `v_inventario_materia_prima`.
2. `AlertaRepository.listarAlertas(incluirInactivas)`:
   ```sql
   SELECT id_materia_prima, nombre, unidad, cantidad_disponible,
          stock_minimo, faltante, estado_stock, costo_unitario, fecha_actualizacion
   FROM v_inventario_materia_prima
   WHERE estado_stock IN ('BAJO','AGOTADO')
     AND (:incluirInactivas OR estado = 1)
   ORDER BY FIELD(estado_stock, 'AGOTADO', 'BAJO'), faltante DESC
   ```
   El `FIELD()` implementa R-03: agotadas primero, luego por faltante.
3. `AlertaRepository.resumen()` — conteos para el indicador global.
4. `AlertaRepository.sugerenciaCompra()` (R-06):
   - por cada materia prima en alerta, buscar el proveedor de su **última compra
     `RECIBIDA`** (`ORDER BY c.fecha DESC LIMIT 1`);
   - `cantidadSugerida = faltante × 1.2`, redondeada hacia arriba a 2 decimales;
   - agrupar por proveedor.
5. `costoReposicion = faltante × costo_unitario`, con `BigDecimal`.
6. `AlertaService` con `@Transactional(readOnly = true)`.
7. `AlertaController` — solo `GET`. El endpoint `/resumen` abierto a todos los roles
   autenticados (alimenta el indicador global de HU-23).
8. **Sin caché**: el conteo alimenta un indicador que debe reflejar la realidad.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/inventario/
├── {AlertaRepository,AlertaService,AlertaController}.java
├── AlertaStockRowMapper.java
└── dto/{AlertaStockResponse,ResumenAlertasResponse,SugerenciaCompraResponse}.java
```

---

## 3. Puntos de cuidado

- **`FIELD(estado_stock, 'AGOTADO', 'BAJO')`** en el `ORDER BY` implementa R-03 en una sola
  expresión. CA-04 pide ordenar por diferencia; ese orden se aplica **dentro** de cada nivel,
  porque una materia prima agotada siempre es más urgente.
- **`<`, no `<=`** (R-02): con disponible igual al mínimo no hay alerta. La vista ya lo
  resuelve, pero conviene no reintroducir la comparación en el servicio.
- **Excluir las inactivas por defecto** (R-05): alertar sobre algo que no se va a reponer es
  ruido permanente.
- El endpoint `/resumen` debe ser muy ligero: se consulta desde todas las pantallas.
- Si la alerta no muestra nada cuando debería, revisar primero que todas las materias primas
  tengan fila de inventario (HU-08 R-01).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
