# HU-08 · Gestión de materias primas — Tarea Backend

CRUD siguiendo el patrón de HU-05, con dos añadidos importantes: validación de FK contra `unidades_medida` y **creación de la fila de inventario** en la misma transacción (R-01).

---

## 1. Pasos

1. `MateriaPrima` (POJO) y `MateriaPrimaRowMapper` con `JOIN unidades_medida` para
   devolver nombre y abreviatura.
2. `MateriaPrimaRepository`: `buscar`, `contar`, `buscarPorId`, `insertar`, `actualizar`,
   `cambiarEstado`. Lista blanca: `{nombre, stockMinimo, costoUnitario, id}`.
3. **`InventarioRepository.crearFilaInicial(idMateriaPrima)`** — `INSERT` con
   `cantidad_disponible = 0`.
4. DTO con `BigDecimal` en `stockMinimo` y `costoUnitario` (R-02), nunca `double`.
5. `MateriaPrimaService.crear()` **`@Transactional`**:
   - valida que la unidad exista → `UNIDAD_INEXISTENTE`;
   - inserta la materia prima;
   - **inserta la fila de inventario en cero** (R-01);
   - si algo falla, se revierte todo.
6. `MateriaPrimaService.cambiarEstado()`: al desactivar, consultar si está en recetas
   activas y devolver el conteo como advertencia (R-04).
7. Controlador con `@PreAuthorize`: lectura amplia, escritura solo ADMIN.
8. `@Auditable` en las escrituras.
9. Método `existeActiva(id)` para que lo usen HU-12, HU-14 y HU-22.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── materiaprima/
│   ├── {MateriaPrima,MateriaPrimaRowMapper,MateriaPrimaRepository,...}.java
│   └── dto/{CrearMateriaPrimaRequest,ActualizarMateriaPrimaRequest,MateriaPrimaResponse}.java
└── inventario/
    └── InventarioRepository.java     ← crearFilaInicial()
```

---

## 3. Puntos de cuidado

- 🔴 **R-01 es el punto crítico.** Sin la fila de inventario, HU-10, HU-23 y HU-27 fallan
  o muestran datos incompletos, y el síntoma aparece tres sprints después, lejos de la causa.
  Debe ir **en la misma transacción** que el alta.
- **`BigDecimal`, no `double`** (R-02). Revisar los DTO y el `RowMapper`:
  `rs.getBigDecimal()`, no `rs.getDouble()`.
- Validar la FK **en el servicio** para devolver `409 UNIDAD_INEXISTENTE` legible, no
  esperar al error de integridad de la base.
- `stock_minimo = 0` es válido (R-06): no rechazarlo.
- Sin `DELETE` (R-05).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Verificado: crear una materia prima deja su fila en `inventario` con cantidad 0.**
