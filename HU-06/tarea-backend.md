# HU-06 · Gestión de proveedores — Tarea Backend

CRUD idéntico a HU-05 más el campo `correo`. **Se implementa copiando el patrón de HU-05**, no reinventándolo: si ambos difieren en estructura, mantenerlos cuesta el doble.

---

## 1. Pasos

1. Copiar la estructura del módulo `cliente` de HU-05.
2. `Proveedor` (POJO) y `ProveedorRowMapper` — incluye `correo`.
3. `ProveedorRepository`: `buscar`, `contar`, `buscarPorId`, `insertar`, `actualizar`,
   `cambiarEstado`. Lista blanca de orden: `{nombre, id}`.
4. DTO: `CrearProveedorRequest` (con `@Email` en `correo`), `ActualizarProveedorRequest`,
   `ProveedorResponse`.
5. `ProveedorService` con `@Transactional` y `@Auditable`.
6. `ProveedorController` con `@PreAuthorize` según la matriz: escritura para ADMIN y
   COMPRAS.
7. Método `existeActivo(id)` que usará HU-13 para validar R-02.
8. **Sin `DELETE`** (CA-05).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/proveedor/
├── {Proveedor,ProveedorRowMapper,ProveedorRepository,ProveedorService,ProveedorController}.java
└── dto/{CrearProveedorRequest,ActualizarProveedorRequest,ProveedorResponse}.java
```

---

## 3. Puntos de cuidado

- `@Email` acepta `null` y `""`. Si se quiere rechazar la cadena vacía, normalizarla a
  `null` en el servicio antes de guardar — así no se distingue entre "sin correo" y
  "correo vacío" en la base.
- Los mismos cuidados de HU-05: `buscar`/`contar` sincronizados, lista blanca de `sort`,
  escape de `%` y `_`.
- Exponer `existeActivo()` desde el repositorio: HU-13 lo necesita y no debe consultar la
  tabla por su cuenta.
- Sin endpoint `DELETE`.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
