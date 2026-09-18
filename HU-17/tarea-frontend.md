# HU-17 · Agregar productos al pedido — Tarea Frontend

Tabla de productos dentro del pedido, con stock visible en tiempo real y los lotes que se van a consumir.

---

## 1. Pasos

1. `detalle-pedido.model.ts` y `detalle-pedido.service.ts`.
2. `detalle-pedido-tabla.component`: producto, cantidad, precio unitario, subtotal,
   lotes consumidos, acciones.
3. `agregar-producto-dialog.component`:
   - selector de producto **excluyendo los ya presentes** (R-06) y **los que no tienen
     stock**;
   - al seleccionar, consultar `GET /productos/{id}/stock` y mostrar el disponible;
   - **previsualización de los lotes que se consumirán** según FEFO, actualizada al cambiar
     la cantidad;
   - bloquear el botón si la cantidad supera el disponible, con mensaje claro.
4. Mostrar los lotes consumidos en cada fila del detalle (trazabilidad, R-10).
5. Advertir cuando se consuma un lote **próximo a vencer**: "Se tomarán 12 unidades del lote
   PF-2026-0915, que vence en 1 día".
6. Edición de cantidad en línea, indicando el ajuste: "Se descontarán 5 unidades más".
7. Confirmación al quitar, avisando que el stock vuelve al inventario (R-08).
8. Modo lectura si el pedido no es editable (R-09).
9. Total del pedido siempre desde el backend.

---

## 2. Archivos

```
frontend/src/app/features/pedido/
├── models/detalle-pedido.model.ts
├── services/detalle-pedido.service.ts
└── components/{detalle-pedido-tabla,agregar-producto-dialog,lotes-consumidos-chip}/
```

---

## 3. Puntos de cuidado

- **Consultar el stock al seleccionar el producto**, no al abrir el diálogo: el dato debe
  ser lo más fresco posible. Aun así, **el stock puede haber cambiado** entre la consulta y
  el envío: el `409 STOCK_INSUFICIENTE` del servidor es la autoridad y hay que manejarlo con
  un mensaje útil, no genérico.
- **La previsualización de lotes FEFO** enseña al usuario qué mercancía va a salir y avisa
  de vencimientos próximos.
- Excluir del selector los productos sin stock evita el error antes de cometerlo.
- Al editar la cantidad, indicar el **delta** (R-07), no solo el valor final.
- El subtotal y el total vienen del backend.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
