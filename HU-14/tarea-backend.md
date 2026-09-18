# HU-14 · Detalle de compra — Tarea Backend

Subrecurso de compras con cálculo de totales. Repite el patrón de detalle de HU-12, añadiendo la verificación de estado del documento padre.

---

## 1. Pasos

1. `DetalleCompra` (POJO) y `DetalleCompraRowMapper` con `JOIN materias_primas` y
   `unidades_medida`; incluye `costo_unitario` para el comparativo de R-04.
2. `DetalleCompraRepository`:
   - `listarPorCompra(idCompra)`, `insertar`, `actualizar`, `eliminar`;
   - `existeMateriaPrima(idCompra, idMateriaPrima)` → R-02;
   - `calcularTotal(idCompra)` → `SELECT COALESCE(SUM(cantidad * precio_unitario), 0)`.
3. DTO: `AgregarDetalleCompraRequest`, `ActualizarDetalleCompraRequest`,
   `DetalleCompraResponse`, `DetalleCompraListaResponse` (con el total).
4. `DetalleCompraService`, todos los métodos con verificación previa:
   - la compra existe → `404 COMPRA_INEXISTENTE`;
   - **la compra está `PENDIENTE`** → `409 COMPRA_NO_EDITABLE` (R-01);
   - la materia prima existe y está activa;
   - no está duplicada → `409 MATERIA_PRIMA_DUPLICADA`, capturando también
     `DuplicateKeyException`.
5. **Cálculo del total con `BigDecimal`** y `setScale(2, RoundingMode.HALF_UP)` (R-03).
   Nunca `double`.
6. Devolver el total recalculado en cada respuesta: evita que el cliente tenga que pedirlo
   aparte y garantiza que la cifra mostrada sea la del servidor.
7. `@Auditable` en agregar, editar y eliminar.
8. Controlador con `@PreAuthorize`: escritura solo ADMIN y COMPRAS.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/compra/
├── {DetalleCompra,DetalleCompraRowMapper,DetalleCompraRepository}.java
├── {DetalleCompraService,DetalleCompraController}.java
└── dto/{AgregarDetalleCompraRequest,ActualizarDetalleCompraRequest,
        DetalleCompraResponse,DetalleCompraListaResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **Verificar el estado de la compra en los cuatro endpoints** (R-01), incluido el
  `DELETE`. Es fácil olvidarlo justamente en el borrado.
- 🔴 **`BigDecimal` en todo el cálculo** (R-03). Un `double` en `cantidad * precio` produce
  totales con centavos fantasma que no cuadran con la factura del proveedor.
- El `UNIQUE` de V2 respalda R-02, pero validar antes da un `409` con mensaje útil que
  sugiere editar la línea existente.
- **No actualizar `materias_primas.costo_unitario`** al registrar la compra (R-04).
- `precio_unitario = 0` es válido; `cantidad = 0` no (R-05).
- El total se recalcula desde la base en cada respuesta, no se acumula en memoria.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
