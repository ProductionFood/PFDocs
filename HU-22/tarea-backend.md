# HU-22 · Registro de consumo de materia prima — Tarea Backend

**Se implementa junto con HU-23**: registrar el consumo descuenta inventario en la misma transacción. Reutiliza `CalculadoraConsumoService` de HU-21 e `InventarioService` de HU-09.

---

## 1. Pasos

1. `ConsumoMateriaPrima` (POJO) y `ConsumoRowMapper` con `JOIN materias_primas` y
   `unidades_medida`.
2. `ConsumoRepository`: `listarPorPlan`, `listarPorDetallePlan`, `insertar`,
   `actualizarReal`, `buscarPorId`, `existeConsumo(idDetallePlan, idMateriaPrima)`.
3. **Reutilizar `CalculadoraConsumoService`** de HU-21 para `cantidad_teorica` (R-01).
   No duplicar la fórmula: si las dos versiones divergen, el teórico estimado que se muestra
   al planificar no coincidirá con el que se registra.
4. `CalculadoraEficienciaService`:
   ```java
   // R-03: el divisor puede ser cero
   public BigDecimal eficiencia(BigDecimal teorica, BigDecimal real) {
       if (real == null || real.compareTo(BigDecimal.ZERO) == 0) return null;
       return teorica.divide(real, 4, RoundingMode.HALF_UP)
                     .multiply(BigDecimal.valueOf(100))
                     .setScale(2, RoundingMode.HALF_UP);
   }
   // R-04: la métrica interpretable
   public BigDecimal desviacion(BigDecimal teorica, BigDecimal real) {
       if (teorica == null || teorica.compareTo(BigDecimal.ZERO) == 0) return null;
       return real.subtract(teorica).divide(teorica, 4, RoundingMode.HALF_UP)
                  .multiply(BigDecimal.valueOf(100)).setScale(2, RoundingMode.HALF_UP);
   }
   ```
5. DTO: `RegistrarConsumoRequest` (**sin `cantidadTeorica`**), `ActualizarConsumoRequest`,
   `ConsumoResponse`, `ConsumoTeoricoResponse`.
6. 🔑 **`ConsumoService.registrar()` `@Transactional`** — junto con HU-23:
   - el detalle de plan existe y su plan está `EN_PROCESO`;
   - la materia prima existe;
   - no hay consumo previo de ese ingrediente → `CONSUMO_DUPLICADO` (R-06);
   - **calcula `cantidad_teorica`** con la receta (R-01); si el ingrediente no está en la
     receta, `0.000` y se marca `fueraDeReceta` (R-08);
   - **`inventarioService.registrarSalida(...)`** con `SALIDA_PRODUCCION`, redondeando a
     2 decimales con `HALF_UP` (R-09);
   - `INSERT` en `consumo_materia_prima`.
7. `ConsumoService.actualizar()` con **ajuste por diferencia** (R-07).
8. `@Auditable(accion="REGISTRAR_CONSUMO")`.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/produccion/
├── {ConsumoMateriaPrima,ConsumoRowMapper,ConsumoRepository}.java
├── {ConsumoService,ConsumoController}.java
├── CalculadoraEficienciaService.java       ← la reutiliza HU-26
└── dto/{RegistrarConsumoRequest,ActualizarConsumoRequest,
        ConsumoResponse,ConsumoTeoricoResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **`cantidadTeorica` nunca llega del cliente** (R-01). Un DTO que la acepte permite
  falsear la eficiencia con solo editar el JSON.
- 🔴 **Proteger la división por cero** (R-03). MySQL devuelve `NULL` sin avisar: el reporte
  de HU-26 se llena de huecos y nadie sabe por qué.
- 🔴 **Redondeo uniforme a 2 decimales al descontar** (R-09): el consumo tiene 3 decimales y
  el inventario 2. Un criterio inconsistente acumula gramos y descuadra el inventario sin
  causa aparente.
- **Ajuste por diferencia al corregir** (R-07), no revertir y volver a descontar.
- **Reutilizar la calculadora de HU-21**, no reescribir la fórmula.
- El `UNIQUE` de V2 evita el doble descuento (R-06).
- Exponer `desviacion` además de `eficiencia` (R-04): es la cifra que se puede leer.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Pruebas unitarias de la eficiencia: real = 0, teórica = 0, ambos = 0, valores normales.**
- [ ] **Verificado: el redondeo de 3 a 2 decimales no descuadra el inventario.**
