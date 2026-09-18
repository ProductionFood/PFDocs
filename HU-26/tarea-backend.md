# HU-26 · Eficiencia de producción — Tarea Backend

Solo lectura, pero es el reporte con más trampas de cálculo del proyecto. Reutiliza `CalculadoraEficienciaService` de HU-22.

---

## 1. Pasos

1. `EficienciaRepository`:
   - `planesConConsumo(fechaInicio, fechaFin)` — solo planes con al menos un consumo (R-05);
   - `detallePlan(idPlan)` — productos con planificado y producido;
   - `consumosPlan(idPlan)` — teórica y real por materia prima;
   - `desviacionPorMateriaPrima(fechaInicio, fechaFin)` — agregado del período (R-06);
   - `productosSinReceta(fechaInicio, fechaFin)` (R-07).
2. **Todos los cálculos con `CASE WHEN divisor = 0 THEN NULL`** (R-03).
3. `EficienciaService` con las tres métricas de R-01 y R-02:
   ```java
   cumplimiento        = producida / planificada * 100;
   rendimiento         = real == 0 ? null : teorica / real * 100;
   rendimientoAjustado = real == 0 ? null
                       : (teorica * producida / planificada) / real * 100;   // R-02
   desviacion          = teorica == 0 ? null : (real - teorica) / teorica * 100;
   ```
   Con `BigDecimal`, escala 4 en los intermedios y 2 en el resultado, `HALF_UP`.
4. 🔴 **Al promediar, excluir los `null`** y contar cuántos se excluyeron
   (`registrosSinRendimiento`, R-03). **Nunca tratarlos como cero.**
5. Separar los planes cancelados del promedio general (R-05).
6. Validar `fechaInicio <= fechaFin`.
7. `@Transactional(readOnly = true)`. Sin endpoints de escritura.
8. Verificar con `EXPLAIN` el uso de `idx_planes_fecha`.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/reporte/
├── {EficienciaRepository,EficienciaService,EficienciaController}.java
└── dto/{EficienciaResponse,EficienciaPlanResponse,ResumenEficienciaResponse,
        DesviacionMateriaPrimaResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **`null` no es cero al promediar** (R-03). `AVG()` de SQL ya ignora los `NULL`, pero un
  promedio calculado en Java con un `orElse(0)` los incluye como ceros y hunde el indicador.
  Es el error más probable de esta historia.
- 🔴 **Implementar `rendimientoAjustado`** (R-02). Sin él, el reporte lleva a la conclusión
  contraria: el plan que produjo la mitad aparece como el más eficiente.
- 🔴 **No fusionar cumplimiento y rendimiento** en un solo número (R-01).
- **Reutilizar `CalculadoraEficienciaService` de HU-22**, no reescribir las fórmulas: si
  divergen, el dato del registro y el del reporte no coincidirán.
- Los planes cancelados se informan aparte (R-05).
- Los productos sin receta se listan, no se omiten (R-07).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Pruebas unitarias de las cuatro fórmulas con divisor cero.**
- [ ] **Prueba: un consumo con `real = 0` no baja el promedio general.**
