# HU-24 · Dashboard resumen — Tarea Backend

Solo lectura. Cinco consultas agregadas sobre datos que ya existen. El reto es el rendimiento (R-04), no la lógica.

---

## 1. Pasos

1. `DashboardRepository` con cinco métodos, cada uno una consulta agregada:
   - `pedidosDelDia()` — cuenta y total con `SUM(cantidad * precio_unitario)`, agrupado por
     estado;
   - `comprasPendientes()` — **sin filtro de fecha** (R-02), incluye la más antigua;
   - `stockBajo()` — sobre `v_inventario_materia_prima`, `estado_stock IN ('BAJO','AGOTADO')`;
   - `produccionDelDia()` — planes con `fecha_produccion = curdate()`, con avance promedio;
   - `proximosAVencer(dias)` — lotes con stock que vencen en ≤ N días, **incluidos los ya
     vencidos** (R-03).
2. **Usar `curdate()` del servidor**, no `LocalDate.now()` de Java (R-01): una sola fuente
   de fecha.
3. `DashboardService` con `@Transactional(readOnly = true)`; ejecuta las cinco consultas y
   arma el DTO.
4. DTO: `DashboardResponse` con los cinco bloques anidados.
5. Endpoint único `GET /dashboard` que las agrupa, más dos endpoints de detalle.
6. **Verificar con `EXPLAIN` que las cinco usan los índices de V2 §C-15.**
7. Sin caché (R-04).
8. `@PreAuthorize("isAuthenticated()")` en el endpoint principal; el filtrado por rol es de
   presentación (R-05).

---

## 2. Archivos

```
backend/src/main/java/co/edu/corposucre/productionfood/reporte/
├── {DashboardRepository,DashboardService,DashboardController}.java
└── dto/{DashboardResponse,PedidosHoyResponse,ComprasPendientesResponse,
        StockBajoResponse,ProduccionHoyResponse,PorVencerResponse}.java
```

---

## 3. Puntos de cuidado

- 🔴 **`curdate()` del servidor, no `LocalDate.now()` de Java** (R-01). Si la aplicación y
  la base están en zonas distintas, el dashboard y los datos discrepan a partir de las 7 p.m.
- **Compras pendientes sin filtro de fecha** (R-02): una compra pendiente de hace un mes es
  la más importante de todas.
- **Los lotes por vencer deben tener stock** (R-03), y los ya vencidos con existencias se
  incluyen: son acción inmediata.
- **Sin caché** (R-04): un dashboard operativo con datos rancios lleva a decisiones erróneas.
- Verificar el plan de ejecución de las cinco consultas: es la historia donde más fácil se
  incumple el RNF-02.
- El total de ventas depende de `detalle_pedido.precio_unitario` (§C-01, R-06).

---

## 4. Checklist de cierre

- [ ] Todos los CA implementados y verificados manualmente.
- [ ] Endpoints documentados en Swagger con ejemplos.
- [ ] `@PreAuthorize` en cada método, según la matriz de roles.
- [ ] Operaciones de escritura auditadas en bitácora (HU-04).
- [ ] Pruebas unitarias de las reglas con ramificación.
- [ ] Sin `System.out.println` ni código comentado.
- [ ] PR bajo ~250 líneas, revisado y aprobado.
