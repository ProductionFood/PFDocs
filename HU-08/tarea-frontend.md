# HU-08 · Gestión de materias primas — Tarea Frontend

Listado y formulario siguiendo el patrón de HU-05, con selector de unidad y campos numéricos.

---

## 1. Pasos

1. `materia-prima.model.ts` y `materia-prima.service.ts`.
2. `materia-prima-lista.component`: columnas nombre, unidad, stock mínimo, costo unitario,
   **disponible**, estado, `⋮`.
3. Indicador visual cuando `cantidadDisponible < stockMinimo` (icono de alerta ámbar) —
   anticipa HU-27 y hace útil el listado.
4. Búsqueda con `debounceTime(350)` + `switchMap`; filtro por estado.
5. `materia-prima-formulario.component`:
   - selector de unidad alimentado por el servicio cacheado de HU-07;
   - campos numéricos con sufijo de unidad dinámico ("kg" junto a stock mínimo);
   - costo con formato de moneda colombiana.
6. Si el catálogo de unidades está vacío, mostrar enlace a la pantalla de HU-07.
7. Los tres estados.
8. Ruta: lectura ADMIN/PRODUCCION/COMPRAS/CONSULTA; escritura solo ADMIN.

---

## 2. Archivos

```
frontend/src/app/features/materia-prima/
├── models/materia-prima.model.ts
├── services/materia-prima.service.ts
├── pages/{materia-prima-lista,materia-prima-formulario}/
└── materia-prima.routes.ts
```

---

## 3. Puntos de cuidado

- **Los decimales se envían como `number` de JSON**, sin separador de miles. El formato
  `$ 3.200,00` es solo de presentación: lo que viaja es `3200.00`.
- El sufijo de unidad junto a "stock mínimo" evita el error de teclear 50 pensando en
  kilos cuando la unidad configurada es gramos.
- `stockMinimo = 0` es válido: no marcarlo como error (R-06).
- El indicador de stock bajo se calcula con los datos que ya devuelve el listado; no hace
  falta una petición extra.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
