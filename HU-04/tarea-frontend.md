# HU-04 · Bitácora de auditoría — Tarea Frontend

Pantalla de consulta, solo lectura, accesible únicamente para ADMIN.

---

## 1. Pasos

1. `features/bitacora/models/bitacora.model.ts`.
2. `bitacora.service.ts` con `listar(filtros)` y `listarAcciones()`.
3. `bitacora-lista.component`: tabla con fecha, usuario, acción, tabla afectada, detalle.
4. Filtros: rango de fechas (`MatDatepicker`), acción (`MatSelect` poblado desde la API),
   tabla afectada y usuario.
5. Mostrar **"Sistema"** cuando `usuario` es `null` (R-01), con estilo diferenciado.
6. Chip de color por tipo de acción: verde `CREAR`, azul `EDITAR`, ámbar `CAMBIAR_ESTADO`,
   rojo `ELIMINAR` / `LOGIN_FALLIDO`.
7. Orden por defecto: fecha descendente (lo más reciente primero).
8. `detalle` truncado en la celda con tooltip para el texto completo.
9. **Sin botones de crear, editar ni eliminar** (CA-04).
10. Ruta con `rolGuard(['ADMIN'])`.

---

## 2. Archivos

```
frontend/src/app/features/bitacora/
├── models/bitacora.model.ts
├── services/bitacora.service.ts
├── pages/bitacora-lista/
└── bitacora.routes.ts
```

---

## 3. Puntos de cuidado

- **Ningún control de escritura en la pantalla** (CA-04): ni crear, ni editar, ni borrar.
- `usuario: null` → "Sistema". Mostrar una celda vacía haría pensar que hubo un error de
  carga.
- El rango de fechas por defecto acota a los últimos 7 días: la bitácora crece rápido y
  cargarla entera es lento y poco útil.
- Validar en el cliente que `fechaInicio <= fechaFin` antes de llamar.
- Formatear la fecha con el locale `es-CO`.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
