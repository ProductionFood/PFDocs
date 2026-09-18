# HU-21 · Productos en plan de producción — Tarea Frontend

Tabla de productos del plan con avance, dentro de la pantalla de detalle de HU-20. Los controles cambian según el estado del plan.

---

## 1. Pasos

1. `detalle-plan.model.ts` y su servicio.
2. `detalle-plan-tabla.component`: producto, planificado, producido, avance, receta, `⋮`.
3. **Barra de avance por línea**, con color: gris < 100%, verde = 100%, azul > 100% (R-04).
4. `agregar-producto-plan-dialog.component`:
   - selector **excluyendo los ya presentes** (R-05);
   - al elegir producto y cantidad, mostrar el **consumo teórico estimado**;
   - **contrastar el consumo teórico con el inventario disponible** y advertir si no alcanza;
   - ⚠️ avisar si el producto **no tiene receta** (R-06).
5. **Registrar producción en línea**, habilitado solo con el plan `EN_PROCESO` (R-03).
6. Los controles de edición de lo planificado solo con el plan `PLANIFICADO` (R-02).
7. Recordatorio tras registrar producción: "Registre el lote de producto terminado para que
   esté disponible para la venta" con enlace a HU-19 (R-07).
8. Confirmación al quitar; manejar `409 DETALLE_CON_CONSUMOS` con mensaje claro.
9. Ruta heredada de HU-20.

---

## 2. Archivos

```
frontend/src/app/features/produccion/
├── models/detalle-plan.model.ts
├── services/detalle-plan.service.ts
└── components/{detalle-plan-tabla,agregar-producto-plan-dialog,registrar-producido-inline}/
```

---

## 3. Puntos de cuidado

- 🔴 **El recordatorio del lote (R-07) no es decorativo.** Sin él, se completa un plan y el
  producto fabricado nunca aparece como stock vendible, sin que nada lo señale. Es el hueco
  más fácil de sufrir de todo el flujo.
- **Contrastar consumo teórico con inventario disponible** en el diálogo evita descubrir a
  mitad de tanda que falta harina.
- Advertir sobre los productos sin receta (R-06): quedarán fuera del análisis de HU-26.
- Mostrar avance > 100% en color distinto, no como error (R-04).
- Los controles deben responder al estado del plan: planificar con `PLANIFICADO`, producir
  con `EN_PROCESO`.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
