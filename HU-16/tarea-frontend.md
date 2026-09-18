# HU-16 · Creación de pedidos — Tarea Frontend

Listado de pedidos y formulario de cabecera. El flujo de estados debe advertir cuando la acción devuelve stock.

---

## 1. Pasos

1. `pedido.model.ts` con el enum `EstadoPedido`, y `pedido.service.ts`.
2. `pedido-lista.component`: número, cliente, fecha de pedido, fecha de entrega, ítems,
   total, estado, `⋮`.
3. Chip de estado: ámbar `PENDIENTE`, azul `EN_PREPARACION`, verde `ENTREGADO`,
   gris `CANCELADO`.
4. **Resaltar los pedidos con entrega para hoy o vencida**: es lo que se busca al abrir la
   pantalla por la mañana.
5. Filtros: cliente, estado, rango de fechas.
6. `pedido-formulario.component`: cliente (solo activos), fecha de entrega
   (`min` = hoy). **Sin campo de fecha de pedido ni de estado** (R-01).
7. `pedido-detalle.component`: cabecera + tabla de productos (HU-17) + acciones de estado.
8. **Diálogo de cancelación que advierte la devolución de stock** (R-04), mostrando qué
   vuelve a cada lote.
9. Ocultar controles de edición si `editable === false`.
10. Ruta: lectura ADMIN/VENTAS/PRODUCCION/CONSULTA; escritura ADMIN/VENTAS.

---

## 2. Archivos

```
frontend/src/app/features/pedido/
├── models/pedido.model.ts
├── services/pedido.service.ts
├── pages/{pedido-lista,pedido-formulario,pedido-detalle}/
├── components/cancelar-pedido-dialog/
└── pedido.routes.ts
```

---

## 3. Puntos de cuidado

- **El formulario no ofrece fecha de pedido ni estado** (R-01): los pone el servidor.
  Mostrarlos como editables invita a manipular algo que no se puede.
- **El diálogo de cancelación debe decir que el stock vuelve al inventario** (R-04): sin
  esa información, cancelar parece una operación menor y se hace sin pensar.
- `min` del datepicker de entrega = fecha del pedido (R-06).
- El total viene del backend.
- Resaltar las entregas de hoy convierte el listado en la pantalla de trabajo del día.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
