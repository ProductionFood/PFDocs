# HU-09 · Lotes de materia prima — Tarea Frontend

Listado de lotes con indicadores de vencimiento y formulario de registro. La pantalla debe dejar claro que registrar un lote **modifica el inventario**.

---

## 1. Pasos

1. `lote.model.ts` y `lote-materia-prima.service.ts`.
2. `lote-mp-lista.component`: código, materia prima, producción, vencimiento, cantidad,
   estado de vencimiento.
3. Semáforo de vencimiento: rojo vencido · ámbar ≤ 7 días · gris ≤ 30 días · verde resto.
4. Filtros: materia prima (`MatSelect`), solo vencidos, próximos a vencer.
5. `lote-mp-formulario.component`:
   - `MatDatepicker` para ambas fechas, con `min` de vencimiento atado a producción;
   - **aviso visible**: "Registrar este lote sumará 100,00 kg al inventario de Harina de
     trigo" — calculado en vivo al completar el formulario;
   - advertencia si la fecha de vencimiento ya pasó (R-05), sin bloquear.
6. Tras guardar, mostrar el saldo resultante: "Inventario actualizado: 12,50 → 112,50 kg".
7. Los tres estados.
8. Ruta: lectura ADMIN/PRODUCCION/COMPRAS/CONSULTA; escritura ADMIN/PRODUCCION/COMPRAS.

---

## 2. Archivos

```
frontend/src/app/features/lote/
├── models/lote.model.ts
├── services/lote-materia-prima.service.ts
├── pages/{lote-mp-lista,lote-mp-formulario}/
└── lote.routes.ts
```

---

## 3. Puntos de cuidado

- **El aviso de impacto en inventario no es decorativo.** Registrar un lote cambia el stock;
  si la pantalla no lo dice, el usuario no relaciona una acción con la otra y acaba
  duplicando entradas (R-02).
- El `min` del datepicker de vencimiento debe atarse a la fecha de producción seleccionada:
  evita el `400 FECHAS_INVALIDAS` antes de enviarlo.
- Los lotes vencidos se muestran, no se ocultan (R-05): están físicamente en la bodega.
- Mostrar el saldo anterior y posterior tras guardar cierra el ciclo: el usuario ve el
  efecto sin ir a otra pantalla.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
