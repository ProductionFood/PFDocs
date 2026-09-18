# HU-05 · Gestión de clientes — Tarea Frontend

**Pantalla de referencia de los listados del proyecto.** Reutiliza `tabla-paginada` y `confirmar-dialog` creados en HU-02.

---

## 1. Pasos

1. `cliente.model.ts`: `Cliente`, `CrearClienteDto`, `ActualizarClienteDto`.
2. `cliente.service.ts`: `listar`, `obtener`, `crear`, `actualizar`, `cambiarEstado`.
3. `cliente-lista.component`: tabla con nombre, contacto, teléfono, estado y menú `⋮`.
4. Búsqueda por nombre con `debounceTime(350)` + `switchMap`.
5. Filtro por estado (todos / activos / inactivos), por defecto **activos**.
6. `cliente-formulario.component` en `MatDialog`, para crear y editar.
7. Confirmación al desactivar, explicando la consecuencia.
8. Los tres estados: cargando / vacío / error, distinguiendo vacío-por-filtro.
9. Ruta con `rolGuard(['ADMIN','VENTAS','PRODUCCION','CONSULTA'])`; los botones de
   escritura con `*siRol="['ADMIN','VENTAS']"`.

---

## 2. Archivos

```
frontend/src/app/features/cliente/
├── models/cliente.model.ts
├── services/cliente.service.ts
├── pages/{cliente-lista,cliente-formulario}/
└── cliente.routes.ts
```

---

## 3. Puntos de cuidado

- **`debounceTime` + `switchMap`** en la búsqueda: sin `switchMap`, las respuestas pueden
  llegar desordenadas y mostrar el resultado de una búsqueda anterior.
- Ocultar los botones de escritura para `CONSULTA` y `PRODUCCION` — cosmética, la
  autorización real es del backend.
- Filtro por defecto en "activos": es lo que casi siempre se busca.
- Como es la plantilla: componentes reutilizables y sin lógica duplicada.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
