# HU-14 · Detalle de compra — Tarea Frontend

Tabla de ítems dentro de la pantalla de detalle de compra (HU-13). Edición en línea y totales en vivo.

---

## 1. Pasos

1. `detalle-compra.model.ts` y `detalle-compra.service.ts`.
2. `detalle-compra-tabla.component` integrado en `compra-detalle`:
   materia prima, cantidad, precio unitario, subtotal, acciones.
3. Fila de totales al pie, con el valor **que devuelve el backend** (R-03).
4. `agregar-item-dialog.component`:
   - selector de materia prima **excluyendo las ya presentes** (R-02);
   - precio unitario **prellenado con el costo de referencia**, editable;
   - subtotal calculado en vivo como previsualización.
5. Mostrar la diferencia con el costo de referencia (R-04): ↑ rojo si es más caro,
   ↓ verde si es más barato.
6. Edición en línea de cantidad y precio con guardado inmediato.
7. Confirmación al quitar un ítem.
8. **Si `editable === false`, la tabla se muestra en modo lectura**: sin botón de agregar,
   sin acciones de fila (R-01).
9. Ruta heredada de HU-13.

---

## 2. Archivos

```
frontend/src/app/features/compra/
├── models/detalle-compra.model.ts
├── services/detalle-compra.service.ts
└── components/{detalle-compra-tabla,agregar-item-dialog}/
```

---

## 3. Puntos de cuidado

- 🔴 **El total mostrado debe ser el del backend** (R-03). La previsualización mientras se
  escribe está bien; la cifra final, no. Dos capas calculando con reglas de redondeo
  distintas producen descuadres que nadie sabe explicar.
- **Excluir del selector las materias primas ya presentes** (R-02): evita el `409` por
  completo y sugiere implícitamente editar la línea existente.
- Prellenar el precio con el costo de referencia ahorra tecleo y hace evidente la desviación
  cuando el proveedor cobra otra cosa.
- En modo no editable, **ocultar** los controles en lugar de deshabilitarlos.
- `step="0.01"` en cantidad y precio (R-07).

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
