# HU-10 · Consulta de inventario — Tarea Frontend

Pantalla de consulta con semáforo de stock y vista de kardex por materia prima.

---

## 1. Pasos

1. `inventario.model.ts` y `inventario.service.ts`.
2. `inventario-lista.component`: nombre, unidad, disponible, mínimo, faltante, estado,
   última actualización.
3. **Semáforo por `estadoStock`**: 🔴 `AGOTADO` · 🟠 `BAJO` · 🟢 `OK`.
4. Barra de progreso por fila: `disponible / stockMinimo`, en color según el estado.
5. Filtros: materia prima, solo stock bajo (`MatCheckbox`), incluir inactivas.
6. Tarjetas de resumen arriba: total de materias primas, cuántas bajo mínimo, cuántas
   agotadas.
7. `inventario-movimientos.component`: kardex de una materia prima, con tipo de movimiento,
   cantidad, saldos, documento de origen y usuario.
8. Icono de entrada/salida con color en cada movimiento.
9. Enlace desde cada fila del listado al kardex correspondiente.
10. Ruta: lectura para ADMIN/PRODUCCION/COMPRAS/CONSULTA. **Sin botones de escritura.**

---

## 2. Archivos

```
frontend/src/app/features/inventario/
├── models/inventario.model.ts
├── services/inventario.service.ts
├── pages/{inventario-lista,inventario-movimientos}/
└── inventario.routes.ts
```

---

## 3. Puntos de cuidado

- **Ningún control de escritura en esta pantalla** (R-01): ni ajustar, ni editar saldos.
- Mostrar el `faltante` en la misma fila evita la resta mental: es el dato que se necesita
  para hacer el pedido.
- El kardex debe mostrar **saldo anterior y posterior** de cada movimiento: es lo que
  permite localizar dónde se descuadró.
- Formatear `fechaActualizacion` con el locale `es-CO`; si es de hoy, mostrar solo la hora.
- Las materias primas inactivas con stock se muestran atenuadas, no ocultas (R-04).

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
