# HU-19 · Lotes de producto terminado — Tarea Backend

**Bloquea HU-17**: sin esta historia no hay stock que vender. Amplía el módulo `lote` de HU-09 y usa `InventarioPtService` de HU-17 — conviene coordinar el orden de implementación con quien haga esa historia.

---

## 1. Pasos

1. Ampliar `LoteRepository` con:
   - `insertarLoteProducto()` — `id_producto` informado, `id_materia_prima` a `NULL`;
   - `buscarProducto(filtros, page)` y `contar(...)`;
   - el `SELECT` hace `JOIN inventario_producto_terminado` para traer el saldo actual.
2. `InventarioPtRepository.crearParaLote(idLote, cantidad)`.
3. DTO: `CrearLoteProductoRequest`, `LoteProductoResponse` (con `cantidadIngresada` y
   `cantidadDisponible` separadas, R-03).
4. 🔑 **`LoteService.crearLoteProducto()` `@Transactional`**:
   - normaliza el código a mayúsculas (R-08);
   - valida código único → `CODIGO_LOTE_DUPLICADO`;
   - valida producto existente → `PRODUCTO_INEXISTENTE`;
   - valida fechas → `FECHAS_INVALIDAS`;
   - **valida que no esté vencido** → `LOTE_YA_VENCIDO` (R-05);
   - `INSERT` en `lotes`;
   - **`INSERT` en `inventario_producto_terminado`** (CA-05, R-02);
   - `INSERT` del movimiento `ENTRADA_PRODUCCION` en `movimientos_inventario_pt`.
5. Filtros: `idProducto`, `soloVencidos`, `proximosAVencer` (días), `conStock`.
6. Kardex del lote desde `movimientos_inventario_pt`.
7. `@Auditable(accion="CREAR", tabla="lotes")`.
8. Controlador: escritura ADMIN y PRODUCCION (`00-base/03-MATRIZ-ROLES.md`).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── lote/
│   ├── LoteRepository.java          (+ insertarLoteProducto, buscarProducto)
│   ├── LoteService.java             (+ crearLoteProducto)
│   ├── LoteController.java          (+ endpoints /lotes-producto)
│   └── dto/{CrearLoteProductoRequest,LoteProductoResponse}.java
└── inventario/
    └── InventarioPtRepository.java  (+ crearParaLote)
```

---

## 3. Puntos de cuidado

- 🔴 **Las tres inserciones en una sola transacción** (R-02). Un lote sin su fila de
  inventario no aparece en el stock, no se puede vender y **no da ningún error**: parece que
  el registro funcionó.
- 🔴 **No confundir `lotes.cantidad` con el saldo** (R-03). Es la cantidad de ingreso y no
  cambia nunca. El disponible está en `inventario_producto_terminado`. Un `SELECT` que lea
  `lotes.cantidad` para el stock mostrará siempre la cantidad original.
- El código de lote es único **en toda la tabla**, compartida con los lotes de materia prima
  de HU-09. Conviene una convención de prefijos (`PF-` producto, `H-` harina) para que no
  colisionen por accidente.
- `CHECK ck_lotes_exclusividad`: `id_materia_prima` debe ir `NULL` (R-06).
- Bloquear el registro de lotes ya vencidos (R-05), a diferencia de HU-09.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Verificado: todo lote creado tiene su fila en `inventario_producto_terminado`.**
