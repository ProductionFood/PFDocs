# 10 — Roadmap y Dependencias

Orden de ejecución de las 27 historias y qué bloquea qué.

---

## 1. Grafo de dependencias

```
FASE 1 — Infraestructura
  HU-01 Registro de usuarios
    ├─► HU-02 Gestión de usuarios
    └─► HU-03 Login ──► HU-04 Bitácora
                          │
                          ▼  (todo lo demás requiere autenticación y auditoría)
FASE 2 — Maestros
  HU-07 Unidades ──┬──► HU-08 Materias primas ──┬──► HU-09 Lotes MP
                   │                            └──► HU-10 Inventario MP
                   └──► HU-11 Productos ──┬─────────► HU-12 Recetas
  HU-05 Clientes                          └──► HU-19 Lotes PT
  HU-06 Proveedores

FASE 3 — Compras (entrada de stock)
  HU-06 ──► HU-13 Compras ──► HU-14 Detalle ──► HU-15 Historial
                                   │
                                   └──► (recepción: suma a inventario)

FASE 4 — Pedidos (salida de stock)
  HU-05 ──► HU-16 Pedidos ──► HU-17 Detalle ──► HU-18 Devoluciones
  HU-19 Lotes PT ──────────────────┘

FASE 5 — Producción
  HU-20 Plan ──► HU-21 Detalle ──► HU-22 Consumo ──► HU-23 Inventario automático
  HU-12 Recetas ──────────────────────┘

FASE 6 — Reportes
  HU-24 Dashboard · HU-25 Pedidos · HU-26 Eficiencia · HU-27 Stock bajo
```

---

## 2. Orden de ejecución

### Sprint 0 — Preparación (antes de la primera historia)

Tareas sin las cuales todo lo demás se atasca:

- [ ] Repositorio creado, con ramas y `.gitignore`.
- [ ] Instancia RDS MySQL 8.4 creada (`11-DESPLIEGUE-AWS-RDS.md`).
- [ ] Migraciones V1, V2 y V3 aplicadas en local y en RDS.
- [ ] Proyecto Spring Boot 4.1 que arranca y conecta a la base.
- [ ] Proyecto Angular 22 que arranca.
- [ ] `GlobalExceptionHandler`, `PageRequest`/`PageResponse` y `SecurityConfig` en su sitio.
- [ ] Colección de Postman con el entorno configurado.

**Esto no es una historia de usuario pero es trabajo real.** Presupuestar una semana.
Empezar HU-01 sin la infraestructura lista hace que la primera historia arrastre todo el
andamiaje y parezca que "HU-01 tardó tres semanas".

### Sprint 1 — Seguridad (HU-01, 02, 03, 04)

Bloquea absolutamente todo lo demás: sin login no hay endpoint protegido que probar y sin
bitácora hay que volver a tocar cada servicio después.

**HU-04 va en el Sprint 1, no al final.** La auditoría se implementa como aspecto AOP
transversal; retrasarla obliga a revisitar todos los servicios ya escritos.

### Sprint 2 — Maestros (HU-07, 05, 06, 08, 11)

HU-07 (unidades) primero: HU-08 y HU-11 dependen de ella.

Son cinco CRUD de estructura casi idéntica. **Se implementa HU-05 completo —backend,
frontend, pruebas— y se usa como plantilla** para los otros cuatro. Hacerlos en paralelo
desde cero multiplica por cinco las decisiones de diseño y produce cinco estilos distintos.

### Sprint 3 — Inventario y recetas (HU-09, 10, 12, 19)

Aquí aparece `InventarioService` con el kardex. Es el componente más delicado del sistema:
lo tocan compras, pedidos, devoluciones y producción.

**Se escribe una vez, bien, con pruebas unitarias, antes de que cuatro módulos dependan
de él.**

### Sprint 4 — Compras (HU-13, 14, 15)

Primer flujo que suma stock. La recepción de compra (`PATCH /compras/{id}/estado` →
`RECIBIDA`) es transaccional y genera movimientos de kardex.

### Sprint 5 — Pedidos (HU-16, 17, 18)

Primer flujo que resta stock de producto terminado, con política FEFO. Requiere HU-19 hecha.

### Sprint 6 — Producción (HU-20, 21, 22, 23)

