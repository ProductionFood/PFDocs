# HU-25 · Consulta de pedidos — Tarea Frontend

Pantalla de consulta con filtros, tarjetas de resumen y dos rankings.

---

## 1. Pasos

1. Ampliar `pedido.service.ts` con `resumen()`, `ventasPorCliente()` y
   `productosMasVendidos()`.
2. `pedido-consulta.component`:
   - filtros: cliente, estado, rango de fechas;
   - por defecto: **mes en curso**, todos los estados.
3. Tarjetas: venta neta, venta comprometida, devoluciones, ticket promedio.
4. **Mostrar bruto, devoluciones y neto juntos** (R-04): el porcentaje de devoluciones es un
   indicador de calidad que se pierde si solo se muestra el neto.
5. Gráfico de barras: ventas por cliente (los 5 mayores).
6. Tabla de productos más vendidos, ordenada por unidades.
7. Tabla de pedidos con fila expandible que carga el detalle bajo demanda (R-06).
8. Cargar antes la guía de visualización de datos del proyecto, para que los gráficos sean
   coherentes con HU-24 y HU-26.
9. Ruta: consulta ADMIN/VENTAS/PRODUCCION/CONSULTA; agregaciones ADMIN/VENTAS.

---

## 2. Archivos

```
frontend/src/app/features/pedido/
├── services/pedido.service.ts        (+ agregaciones)
├── pages/pedido-consulta/
└── components/{resumen-ventas-cards,ventas-cliente-chart,productos-vendidos-tabla}/
```

---

## 3. Puntos de cuidado

- **Las tres cifras juntas** (bruto, devoluciones, neto): mostrar solo el neto oculta cuánto
  se está devolviendo, que es justamente lo que conviene vigilar.
- Cargar el detalle al expandir (R-06).
- El rango por defecto (mes en curso) evita traer todo el histórico al abrir.
- Los importes se formatean con locale `es-CO`; los cálculos vienen del backend.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
