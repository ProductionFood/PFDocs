# HU-20 · Plan de producción — Tarea Frontend

Listado y formulario de cabecera, más la pantalla de detalle que aloja HU-21 y HU-22.

---

## 1. Pasos

1. `plan-produccion.model.ts` con el enum `EstadoPlan`, y su servicio.
2. `plan-lista.component`: número, fecha de producción, responsable, nº de productos,
   estado, `⋮`.
3. Chip de estado: gris `PLANIFICADO`, azul `EN_PROCESO`, verde `COMPLETADO`,
   rojo tenue `CANCELADO`.
4. **Resaltar los planes cuya producción es hoy**: es la pantalla de trabajo de la jornada.
5. Filtros: estado, responsable, rango de fechas de producción.
6. `plan-formulario.component`: fechas (`min` de producción atado a planificación),
   observaciones con contador. **Sin campo de estado** (R-01).
7. `plan-detalle.component`: cabecera + productos (HU-21) + consumos (HU-22) + acciones de
   estado.
8. Botón "Iniciar producción" **deshabilitado si el plan no tiene productos** (R-02), con
   explicación en tooltip.
9. Diálogo de cancelación que advierte: "Los consumos ya registrados no se revertirán."
10. Ruta: lectura ADMIN/PRODUCCION/CONSULTA; escritura ADMIN/PRODUCCION.

---

## 2. Archivos

```
frontend/src/app/features/produccion/
├── models/plan-produccion.model.ts
├── services/plan-produccion.service.ts
├── pages/{plan-lista,plan-formulario,plan-detalle}/
└── produccion.routes.ts
```

---

## 3. Puntos de cuidado

- **El formulario no ofrece el estado** (R-01): nace `PLANIFICADO` y cambia por el flujo.
- Deshabilitar "Iniciar producción" con tooltip explicativo es mejor que permitir el clic y
  devolver `409` (R-02).
- El diálogo de cancelación debe ser honesto sobre R-06: la materia prima consumida no
  vuelve. Sin esa advertencia, cancelar parece inocuo.
- Resaltar la producción del día conecta con el dashboard de HU-24.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
