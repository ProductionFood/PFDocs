# HU-13 · Registro de compras — Tarea Frontend

Listado de compras y flujo de recepción. **La pantalla debe dejar claro que recibir una compra modifica el inventario de forma irreversible.**

---

## 1. Pasos

1. `compra.model.ts` con el enum `EstadoCompra`, y `compra.service.ts`.
2. `compra-lista.component`: número, proveedor, fecha, nº de ítems, total, estado, `⋮`.
3. Chip de estado con color: ámbar `PENDIENTE`, verde `RECIBIDA`, gris `CANCELADA`.
4. Filtros: proveedor, estado, rango de fechas.
5. `compra-formulario.component`: proveedor, fecha (por defecto hoy, `max` = hoy).
6. `compra-detalle.component`: cabecera + tabla de líneas (HU-14) + acciones de estado.
7. 🔑 **Diálogo de recepción** que muestra, **antes de confirmar**, el efecto exacto sobre
   el inventario línea por línea (ver `diseno.md`).
8. Tras recibir, mostrar los saldos resultantes.
9. Los controles de edición solo aparecen si `editable === true` (R-04).
10. Ruta: lectura ADMIN/COMPRAS/PRODUCCION/CONSULTA; escritura ADMIN/COMPRAS.

---

## 2. Archivos

```
frontend/src/app/features/compra/
├── models/compra.model.ts
├── services/compra.service.ts
├── pages/{compra-lista,compra-formulario,compra-detalle}/
├── components/recepcion-dialog/
└── compra.routes.ts
```

---

## 3. Puntos de cuidado

- 🔴 **El diálogo de recepción debe decir "esta acción no se puede deshacer"** y mostrar
  qué va a pasar con cada materia prima. Es una operación irreversible que altera el
  inventario: no puede dispararse con un clic distraído en un menú.
- Ocultar los botones de edición cuando `editable === false`: es más claro que mostrarlos y
  devolver `409`.
- `max` del datepicker = hoy (R-07).
- El total de la compra lo calcula el backend; el frontend solo lo muestra.
- El chip de estado debe ser legible de un vistazo: es lo que se busca al abrir el listado.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
