# HU-19 · Lotes de producto terminado — Tarea Frontend

Listado de lotes en estante con semáforo de vencimiento y formulario de registro.

---

## 1. Pasos

1. Ampliar `lote.service.ts` con los métodos de producto terminado.
2. `lote-producto-lista.component`: código, producto, producción, vencimiento,
   **ingresado**, **disponible**, estado.
3. Mostrar **ingresado y disponible en columnas distintas** (R-03): la diferencia es lo
   vendido, y verla ahí ahorra ir al kardex.
4. Semáforo: 🔴 vencido · 🟠 ≤ 3 días · 🟡 ≤ 7 días · 🟢 resto. Los umbrales son más
   estrictos que en materia prima: el producto terminado dura menos.
5. Filtros: producto, solo con stock, próximos a vencer, incluir vencidos.
6. `lote-producto-formulario.component`:
   - `MatDatepicker` con `min` de vencimiento atado a producción;
   - **bloquear fechas de vencimiento pasadas** (R-05);
   - aviso del stock resultante del producto.
7. Tarjetas de resumen: lotes vigentes, próximos a vencer, vencidos con stock.
8. Vista de kardex del lote: qué entró, qué se vendió, qué volvió.
9. Ruta: lectura amplia; escritura ADMIN y PRODUCCION.

---

## 2. Archivos

```
frontend/src/app/features/lote/
├── services/lote-producto.service.ts
├── pages/{lote-producto-lista,lote-producto-formulario,lote-producto-kardex}/
└── components/semaforo-vencimiento/
```

---

## 3. Puntos de cuidado

- **Dos columnas, ingresado y disponible** (R-03). Mostrar solo una lleva a que alguien
  interprete la cantidad de ingreso como stock actual.
- **Los lotes vencidos con stock deben ser visibles y destacados**: son producto que hay que
  retirar físicamente del estante. Ocultarlos los hace invisibles al problema.
- Umbrales de vencimiento más cortos que en materia prima: un pan dura días, la harina meses.
- El datepicker de vencimiento con `min` = hoy (R-05) evita el `400` antes de enviarlo.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
