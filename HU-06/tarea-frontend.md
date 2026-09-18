# HU-06 · Gestión de proveedores — Tarea Frontend

Copia de la pantalla de HU-05 con el campo de correo añadido. Reutiliza `tabla-paginada` y `confirmar-dialog`.

---

## 1. Pasos

1. `proveedor.model.ts`, `proveedor.service.ts` siguiendo el patrón de HU-05.
2. `proveedor-lista.component`: columnas nombre, contacto, teléfono, correo, estado, `⋮`.
3. Búsqueda con `debounceTime(350)` + `switchMap`; filtro por estado (activos por defecto).
4. `proveedor-formulario.component` con el campo de correo y validación de formato.
5. El correo en la tabla como enlace `mailto:`.
6. Confirmación al desactivar: "No se podrán registrar nuevas compras a este proveedor."
7. Los tres estados de la pantalla.
8. Ruta con `rolGuard(['ADMIN','COMPRAS','CONSULTA'])`; botones de escritura con
   `*siRol="['ADMIN','COMPRAS']"`.

---

## 2. Archivos

```
frontend/src/app/features/proveedor/
├── models/proveedor.model.ts
├── services/proveedor.service.ts
├── pages/{proveedor-lista,proveedor-formulario}/
└── proveedor.routes.ts
```

---

## 3. Puntos de cuidado

- El correo es **opcional**: no marcarlo con asterisco ni bloquear el guardado si está
  vacío. Solo validar el formato cuando tiene contenido.
- Si el correo está vacío, enviar `null`, no `""`.
- En pantallas estrechas, la columna de correo es la primera que se oculta: es la menos
  crítica para identificar a un proveedor.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
