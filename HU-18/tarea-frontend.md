# HU-18 · Registro de devoluciones — Tarea Frontend

Registro desde el detalle del pedido y bandeja de aprobación. El diálogo de aprobación debe dejar claro que repone stock.

---

## 1. Pasos

1. `devolucion.model.ts` y `devolucion.service.ts`.
2. Acción **"Registrar devolución"** en cada línea del detalle del pedido (HU-17),
   visible solo si el pedido es devolvible (R-05).
3. `devolucion-formulario.component`:
   - muestra cantidad pedida, ya devuelta y **disponible para devolver**;
   - `max` de cantidad = disponible;
   - motivo como `MatSelect` con opción "Otro" que habilita texto libre (R-07);
   - descripción opcional con contador de caracteres.
4. `devolucion-lista.component`: bandeja con pedido, cliente, producto, cantidad, motivo,
   fecha, estado.
5. Filtro por defecto: **`PENDIENTE`** — es la bandeja de trabajo.
6. 🔑 **Diálogo de aprobación** que advierte la reposición de stock y muestra a qué lote
   vuelve (R-01, R-02).
7. Diálogo de rechazo que solicita el motivo del rechazo.
8. Los tres estados.
9. Ruta: lectura amplia; registrar y aprobar, ADMIN y VENTAS.

---

## 2. Archivos

```
frontend/src/app/features/devolucion/
├── models/devolucion.model.ts
├── services/devolucion.service.ts
├── pages/{devolucion-lista,devolucion-formulario}/
├── components/{aprobar-devolucion-dialog,rechazar-devolucion-dialog}/
└── devolucion.routes.ts
```

---

## 3. Puntos de cuidado

- **Mostrar siempre los tres números**: pedido, ya devuelto y disponible. Sin el segundo,
  el `422` resulta incomprensible para quien registra.
- **El diálogo de aprobación debe decir que el stock vuelve al inventario** (R-01): es una
  operación que altera existencias, no un simple cambio de etiqueta.
- La bandeja filtrada por `PENDIENTE` convierte la pantalla en una lista de tareas.
- El `max` del cliente es orientativo: otra devolución pendiente puede haberse registrado
  mientras tanto. El `422` del servidor manda.
- Advertir si el lote de destino está vencido: la unidad no volverá como stock vendible
  (R-02).

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
