# HU-14 · Detalle de compra — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01..03 | Agregar ítem válido | `{"idMateriaPrima":5,"cantidad":100,"precioUnitario":3200}` | `201` con `subtotal: 320000.00` | Positivo |
| CP-02 | CA-01 | Compra inexistente | `POST /compras/9999/detalles` | `404 COMPRA_INEXISTENTE` | Negativo |
| CP-03 | CA-01 | 🔴 **Agregar ítem a compra RECIBIDA** | `POST` sobre una recibida | `409 COMPRA_NO_EDITABLE` (R-01) | Negativo |
| CP-04 | CA-01 | 🔴 **Editar ítem de compra RECIBIDA** | `PUT` sobre una recibida | `409 COMPRA_NO_EDITABLE` | Negativo |
| CP-05 | CA-01 | 🔴 **Eliminar ítem de compra RECIBIDA** | `DELETE` sobre una recibida | `409 COMPRA_NO_EDITABLE` | Negativo |
| CP-06 | CA-02 | Materia prima inexistente | `{"idMateriaPrima":9999}` | `409 MATERIA_PRIMA_INEXISTENTE` | Negativo |
| CP-07 | CA-03 | Cantidad cero | `{"cantidad":0}` | `400 CANTIDAD_INVALIDA` — `CHECK > 0` | Negativo |
| CP-08 | CA-03 | Cantidad negativa | `{"cantidad":-5}` | `400` | Negativo |
| CP-09 | CA-03 | Precio cero | `{"precioUnitario":0}` | `201` — válido (R-05) | Borde |
| CP-10 | CA-03 | Precio negativo | `{"precioUnitario":-100}` | `400` — `CHECK >= 0` | Negativo |
| CP-11 | CA-04 | 🔴 **Cálculo del subtotal** | `cantidad 100 × precio 3.200` | `subtotal: 320000.00` exacto | Positivo |
| CP-12 | CA-04 | 🔴 **Cálculo del total** | 2 ítems: 320.000 + 160.000 | `totalCompra: 480000.00` | Positivo |
| CP-13 | CA-04 | Total con decimales | `33,33 × 3.333,33` | `111099.89` — redondeo HALF_UP, sin centavos fantasma | Borde |
| CP-14 | CA-04 | El total se actualiza al editar | Cambiar cantidad de 100 a 150 | `totalCompra` refleja el cambio | Positivo |
| CP-15 | CA-04 | El total se actualiza al eliminar | Quitar un ítem | `totalCompra` disminuye | Positivo |
| CP-16 | CA-05 | Listar el detalle | `GET /compras/12/detalles` | Ítems con nombre, unidad y subtotales | Positivo |
| CP-17 | CA-05 | Detalle de compra vacía | Compra sin ítems | `detalles: []`, `totalCompra: 0.00` | Borde |
| CP-18 | — | 🔴 **Materia prima duplicada** | Agregar una MP ya presente | `409 MATERIA_PRIMA_DUPLICADA` (R-02) | Negativo |
| CP-19 | — | 🔴 **Duplicado por SQL directo** | `INSERT` repitiendo `(id_compra, id_materia_prima)` | La base lo rechaza — `uk_detalle_compra` | Integridad |
| CP-20 | — | El costo de referencia NO se actualiza | Comprar a $3.500 una MP con costo $3.200 | `materias_primas.costo_unitario` sigue en `3200.00` (R-04) | Integridad |
| CP-21 | — | Rol VENTAS no puede agregar ítems | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-22 | — | Rol CONSULTA puede listar | `GET` con token CONSULTA | `200` | Positivo |

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

> **CP-03 a CP-05 cubren el mismo riesgo desde tres ángulos.** Es fácil recordar la
> validación de estado al agregar y olvidarla al eliminar. Una línea borrada de una compra
> ya recibida deja el total del documento sin correspondencia con el stock que entró, y sin
> ninguna señal de que eso ocurrió.
>
> **CP-13 verifica R-03.** Si el total llega con decimales como `111099.88999999999`, se
> está calculando con `double` en algún punto de la cadena.

---

## 3. Verificación en base de datos

```sql
-- 🔴 CP-19: la restricción de V2 §C-13. La segunda sentencia DEBE fallar.
INSERT INTO detalle_compra (id_compra, id_materia_prima, cantidad, precio_unitario)
VALUES (12, 5, 10, 1000);
INSERT INTO detalle_compra (id_compra, id_materia_prima, cantidad, precio_unitario)
VALUES (12, 5, 20, 1000);
-- Error 1062 esperado. Si ambas pasan, la recepción de HU-13 sumará dos veces.

SHOW CREATE TABLE detalle_compra\G
-- UNIQUE KEY `uk_detalle_compra` (`id_compra`,`id_materia_prima`)

-- 🔴 CP-11 a CP-13: el total debe coincidir exactamente con el de la API
SELECT id_compra,
       SUM(cantidad * precio_unitario) AS total_calculado,
       COUNT(*) AS items
FROM detalle_compra WHERE id_compra = 12 GROUP BY id_compra;

-- CP-13: verificar que no hay error de coma flotante
SELECT cantidad, precio_unitario, cantidad * precio_unitario AS subtotal
FROM detalle_compra WHERE id_compra = 12;
-- Los decimales deben ser exactos, no 111099.88999999999

-- CP-20: el costo de referencia permanece
SELECT nombre, costo_unitario FROM materias_primas WHERE id_materia_prima = 5;

-- CP-07/CP-10: los CHECK de V2
INSERT INTO detalle_compra (id_compra, id_materia_prima, cantidad, precio_unitario)
VALUES (12, 6, 0, 100);      -- DEBE fallar (cantidad > 0)
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
