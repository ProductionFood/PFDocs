# HU-07 · Unidades de medida — Tarea Backend

El CRUD más simple del proyecto, con dos particularidades: no se pagina y no se elimina. Alimenta los desplegables de HU-08, HU-11 y HU-12.

---

## 1. Pasos

1. `UnidadMedida` (POJO) y `UnidadMedidaRowMapper`.
2. `UnidadMedidaRepository`:
   - `listarTodas()` → `ORDER BY nombre`, **sin `LIMIT`** (R-01);
   - `buscarPorId()`, `insertar()`, `actualizar()`;
   - `existePorAbreviatura(abrev)` y `existePorAbreviaturaExcluyendo(abrev, id)`;
   - `contarUsos(idUnidad)` → suma de referencias en `materias_primas`, `productos` y
     `recetas` (R-04).
3. DTO: `CrearUnidadRequest`, `ActualizarUnidadRequest`, `UnidadMedidaResponse`
   y `UnidadEnUsoResponse` (con el conteo por tabla).
4. `UnidadMedidaService`:
   - `crear()` valida abreviatura no duplicada → `ABREVIATURA_DUPLICADA`;
   - `actualizar()` valida excluyendo la propia; si cambia la abreviatura y
     `contarUsos() > 0`, exige confirmación explícita (parámetro `confirmar=true`) o
     devuelve `409 UNIDAD_EN_USO` (R-04);
   - captura `DuplicateKeyException` por si hay concurrencia.
5. `UnidadMedidaController`: `GET` para todos los roles autenticados, escritura solo ADMIN.
6. `@Auditable` en crear y editar.
7. **Sin `DELETE`** (R-03).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/unidadmedida/
├── {UnidadMedida,UnidadMedidaRowMapper,UnidadMedidaRepository,...}.java
└── dto/{CrearUnidadRequest,ActualizarUnidadRequest,UnidadMedidaResponse}.java
```

---

## 3. Puntos de cuidado

- **Devolver un array plano, no `PageResponse`** (R-01). Es la excepción documentada a la
  regla de paginación.
- **`GET` abierto a todos los roles autenticados**: PRODUCCION, COMPRAS y VENTAS necesitan
  el catálogo para sus formularios aunque no puedan editarlo.
- El `UNIQUE` con collation `_ai_ci` es *case-insensitive*: `kg` y `KG` colisionan (R-02).
  El mensaje de error debe ser claro al respecto.
- **R-04 no es un capricho**: cambiar la abreviatura de una unidad en uso reinterpreta
  todos los registros que la referencian sin tocar un solo dato de esas tablas.
- Sin `DELETE`. Si alguien lo pide, la respuesta está en R-03.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
