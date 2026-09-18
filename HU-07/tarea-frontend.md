# HU-07 · Unidades de medida — Tarea Frontend

Pantalla de catálogo simple, sin paginación. Además, el servicio se **cachea** porque lo consumen los formularios de varios módulos.

---

## 1. Pasos

1. `unidad-medida.model.ts` y `unidad-medida.service.ts`.
2. **Cachear el catálogo** con `shareReplay(1)`: HU-08, HU-11 y HU-12 lo piden en cada
   apertura de formulario. Sin caché se descarga la misma lista una y otra vez.
3. Método `invalidarCache()` que se llama tras crear o editar una unidad.
4. `unidad-lista.component`: tabla simple sin paginador, ordenada por nombre.
5. Columna "En uso" con el número de referencias, para que se vea qué unidades importan.
6. `unidad-formulario.component` en `MatDialog`.
7. Al editar una unidad en uso, mostrar advertencia clara:
   *"Esta unidad se usa en 12 materias primas y 5 productos. Cambiar su abreviatura
   modificará cómo se interpretan esos registros."* — con confirmación explícita.
8. **Sin acción de eliminar** (R-03).
9. Ruta: lectura para todos los roles; botones de escritura con `*siRol="['ADMIN']"`.

---

## 2. Archivos

```
frontend/src/app/features/unidad-medida/
├── models/unidad-medida.model.ts
├── services/unidad-medida.service.ts     ← con caché shareReplay(1)
├── pages/{unidad-lista,unidad-formulario}/
└── unidad-medida.routes.ts
```

---

## 3. Puntos de cuidado

- **La caché debe invalidarse al crear o editar.** Si no, el usuario agrega una unidad y no
  la ve en el desplegable de materias primas hasta recargar la página — y concluye que la
  creación falló.
- Sin `MatPaginator` en esta pantalla (R-01).
- La advertencia de R-04 debe decir **qué va a pasar**, no solo "¿está seguro?".
- El desplegable en otros módulos muestra `"Kilogramo (kg)"`: solo el nombre es ambiguo
  entre unidades parecidas, y solo la abreviatura es críptica.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