HU-23 es la más compleja del proyecto: transaccional, con bloqueo de filas y alerta de
stock mínimo.

### Sprint 7 — Reportes (HU-24, 25, 26, 27)

Solo lectura. Dependen de que haya datos de los sprints anteriores: **son las últimas por
necesidad**, no por prioridad.

---

## 3. Dependencias externas de cada historia

| HU | Requiere cerradas | Nota |
|---|---|---|
| HU-01 | — | Sprint 0 completo |
| HU-02 | HU-01 | |
| HU-03 | HU-01 | Bloquea todo lo demás |
| HU-04 | HU-03 | Transversal: se aplica a cada módulo posterior |
| HU-05 | HU-03, HU-04 | **Plantilla de CRUD** |
| HU-06 | HU-05 | Mismo patrón |
| HU-07 | HU-03 | Bloquea HU-08 y HU-11 |
| HU-08 | HU-07 | |
| HU-09 | HU-08 | Suma inventario → requiere `InventarioService` |
| HU-10 | HU-08 | Consulta la vista `v_inventario_materia_prima` |
| HU-11 | HU-07 | |
| HU-12 | HU-08, HU-11 | UNIQUE de ingrediente por receta |
| HU-13 | HU-06 | |
| HU-14 | HU-13, HU-08 | Cálculo de totales |
| HU-15 | HU-14 | |
| HU-16 | HU-05 | |
| HU-17 | HU-16, HU-19 | **Requiere stock de PT: sin HU-19 no hay nada que vender** |
| HU-18 | HU-17 | Devuelve stock al aprobar |
| HU-19 | HU-11 | Crea inventario de PT |
| HU-20 | HU-01 | |
| HU-21 | HU-20, HU-11 | |
| HU-22 | HU-21, HU-12 | Teórico desde la receta |
| HU-23 | HU-22, HU-10 | La más compleja |
| HU-24 | HU-13, HU-16, HU-20 | Necesita datos de varios módulos |
| HU-25 | HU-17 | |
| HU-26 | HU-22 | |
| HU-27 | HU-10 | La más simple de la fase |

---

## 4. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| **HU-17 planificada antes que HU-19** | Alto | No hay stock de producto terminado que descontar: la historia no se puede probar. El roadmap ya las ordena |
| **`InventarioService` escrito dos veces** | Alto | Cuatro módulos lo consumen. Se escribe una vez en el Sprint 3, con pruebas |
| **HU-04 dejada para el final** | Medio | Obliga a revisitar todos los servicios. Va en el Sprint 1 |
| **Spring Boot 4 + Security 7 sin material de apoyo** | Medio | `06-ARQUITECTURA-BACKEND.md` trae la configuración escrita. Plan B: retroceder a 3.5.x si el bloqueo supera dos días |
| **Los cinco CRUD en paralelo desde cero** | Medio | HU-05 primero como plantilla |
| **Descuadres de inventario detectados tarde** | Alto | Conciliación de kardex al cerrar cada fase (`05-ESTANDARES-QA.md` §5) |
| **Instancia RDS olvidada facturando** | Medio | Alarma de facturación y borrado al terminar el semestre |

---

## 5. Reparto sugerido

Según el cronograma de la Entrega 1 (§10):

| Sprint | Backend | Frontend | QA / BD |
|---|---|---|---|
| 0 | Greylis, Carlos, Valeria | Ismael, Natalia | Greylis (RDS) |
| 1 | Greylis, Carlos | Ismael, Natalia | Natalia |
| 2 | Carlos, Valeria | Ismael, Natalia | Greylis |
| 3 | Greylis, Carlos | Ismael | Natalia |
| 4 | Valeria, Carlos | Natalia | Greylis |
| 5 | Greylis, Valeria | Ismael, Natalia | Natalia |
| 6 | Carlos, Greylis | Ismael | Greylis |
| 7 | Valeria | Ismael, Natalia | Natalia, Greylis |

**El frontend de una historia va un sprint detrás de su backend.** Mientras backend hace el
Sprint N, frontend cierra el N-1 contra un contrato ya estable. Trabajar ambos a la vez
sobre la misma historia obliga a rehacer el cliente cada vez que cambia una respuesta.

El contrato de `especificacion.md` es lo que permite a frontend empezar antes: se trabaja
contra él, no contra la implementación.
