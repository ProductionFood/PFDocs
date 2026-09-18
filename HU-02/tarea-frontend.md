# HU-02 · Gestión de usuarios — Tarea Frontend

Listado paginado con filtro, edición y activación/desactivación. **Es la pantalla plantilla de todos los listados del proyecto**: se implementa con cuidado porque los demás CRUD la copian.

---

## 1. Pasos

1. `shared/components/tabla-paginada/` — tabla genérica reutilizable con `MatTable`,
   `MatPaginator` y `MatSort`, parametrizada por columnas.
2. `shared/components/confirmar-dialog/` — diálogo de confirmación reutilizable.
3. `usuario-lista.component`:
   - columnas: nombre, correo, rol, estado, acciones;
   - campo de búsqueda con `debounceTime(350)` — sin él se lanza una petición por tecla;
   - filtros por rol y por estado;
   - paginación conectada a `PageResponse`.
4. Chip de estado con color: verde "Activo" / gris "Inactivo".
5. Acción de editar: abre el diálogo de HU-01 en modo edición (sin campos de contraseña).
6. Acción de activar/desactivar con confirmación explícita:
   "¿Desactivar a María Gómez? No podrá iniciar sesión."
7. **No incluir acción de eliminar** (CA-05): que no exista el botón.
8. Manejar `409 AUTO_DESACTIVACION` y `409 ULTIMO_ADMIN` con el mensaje del servidor.
9. Los tres estados: cargando / vacío / error.

---

## 2. Archivos

```
frontend/src/app/
├── shared/components/tabla-paginada/
├── shared/components/confirmar-dialog/
└── features/usuario/pages/usuario-lista/
```

---

## 3. Puntos de cuidado

- **El `debounceTime` no es opcional.** Sin él, escribir "maria" lanza cinco peticiones y
  las respuestas pueden llegar desordenadas, mostrando resultados de una búsqueda anterior.
  Usar también `switchMap` para cancelar la petición en vuelo.
- Al desactivarse a sí mismo, el servidor responde `409`: mostrar el mensaje, no cerrar
  sesión.
- El `PageResponse` usa `page` base 0; `MatPaginator` también. No sumar ni restar 1.
- Sin botón de eliminar: si aparece en la interfaz, alguien lo va a pedir.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
