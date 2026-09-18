# HU-11 · Gestión de productos — Tarea Frontend

Listado y formulario gemelos de HU-08, con indicadores de receta y stock.

---

## 1. Pasos

1. `producto.model.ts` y `producto.service.ts`.
2. `producto-lista.component`: nombre, descripción, precio, unidad, **receta**, **stock**,
   estado, `⋮`.
3. Indicador de receta: ✅ tiene · ⚠️ sin receta (con enlace a HU-12 para crearla).
4. Indicador de stock desde `stockDisponible`.
5. Búsqueda con `debounceTime(350)`; filtros por estado y por "sin receta".
6. `producto-formulario.component`:
   - precio con formato de moneda colombiana (`$ 1.200,00`);
   - selector de unidad desde el servicio cacheado de HU-07;
   - contador de caracteres en la descripción (máx. 150).
7. Al desactivar un producto con stock, advertir: "Este producto tiene 45 unidades en
   estante. Desactivarlo impedirá nuevas ventas, pero el stock se conserva."
8. Los tres estados.
9. Ruta: lectura ADMIN/VENTAS/PRODUCCION/CONSULTA; escritura solo ADMIN.

---

## 2. Archivos

```
frontend/src/app/features/producto/
├── models/producto.model.ts
├── services/producto.service.ts
├── pages/{producto-lista,producto-formulario}/
└── producto.routes.ts
```

---

## 3. Puntos de cuidado

- El precio se **envía** como `1200.00`; el formato `$ 1.200,00` es solo presentación.
  Con `LOCALE_ID = 'es-CO'` el pipe `currency` lo resuelve.
- El indicador "sin receta" con enlace directo a HU-12 evita que el hueco pase inadvertido:
  un producto sin receta no se puede planificar (R-02).
- Filtrar por "sin receta" es lo que permite completar el catálogo sin revisarlo entero.
- La advertencia al desactivar con stock explica la consecuencia real (R-03).

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
