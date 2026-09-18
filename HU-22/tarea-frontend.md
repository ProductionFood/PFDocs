# HU-22 · Registro de consumo de materia prima — Tarea Frontend

Tabla de consumos dentro del plan, con teórico prellenado y comparativa visible. La pantalla debe advertir que registrar consumo **descuenta inventario**.

---

## 1. Pasos

1. `consumo.model.ts` y `consumo.service.ts`.
2. `consumo-tabla.component` dentro del detalle del plan (HU-20):
   materia prima, teórico, real, desviación, eficiencia, fecha.
3. **Prellenar el campo real con el teórico**: en el caso normal se consume lo previsto, y
   así solo se corrige lo que difiere.
4. Mostrar **desviación y eficiencia** juntas (R-04), con la desviación destacada por ser la
   legible: `+8,0%` en rojo, `−5,0%` en azul, `0%` en verde.
5. **Mostrar `—` cuando la eficiencia sea `null`** (R-03), nunca `NaN` ni `Infinity`, con
   tooltip: "No se puede calcular: no se registró consumo".
6. `registrar-consumo-dialog.component`:
   - lista los ingredientes de la receta con su teórico y el disponible en inventario;
   - permite registrar **todos de una vez** o uno a uno;
   - **advierte el impacto en el inventario** antes de confirmar;
   - permite agregar un ingrediente fuera de receta, marcándolo (R-08).
7. Marcar visualmente los consumos **fuera de receta**.
8. Al corregir, indicar el ajuste: "Se descontarán 3,000 kg adicionales" (R-07).
9. Solo habilitado con el plan `EN_PROCESO`.

---

## 2. Archivos

```
frontend/src/app/features/produccion/
├── models/consumo.model.ts
├── services/consumo.service.ts
└── components/{consumo-tabla,registrar-consumo-dialog,indicador-desviacion}/
```

---

## 3. Puntos de cuidado

- 🔴 **`eficiencia: null` se muestra como `—`** (R-03). Un `NaN` o un `Infinity` en pantalla
  es un defecto visible de inmediato; peor es un `0%` inventado por el cliente para
  disimularlo.
- 🔴 **La desviación es la cifra que la gente entiende** (R-04). Una eficiencia del 200% se
  malinterpreta como algo bueno; un "−50% de consumo" se lee solo.
- **Prellenar con el teórico** reduce el trabajo y hace que la diferencia salte a la vista.
- **Advertir el impacto en inventario** antes de confirmar: es una operación que descuenta
  stock (R-05), no un simple apunte.
- Mostrar el disponible junto a cada teórico evita intentar consumir lo que no hay.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
