# HU-24 · Dashboard resumen — Tarea Frontend

Pantalla de inicio de la aplicación. Debe cargar rápido y priorizar lo accionable.

---

## 1. Pasos

1. `dashboard.model.ts` y `dashboard.service.ts`.
2. `dashboard.component` con cinco tarjetas, **ordenadas por urgencia**, no por número de
   criterio: primero lo que exige acción hoy.
3. Cada tarjeta muestra la cifra grande, el desglose y un enlace a la pantalla
   correspondiente.
4. **Filtrado por rol** (R-05) con la directiva `*siRol`.
5. Semáforo en stock bajo y vencimientos.
6. Botón de recarga manual y marca de hora de la última actualización.
7. Ruta por defecto tras iniciar sesión.
8. Los tres estados **por tarjeta**: una consulta lenta no debe bloquear las otras cuatro.
9. Antes de escribir cualquier gráfico, cargar la guía de visualización de datos del
   proyecto: los colores y las formas deben ser consistentes con HU-26.

---

## 2. Archivos

```
frontend/src/app/features/dashboard/
├── models/dashboard.model.ts
├── services/dashboard.service.ts
├── pages/dashboard/
└── components/{tarjeta-pedidos,tarjeta-compras,tarjeta-stock,
              tarjeta-produccion,tarjeta-vencimientos}/
```

---

## 3. Puntos de cuidado

- **Estados independientes por tarjeta**: si la consulta de vencimientos tarda, las otras
  cuatro ya deben estar visibles. Un dashboard que se bloquea entero por una consulta lenta
  es peor que uno incompleto.
- **Ordenar por urgencia, no por número de CA**: lo vencido y lo agotado arriba.
- Mostrar la hora de la última actualización: sin ella, nadie sabe si está viendo datos de
  ahora o de cuando abrió la pestaña por la mañana.
- Cada cifra debe ser un enlace: el dashboard sirve para detectar, y de ahí se va a actuar.
- Nada de auto-refresco agresivo: recarga manual y al volver a la pestaña.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
