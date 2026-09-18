# HU-09 · Lotes de materia prima — Tarea Backend

**Primera historia que suma stock.** Aquí nace `InventarioService`, el componente más delicado del proyecto: lo consumirán HU-13, HU-17, HU-18, HU-22 y HU-23. Se escribe una vez, con pruebas, antes de que cinco módulos dependan de él.

---

## 1. Pasos

### `InventarioService` (se crea aquí, lo usa todo el sistema)
1. `InventarioService.registrarEntrada(idMateriaPrima, cantidad, tipo, tablaOrigen, idOrigen, observacion)`:
   - `SELECT ... FOR UPDATE` sobre la fila de inventario (bloqueo);
   - calcula `saldoAnterior` y `saldoPosterior`;
   - `UPDATE inventario`;
   - `INSERT` en `movimientos_inventario_mp`;
   - **todo dentro de la transacción del llamador**.
2. `InventarioService.registrarSalida(...)` — simétrico, con validación de saldo suficiente.
3. **Regla de oro:** ningún otro componente ejecuta `UPDATE inventario`. Si aparece un
   `UPDATE inventario` fuera de esta clase, es un defecto.

### Módulo `lote`
4. `Lote` (POJO), `LoteRowMapper` con `JOIN materias_primas` y `unidades_medida`.
5. `LoteRepository`: `insertar`, `buscarPorId`, `existeCodigoLote(codigo)`,
   `buscarMateriaPrima(filtros, page)`, `contar(...)`.
6. DTO: `CrearLoteMateriaPrimaRequest`, `LoteMateriaPrimaResponse` (con `diasParaVencer`,
   `vencido` y los saldos).
7. `LoteService.crearLoteMateriaPrima()` **`@Transactional`**:
   - normaliza el código a mayúsculas (R-06);
   - valida código único → `CODIGO_LOTE_DUPLICADO`;
   - valida materia prima existente → `MATERIA_PRIMA_INEXISTENTE`;
   - valida fechas → `FECHAS_INVALIDAS`;
   - `INSERT` en `lotes` con `id_producto = NULL` (R-03);
   - **si `idDetalleCompra` es `null`, llama a `registrarEntrada()`** (R-01 y R-02);
     si viene informado, **no suma** — el stock ya entró con la recepción.
8. Filtros del listado: `idMateriaPrima`, `soloVencidos`, `proximosAVencer` (días).
9. `@Auditable(accion="CREAR", tabla="lotes")`.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── inventario/
│   ├── InventarioService.java          ← 🔑 componente central
│   ├── InventarioRepository.java       (+ bloquearYObtener, actualizarSaldo)
│   ├── MovimientoInventarioRepository.java
│   └── TipoMovimiento.java             (enum)
└── lote/
    ├── {Lote,LoteRowMapper,LoteRepository,LoteService,LoteController}.java
    └── dto/{CrearLoteMateriaPrimaRequest,LoteMateriaPrimaResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **R-02, la doble suma, es el defecto más costoso posible aquí.** Un lote con
  `idDetalleCompra` informado **no debe sumar**: el stock ya entró con la recepción de la
  compra. Sumar dos veces no produce ningún error visible; el inventario simplemente miente.
- **`InventarioService` es el único que hace `UPDATE inventario`.** Establecer esta regla
  ahora evita que cinco módulos la incumplan después.
- `SELECT ... FOR UPDATE` obligatorio: sin bloqueo, dos entradas simultáneas sobre la misma
  materia prima pierden una de las dos sumas.
- **El movimiento de kardex va en la misma transacción que el `UPDATE`.** Si se separan,
  el saldo y el kardex divergen y la conciliación de `05-ESTANDARES-QA.md` §5 empieza a
  fallar sin causa aparente.
- El `CHECK` de fechas ya está en la base, pero validar en el servicio da un `400` legible
  en vez de un error de integridad.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **`InventarioService` con pruebas unitarias de entrada, salida y saldo insuficiente.**
- [ ] **Verificado: un lote con `idDetalleCompra` NO suma al inventario.**
