# HU-11 · Gestión de productos — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-05 | Crear producto válido | `{"nombre":"Pan francés","precio":1200,"idUnidad":5}` | `201`, `activo: true`, `tieneReceta: false` | Positivo |
| CP-02 | CA-01 | Nombre vacío | `{"nombre":""}` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-03 | CA-01 | Nombre de 101 caracteres | `"A"×101` | `400` | Borde |
| CP-04 | CA-02 | Sin descripción | Omitir `descripcion` | `201`, `descripcion: null` | Positivo |
| CP-05 | CA-02 | Descripción de 151 caracteres | `"A"×151` | `400` | Borde |
| CP-06 | CA-02 | Descripción de exactamente 150 | `"A"×150` | `201` | Borde |
| CP-07 | CA-03 | Precio omitido | Sin `precio` | `400` | Negativo |
| CP-08 | CA-03 | Precio negativo | `{"precio":-100}` | `400` o `409` — `CHECK >= 0` de V2 | Negativo |
| CP-09 | CA-03 | Precio cero | `{"precio":0}` | `201` — válido (R-06) | Borde |
| CP-10 | CA-03 | Precio con 3 decimales | `{"precio":1200.555}` | Se redondea a `1200.56` | Borde |
| CP-11 | CA-04 | Unidad inexistente | `{"idUnidad":9999}` | `409 UNIDAD_INEXISTENTE` | Negativo |
| CP-12 | CA-05 | Estado por defecto | Crear sin `estado` | `activo: true`; `estado = 1` en base | Positivo |
| CP-13 | CA-06 | Listar paginado | `?page=0&size=5` | `200`, máx. 5 | Positivo |
| CP-14 | CA-06 | Filtrar por nombre | `?nombre=Pan` | Solo los que empiezan por "Pan" | Positivo |
| CP-15 | CA-06 | Orden por precio | `?sort=precio,desc` | Ordenado descendente | Positivo |
| CP-16 | — | 🔴 **Cambiar el precio no altera pedidos existentes** | Crear pedido con producto a $1.200, subir a $1.500, consultar el pedido | El detalle sigue en `$1.200` (R-01) | Integridad |
| CP-17 | — | Desactivar con stock | Producto con lotes disponibles | `200` + advertencia; el stock se conserva (R-03) | Borde |
| CP-18 | — | **No existe DELETE** | `DELETE /productos/3` | `405` o `404` | Negativo |
| CP-19 | — | Rol VENTAS puede listar | `GET` con token VENTAS | `200` | Positivo |
| CP-20 | — | Rol VENTAS no puede crear | `POST` con token VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-21 | — | `tieneReceta` refleja la realidad | Crear receta en HU-12 y volver a consultar | `tieneReceta: true` | Positivo |

---

## 2. Batería de seguridad

Obligatoria en **cada endpoint** de esta historia:

| ID | Caso | Esperado |
|---|---|---|
| SEC-01 | Sin cabecera `Authorization` | `401 NO_AUTENTICADO` |
| SEC-02 | Token malformado | `401 NO_AUTENTICADO` |
| SEC-03 | Token expirado | `401 NO_AUTENTICADO` |
| SEC-04 | Token de rol **sin** permiso | `403 SIN_PERMISO` |
| SEC-05 | Token de usuario con `estado = 0` | `403 USUARIO_INACTIVO` |

> **SEC-04 es el caso crítico.** Si devuelve `200`, falta `@EnableMethodSecurity` y la
> autorización del sistema completo está desactivada.

> **CP-16 verifica la corrección más importante del esquema** (`02-CORRECCIONES-DB.md`
> §C-01). Si el precio del detalle del pedido se lee del catálogo en lugar de guardarse,
> cada cambio de precio reescribe silenciosamente el valor de todas las ventas anteriores,
> y los reportes de HU-25 dan un número distinto cada mes sin que nadie toque un pedido.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-16: el histórico de precios debe quedar congelado en el pedido
SELECT dp.id_detalle_pedido, dp.cantidad, dp.precio_unitario AS precio_del_pedido,
       p.precio AS precio_actual_catalogo
FROM detalle_pedido dp JOIN productos p USING (id_producto)
WHERE dp.id_pedido = 1;
-- precio_del_pedido NO debe cambiar aunque precio_actual_catalogo suba.
-- Si coinciden siempre, es que se está leyendo el precio del catálogo: defecto de §C-01.

-- CP-08: el CHECK de V2
INSERT INTO productos (nombre, precio, id_unidad) VALUES ('X', -100, 5);
-- DEBE fallar con error 3819

-- CP-12: el DEFAULT añadido en V2 §C-06
SHOW CREATE TABLE productos\G

-- CP-21: productos sin receta (R-02)
SELECT p.id_producto, p.nombre FROM productos p
LEFT JOIN recetas r USING (id_producto)
WHERE r.id_receta IS NULL AND p.estado = 1;
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
