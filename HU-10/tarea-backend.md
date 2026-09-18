# HU-10 · Consulta de inventario — Tarea Backend

Historia de solo lectura. Se apoya en la vista de V2 y expone el kardex creado en HU-09.

---

## 1. Pasos

1. `InventarioConsultaRowMapper` sobre `v_inventario_materia_prima`.
2. `InventarioRepository.consultar(idMateriaPrima, soloStockBajo, incluirInactivas, page)`:
   - `SELECT ... FROM v_inventario_materia_prima WHERE 1=1` + filtros opcionales;
   - `soloStockBajo=true` → `AND estado_stock IN ('BAJO','AGOTADO')`;
   - lista blanca de orden: `{nombre, cantidadDisponible, faltante, fechaActualizacion}`.
3. `InventarioRepository.contarConsulta(...)` — mismo `WHERE`.
4. `MovimientoInventarioRepository.buscarPorMateriaPrima(id, fechaInicio, fechaFin, page)`
   con `LEFT JOIN usuarios` para el nombre (**`getObject(..., Long.class)`**, el usuario
   puede ser `NULL`).
5. DTO: `InventarioResponse`, `MovimientoResponse`.
6. `InventarioService` con `@Transactional(readOnly = true)` en los tres métodos.
7. `InventarioController` con `@PreAuthorize` de lectura amplia.
8. **Sin endpoints de escritura** (R-01).
9. Endpoint de conciliación para QA (opcional, solo perfil `dev`): compara el saldo de
   `inventario` con el último `saldo_posterior` del kardex.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/inventario/
├── InventarioController.java        ← solo GET
├── InventarioService.java           (+ métodos de consulta)
├── InventarioRepository.java        (+ consultar, contarConsulta)
├── InventarioConsultaRowMapper.java
└── dto/{InventarioResponse,MovimientoResponse}.java
```

---

## 3. Puntos de cuidado

- **Sin endpoints de escritura aquí** (R-01). El inventario solo se mueve desde
  `InventarioService.registrarEntrada/registrarSalida`, invocados por HU-09, HU-13 y HU-23.
- Consultar **la vista**, no reconstruir los `JOIN` en cada endpoint (R-02).
- En el kardex, `id_usuario` puede ser `NULL`: usar `getObject(..., Long.class)` y mostrar
  "Sistema".
- El kardex crece rápido: paginar siempre y ordenar por `id_movimiento DESC`, que usa el
  índice `idx_mov_mp_materia`.
- No ocultar las materias primas inactivas con stock (R-04).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
