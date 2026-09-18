# HU-12 · Gestión de recetas — Tarea Frontend

Pantalla de cabecera más tabla de ingredientes editable. **La interfaz debe dejar explícito el significado de `cantidad_producir`** (R-02): es donde más fácilmente se introduce un error que no se detecta hasta la Fase 7.

---

## 1. Pasos

1. `receta.model.ts` y `receta.service.ts`.
2. `receta-lista.component`: producto, nombre de receta, rendimiento, nº de ingredientes,
   costo estimado, estado.
3. Marcar las recetas **sin ingredientes** como incompletas (R-05).
4. `receta-formulario.component` en dos secciones:
   - **Cabecera**: producto (solo los que no tienen receta), nombre, cantidad a producir,
     unidad;
   - **Ingredientes**: tabla editable con agregar, editar cantidad y quitar.
5. **Texto explícito junto a "cantidad a producir"**: "Cantidad que rinde una tanda
   completa de esta receta" con ejemplo. Es la mitigación de R-02.
6. Al agregar un ingrediente, **excluir del selector los ya incluidos** (CA-05): así el
   duplicado no llega ni a intentarse.
7. Costo estimado por ingrediente y total, recalculado en vivo.
8. **Mostrar el consumo por unidad**: "Esta receta usa 0,167 kg de harina por pan",
   calculado como `cantidad / cantidadProducir`. Hace visible si el rendimiento se
   interpretó mal.
9. Confirmación al quitar un ingrediente.
10. Ruta: lectura ADMIN/PRODUCCION/CONSULTA; escritura ADMIN/PRODUCCION.

---

## 2. Archivos

```
frontend/src/app/features/receta/
├── models/receta.model.ts
├── services/receta.service.ts
├── pages/{receta-lista,receta-formulario}/
├── components/detalle-receta-tabla/
└── receta.routes.ts
```

---

## 3. Puntos de cuidado

- 🔴 **El punto 5 y el punto 8 son la defensa contra R-02.** Mostrar el consumo por unidad
  calculado hace evidente el error: si alguien pone "1" en cantidad a producir pensando en
  "por unidad", verá que cada pan lleva 5 kg de harina y lo corregirá al instante.
- **Excluir del selector los ingredientes ya agregados** evita el `409` por completo.
- Las cantidades admiten **3 decimales** (R-06): el `step` del input debe ser `0.001`,
  no `0.01`. Con `0.01` no se puede registrar 0,005 kg de sal.
- El selector de producto solo muestra los que **no tienen receta** (CA-01).
- El costo estimado es referencial: indicarlo en la interfaz para que nadie lo tome como
  costo real.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
