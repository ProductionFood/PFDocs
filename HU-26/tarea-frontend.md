# HU-26 · Eficiencia de producción — Tarea Frontend

Reporte analítico. La presentación importa tanto como el cálculo: un número mal etiquetado lleva a la decisión equivocada.

---

## 1. Pasos

1. `eficiencia.model.ts` y `eficiencia.service.ts`.
2. **Cargar la guía de visualización de datos del proyecto antes de escribir cualquier
   gráfico**, para mantener la paleta coherente con HU-24 y HU-25.
3. `eficiencia-reporte.component`:
   - filtro de rango de fechas (por defecto, mes en curso);
   - tarjetas con las tres métricas **claramente etiquetadas** (R-01).
4. **Explicar cada métrica con un tooltip**: "Cumplimiento: se produjo el 87% de lo
   planificado" / "Rendimiento ajustado: por cada unidad producida se consumió un 3% menos
   de lo previsto".
5. Tabla de planes con las tres columnas, expandible al detalle por producto y materia prima.
6. **Mostrar `—` donde la métrica sea `null`** (R-03), con tooltip explicativo. Nunca `0%`.
7. Indicar cuántos registros quedaron fuera del promedio (`registrosSinRendimiento`).
8. Gráfico de desviación por materia prima, ordenado por magnitud (R-06): es el hallazgo
   más accionable.
9. Sección aparte con los productos sin receta (R-07), con enlace a HU-12.
10. Ruta: ADMIN y PRODUCCION; el endpoint de materias primas también para COMPRAS.

---

## 2. Archivos

```
frontend/src/app/features/reporte/
├── models/eficiencia.model.ts
├── services/eficiencia.service.ts
├── pages/eficiencia-reporte/
└── components/{metricas-cards,desviacion-mp-chart,plan-eficiencia-tabla}/
```

---

## 3. Puntos de cuidado

- 🔴 **Etiquetar las métricas con lo que significan, no con su fórmula.** "Eficiencia:
  92,6%" no dice nada; "Se consumió un 8% más de materia prima de lo previsto" sí.
- 🔴 **`—` para los nulos** (R-03), nunca `0%`: un cero inventado por el frontend es peor
  que un hueco honesto.
- **Mostrar el conteo de registros excluidos** del promedio: sin él, el promedio parece
  cubrir todo cuando no es así.
- El gráfico de desviación por insumo ordenado por magnitud pone arriba lo que hay que
  revisar (R-06).
- Los productos sin receta visibles: son un hueco de datos, no un cero.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
