# HU-15 · Historial de compras — Tarea Frontend

Pantalla de consulta con filtros, tarjetas de resumen y desglose por proveedor.

---

## 1. Pasos

1. Ampliar `compra.service.ts` con `resumen()` y `gastoPorProveedor()`.
2. `compra-historial.component`:
   - filtros: proveedor, estado, rango de fechas (`MatDateRangePicker`);
   - por defecto: **mes actual**, todos los estados.
3. Tarjetas de resumen: gasto ejecutado, gasto comprometido, nº de compras.
4. Gráfico de barras horizontal con el gasto por proveedor (los 5 mayores).
5. Tabla de compras con fila expandible que carga el detalle **bajo demanda** (R-06):
   al desplegar una fila, se pide el detalle de esa compra.
6. Validar en el cliente `fechaInicio <= fechaFin` antes de llamar.
7. Los tres estados.
8. Ruta: ADMIN/COMPRAS/PRODUCCION/CONSULTA. Los endpoints de agregación, solo
   ADMIN/COMPRAS.

---

## 2. Archivos

```
frontend/src/app/features/compra/
├── services/compra.service.ts            (+ resumen, gastoPorProveedor)
├── pages/compra-historial/
└── components/{resumen-compras-cards,gasto-proveedor-chart}/
```

---

## 3. Puntos de cuidado

- **Cargar el detalle al expandir la fila**, no todo de golpe (R-06): mantiene la pantalla
  rápida con cualquier volumen de historial.
- Mostrar gasto ejecutado y comprometido **por separado** (R-04): fundirlos en una cifra da
  un número que no significa nada.
- El rango por defecto (mes actual) evita cargar el historial completo en la primera
  apertura.
- Los importes se formatean con el locale `es-CO`; los cálculos vienen del backend.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
