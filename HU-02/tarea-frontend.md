# HU-02 · Gestión de usuarios — Tarea Frontend

Listado paginado con filtro, edición y activación/desactivación. **Es la pantalla plantilla
de todos los listados del proyecto**: se implementa con cuidado porque los demás CRUD la
copian.

> **Revisión 2026-10-01.** El repo Frontend aún no tiene el formulario de usuario de HU-01
> ni componentes compartidos: esta tarea los **construye**, no los reutiliza. Contrato
> ajustado a la especificación revisada (filtros `idRol`/`activo`, `AUTO_DEGRADACION`,
> `405` en `DELETE`).

---

## 1. Plan de trabajo

Estimación: **~900 líneas** (TS + HTML + SCSS + pruebas). Se parte en tres entregas por
comportamiento, alineadas con las tareas del backend (`tarea-backend.md` §1). Cada una se
puede mergear sola y deja la pantalla usable hasta donde llega.

| Entrega | Qué habilita | Requiere backend | Líneas aprox. |
|---|---|---|---|
| **F1 · Consultar** | Listado con búsqueda, filtros, orden y paginación (CA-01, CA-02) | T1 + T2 | ~400 |
| **F2 · Editar** | Diálogo de edición con errores por campo (CA-03) | PR-D | ~300 |
| **F3 · Estado** | Activar/desactivar con confirmación (CA-04, CA-05) | PR-E | ~200 |

F1 supera las ~250 líneas por PR: se entrega en **dos PRs**.

| PR | Contenido | Líneas |
|---|---|---|
| F1a | `core/models/{page-response,usuario}.ts`, `UsuarioService.listar/obtener`, `shared/components/tabla-paginada/` | ~200 |
| F1b | `usuario-lista.component` con búsqueda (`debounceTime` + `switchMap`), filtros y los tres estados | ~200 |
| F2 | `usuario-form-dialog` (modo edición, sin contraseña) + `UsuarioService.actualizar` | ~300 → partir si pasa de 250 (formulario / manejo de errores) |
| F3 | `shared/components/confirmar-dialog/` + `UsuarioService.cambiarEstado` + acciones del menú | ~200 |

Reglas de commit: una cosa por commit, refactor separado del cambio de comportamiento
(skill `change-size`). El formulario de F2 se diseña para servir también al **alta** de
HU-01 (`modo: 'crear' | 'editar'`), pero el modo crear entra en su propio PR.

---

## 2. Pasos

1. **Modelos espejo** de los DTO: `PageResponse<T>`, `Usuario`
   (`idUsuario, nombre, correo, activo, rol{idRol,nombre}`), `ActualizarUsuarioRequest`,
   `CambiarEstadoRequest`, `ErrorResponse`.
2. `shared/components/tabla-paginada/` — tabla genérica con `MatTable`, `MatPaginator` y
   `MatSort`, parametrizada por columnas; solo las columnas marcadas `ordenable` llevan
   `mat-sort-header` (aquí: nombre y correo).
3. `usuario-lista.component`:
   - columnas: nombre, correo, rol, estado, acciones;
   - búsqueda con `debounceTime(350)` + `switchMap`;
   - filtros rol (`GET /roles`) y estado, que se traducen a `idRol` y `activo`;
   - parámetros vacíos **no se envían** (nada de `?idRol=null`);
   - paginación conectada a `PageResponse`.
4. Chip de estado: verde "Activo" / gris "Inactivo".
5. Editar: diálogo sin campos de contraseña. En la fila del propio usuario, el selector de
   rol está deshabilitado.
6. Activar/desactivar con confirmación explícita:
   "¿Desactivar a María Gómez? No podrá iniciar sesión ni continuar su sesión actual."
   En la fila del propio usuario no se ofrece **Desactivar**.
7. **No incluir acción de eliminar** (CA-05).
8. Errores del servidor según la tabla de `diseno.md` §4 (`CORREO_DUPLICADO`,
   `ROL_INEXISTENTE`, `ULTIMO_ADMIN`, `AUTO_DEGRADACION`, `AUTO_DESACTIVACION`).
9. Los tres estados: cargando / vacío (con y sin filtro) / error.

---

## 3. Archivos

```
frontend/src/app/
├── core/models/{page-response,usuario,error-response}.ts
├── features/usuario/
│   ├── services/usuario.service.ts
│   ├── pages/usuario-lista/
│   └── components/usuario-form-dialog/
├── shared/components/tabla-paginada/
└── shared/components/confirmar-dialog/
```

---

## 4. Puntos de cuidado

- **El `debounceTime` no es opcional**, y va con `switchMap` para cancelar la petición en
  vuelo: sin él, las respuestas pueden llegar desordenadas y mostrar una búsqueda anterior.
- `403 SIN_PERMISO` o `403 USUARIO_INACTIVO` en cualquier llamada: el interceptor cierra la
  sesión o redirige; la pantalla no los trata. Un admin degradado (R-06) los recibirá en
  su siguiente petición.
- Los `409` de esta HU **no** cierran sesión: se muestra el `message` del servidor.
- El `PageResponse` usa `page` base 0; `MatPaginator` también. No sumar ni restar 1.
- No enviar `sort` sobre columnas no ordenables: el backend responde
  `400 CAMPO_ORDEN_INVALIDO`.
- Sin botón de eliminar: si aparece en la interfaz, alguien lo va a pedir.

---

## 5. Checklist de cierre

- [ ] F1, F2 y F3 mergeadas, cada PR por debajo de ~250 líneas (o razón escrita).
- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
