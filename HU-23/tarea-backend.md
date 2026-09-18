# HU-23 · Inventario automático por producción — Tarea Backend

**No es una historia independiente**: es el comportamiento transaccional de HU-22. Se implementa en el mismo PR. El foco está en las garantías de consistencia.

---

## 1. Pasos

### Lo que se implementa dentro de `ConsumoService.registrar()`
1. Anotar el método con **`@Transactional`** — sin `readOnly`, sin propagación especial.
2. Aceptar **varios consumos en una sola petición** y envolverlos todos en la misma
   transacción (R-02).
3. Por cada consumo, en orden:
   - `inventarioRepository.bloquearYObtener(idMateriaPrima)` →
     `SELECT ... FOR UPDATE` (R-03);
   - si no existe la fila → `500 INVENTARIO_NO_INICIALIZADO` (defecto de HU-08 R-01);
   - validar `cantidadReal <= disponible` → `409 STOCK_INSUFICIENTE` (R-05);
   - `inventarioService.registrarSalida(idMateriaPrima, cantidadRedondeada,
     SALIDA_PRODUCCION, "consumo_materia_prima", idConsumo)`;
   - **redondear a 2 decimales con `HALF_UP`** antes de descontar (R-08).
4. **No asignar `fecha_actualizacion`**: la pone el `ON UPDATE` de V2 (R-06).
5. Tras completar los descuentos, consultar `v_inventario_materia_prima` para las materias
   primas afectadas y **construir el array de alertas** (R-04). Informativo, nunca bloquea.
6. Devolver los saldos anterior y posterior de cada movimiento.

### Verificaciones que deben quedar escritas
7. Prueba de integración: consumo de 3 ingredientes con el tercero sin stock →
   **ningún descuento aplicado**.
8. Prueba de concurrencia: dos peticiones simultáneas sobre la misma materia prima →
   los saldos finales cuadran y no hay negativos.
9. Endpoint de conciliación (perfil `dev`) que ejecute la consulta de
   `00-base/05-ESTANDARES-QA.md` §5.

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/
├── produccion/ConsumoService.java          ← @Transactional, orquesta todo
└── inventario/
    ├── InventarioService.java              (registrarSalida — ya existe desde HU-09)
    ├── InventarioRepository.java           (+ bloquearYObtener con FOR UPDATE)
    └── dto/{AlertaStockResponse,MovimientoAplicadoResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **`@Transactional` sobre el método completo** (R-02), no sobre cada consumo por
  separado. Con varios ingredientes, un fallo en el tercero debe revertir los dos primeros.
- 🔴 **`SELECT ... FOR UPDATE` antes de validar** (R-03). Validar y descontar en operaciones
  separadas permite que dos producciones consuman el mismo saldo.
- 🔴 **No asignar `fecha_actualizacion` desde Java** (R-06): la pone la base. Asignarla a
  mano introduce una segunda fuente de verdad que se desincroniza.
- 🔴 **Redondeo `HALF_UP` a 2 decimales, siempre igual** (R-08). Es el error que no se ve:
  cada producción pierde miligramos y a las semanas el inventario no cuadra con la bodega.
- **La alerta no bloquea** (R-04): la materia prima ya se consumió.
- **Todo pasa por `InventarioService`** (R-07). Si aparece un `UPDATE inventario` en este
  servicio, es un defecto.
- El `CHECK (cantidad_disponible >= 0)` es la última red, no la primera: si salta, hay un
  fallo en el bloqueo o en la validación.

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
- [ ] **Prueba de rollback: fallo en el tercer ingrediente no deja descuentos aplicados.**
- [ ] **Prueba de concurrencia: dos peticiones simultáneas, sin saldos negativos.**
- [ ] **Conciliación kardex ↔ saldo: 0 filas.**
