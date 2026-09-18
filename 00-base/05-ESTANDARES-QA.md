# 05 — Estándares de QA

Cómo se prueba, qué se reporta y cuándo una historia se considera cerrada.

---

## 1. Alcance de las pruebas

El stack acordado incluye **Postman** como herramienta de prueba de API. Se organiza así:

| Nivel | Herramienta | Quién | Cobertura mínima |
|---|---|---|---|
| Prueba de API (funcional) | Postman | QA | **100% de los criterios de aceptación**, casos positivos y negativos |
| Prueba unitaria de servicio | JUnit 5 + Mockito | Backend | Reglas de negocio con ramificación: cálculos, validaciones, transiciones |
| Prueba de integración | `@SpringBootTest` + MockMvc | Backend | Endpoints con efectos sobre varias tablas (HU-14, 17, 18, 23) |
| Prueba manual de interfaz | Navegador | QA | Flujos completos por HU, incluidos errores |

**No se exige cobertura total de pruebas unitarias.** Se exige que toda regla con
ramificación esté cubierta: los cálculos de HU-14, HU-22 y HU-26, las transiciones de
estado, y las validaciones de stock. Un *getter* no necesita prueba; una fórmula de
eficiencia que divide por cero, sí.

---

## 2. Colección de Postman

### Estructura

```
ProductionFood API/
├── 00 - Setup/
│   ├── Login ADMIN
│   ├── Login PRODUCCION
│   ├── Login COMPRAS
│   ├── Login VENTAS
│   └── Login CONSULTA
├── HU-01 - Registro de usuarios/
│   ├── CP-01 Crear usuario válido
│   ├── CP-02 Crear con correo duplicado → 409
│   └── ...
└── HU-NN - .../
```

### Variables de entorno

| Variable | Ejemplo | Origen |
|---|---|---|
| `baseUrl` | `http://localhost:8080/api/v1` | Fija |
| `tokenAdmin` | `eyJ...` | Guardada por el script de "Login ADMIN" |
| `tokenVentas` | `eyJ...` | Guardada por el script de "Login VENTAS" |
| `idClienteCreado` | `7` | Encadenamiento entre peticiones |

Script de la petición de login:

```javascript
pm.test("Login exitoso", () => pm.response.to.have.status(200));
pm.environment.set("tokenAdmin", pm.response.json().token);
```

**Ningún token se escribe a mano en la colección.** Expiran en una hora; una colección con
tokens pegados deja de funcionar al día siguiente y hace perder media mañana averiguando
por qué.

### Aserciones mínimas por petición

```javascript
pm.test("Código de estado correcto", () => pm.response.to.have.status(201));
pm.test("Responde en menos de 2 segundos", () => pm.expect(pm.response.responseTime).to.be.below(2000));
pm.test("Devuelve el id generado", () => pm.expect(pm.response.json().idCliente).to.be.a('number'));
```

El umbral de 2 segundos es el RNF-02 del documento de requerimientos. Se verifica en cada
petición, no solo cuando alguien se acuerda.

---

## 3. Casos de prueba: formato

Cada `tarea-qa.md` de HU usa esta tabla:

| ID | Criterio | Descripción | Datos de entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01 | Crear cliente con nombre válido | `{"nombre":"Panadería El Trigo"}` | `201`, cuerpo con `idCliente` y `activo: true` | Positivo |
| CP-02 | CA-01 | Crear cliente sin nombre | `{"nombre":""}` | `400`, `code: VALIDACION_FALLIDA`, `fieldErrors[0].field = "nombre"` | Negativo |

**Todo criterio de aceptación tiene al menos un caso positivo y uno negativo.** Un CA que
solo se prueba por el camino feliz no está probado: los defectos viven en los bordes.

### Tipos de caso

- **Positivo** — el flujo esperado funciona.
- **Negativo** — la entrada inválida se rechaza con el código correcto y mensaje útil.
- **Borde** — límites exactos: longitud máxima, cantidad = 0, stock exactamente igual al
  solicitado, fecha de vencimiento = hoy.
- **Seguridad** — sin token, con token de rol no autorizado, con token expirado.
- **Integridad** — el estado de la base después de la operación es el correcto (no basta
  con que la respuesta sea `200`).

---

## 4. Batería obligatoria de seguridad

Se aplica **a cada endpoint**, sin excepción:

| ID | Caso | Esperado |
|---|---|---|
| SEC-01 | Petición sin cabecera `Authorization` | `401 NO_AUTENTICADO` |
| SEC-02 | Token malformado (`Bearer xxx`) | `401 NO_AUTENTICADO` |
| SEC-03 | Token expirado | `401 NO_AUTENTICADO` |
| SEC-04 | Token válido de un rol **sin** permiso | `403 SIN_PERMISO` |
| SEC-05 | Token de un usuario con `estado = 0` | `403 USUARIO_INACTIVO` |

**SEC-04 no es opcional.** Es la prueba que detecta la falta de `@EnableMethodSecurity`
(ver `03-MATRIZ-ROLES.md` §4): sin esa anotación, las `@PreAuthorize` se ignoran en
silencio, todo responde `200` y nada parece roto. Es el tipo de defecto que sobrevive
hasta producción porque nadie lo busca. Si SEC-04 devuelve `200`, la autorización del
sistema completo está desactivada.

### Inyección SQL

En todo campo de filtro de texto se prueba con:

```
' OR '1'='1
'; DROP TABLE clientes; --
%
```

Esperado: se tratan como texto literal. Cero resultados o resultados coherentes, nunca un
error 500 ni una lista completa. El `%` merece atención aparte: sin escapado devuelve
**todos** los registros, y eso no lanza ningún error — simplemente filtra de más
(ver `04-CONTRATO-API.md` §4).

---

## 5. Pruebas de integridad de datos

Para las HU que mueven inventario (HU-13, 14, 17, 18, 19, 22, 23) **no basta la respuesta
HTTP**. Se verifica el estado de la base:

```sql
-- Antes de la operación
SELECT cantidad_disponible FROM inventario WHERE id_materia_prima = 3;   -- 50.00

-- (ejecutar el endpoint que consume 12 kg)

-- Después
SELECT cantidad_disponible FROM inventario WHERE id_materia_prima = 3;   -- 38.00 esperado

-- Y el kardex debe tener el movimiento correspondiente
SELECT tipo_movimiento, cantidad, saldo_anterior, saldo_posterior
FROM movimientos_inventario_mp
WHERE id_materia_prima = 3 ORDER BY id_movimiento DESC LIMIT 1;
-- SALIDA_PRODUCCION | 12.000 | 50.00 | 38.00
```

La regla de oro del kardex: **`saldo_posterior` del último movimiento debe ser igual a
`cantidad_disponible` en `inventario`.** Si difieren, hay una ruta de código que actualiza
el saldo sin registrar el movimiento — exactamente lo que el kardex existe para impedir.

Consulta de conciliación, ejecutada al cerrar cada fase:

```sql
SELECT i.id_materia_prima, i.cantidad_disponible, m.saldo_posterior
FROM inventario i
LEFT JOIN (
    SELECT id_materia_prima, saldo_posterior,
           ROW_NUMBER() OVER (PARTITION BY id_materia_prima ORDER BY id_movimiento DESC) rn
    FROM movimientos_inventario_mp
) m ON m.id_materia_prima = i.id_materia_prima AND m.rn = 1
WHERE i.cantidad_disponible <> COALESCE(m.saldo_posterior, 0);
-- Debe devolver 0 filas.
```

---

## 6. Prueba de transacciones

Las HU-14, HU-17, HU-18 y HU-23 declaran operaciones transaccionales. Se prueba que el
*rollback* funciona, provocando un fallo en mitad de la operación:

1. Registrar consumo de materia prima para un plan con **dos** ingredientes.
2. El segundo ingrediente tiene stock insuficiente a propósito.
3. Esperado: `409 STOCK_INSUFICIENTE` **y el primer ingrediente no descontado**.

Un sistema que descuenta el primero y falla en el segundo deja el inventario corrupto de
forma silenciosa. Este caso es obligatorio en toda HU marcada como transaccional.

---

## 7. Pruebas de interfaz

Cada pantalla se verifica en sus **tres estados**, no solo con datos:

| Estado | Qué se espera |
|---|---|
| **Cargando** | Indicador visible; los controles deshabilitados |
| **Vacío** | Mensaje explicativo con acción sugerida, no una tabla en blanco |
| **Error** | Mensaje comprensible en español y opción de reintentar |
| **Con datos** | Paginación, ordenamiento y filtros funcionando |

Además, por formulario:

- [ ] Los campos obligatorios se marcan antes de enviar.
- [ ] Los mensajes de error del servidor se muestran junto al campo correspondiente
      (`fieldErrors` del contrato).
- [ ] El botón de envío se deshabilita mientras la petición está en vuelo
      (evita el doble registro por doble clic).
- [ ] Al cancelar con cambios sin guardar se pide confirmación.
- [ ] La navegación por teclado (Tab) recorre los campos en orden lógico.

Se verifica en **Chrome y Firefox**, a 1366×768 como mínimo.

---

## 8. Reporte de defectos

```markdown
### DEF-NNN · [HU-NN] Título breve y específico

**Severidad:** Crítica | Alta | Media | Baja
**Entorno:** Local · Backend commit `a1b2c3d` · Frontend commit `e4f5g6h`

**Pasos para reproducir**
1. Autenticarse como VENTAS
2. POST /api/v1/pedidos/5/detalles con {"idProducto": 3, "cantidad": 1000}
3. Observar la respuesta

**Resultado esperado:** 409 STOCK_INSUFICIENTE
**Resultado obtenido:** 201, el pedido se crea y el inventario queda en -970.00

**Evidencia:** captura + respuesta JSON + consulta SQL del estado posterior
```

### Escala de severidad

| Nivel | Definición | Ejemplos |
|---|---|---|
| **Crítica** | Pérdida o corrupción de datos, o brecha de seguridad | Inventario negativo, endpoint sin autorización, contraseñas en texto plano |
| **Alta** | Un criterio de aceptación no se cumple y no hay forma de sortearlo | No se puede crear un pedido |
| **Media** | CA incumplido con alternativa viable, o cálculo incorrecto en un reporte | El filtro por fecha excluye el último día |
| **Baja** | Cosmético, texto, alineación | Etiqueta mal redactada |

**Un defecto crítico bloquea el cierre de la fase completa**, no solo de su HU.

---

## 9. Criterios de cierre

### Por historia de usuario
- [ ] 100% de los casos de `tarea-qa.md` ejecutados.
- [ ] 100% de los positivos pasan.
- [ ] 100% de los negativos devuelven el código y `code` correctos.
- [ ] La batería SEC-01..SEC-05 pasa en todos los endpoints de la HU.
- [ ] Cero defectos críticos o altos abiertos.
- [ ] La colección de Postman está actualizada y versionada en el repositorio.

### Por fase
- [ ] Todas las HU de la fase cerradas.
- [ ] Prueba de regresión de las fases anteriores ejecutada.
- [ ] Conciliación de inventario (§5) devuelve 0 filas.
- [ ] Ninguna operación principal supera los 2 segundos (RNF-02).

---

## 10. Datos de prueba

Se mantiene un script `datos_prueba.sql` **fuera de las migraciones** (las migraciones son
esquema y semilla mínima; los datos de prueba son otra cosa y no deben viajar a un entorno
real). Contiene:

- 3 proveedores, 5 clientes
- 8 materias primas con inventario variado: una **agotada**, una **bajo mínimo**, una normal
- 6 productos, 4 con receta completa
- 2 compras (una `PENDIENTE`, una `RECIBIDA`)
- 3 lotes de producto terminado: uno **vencido**, uno que vence **en 3 días**, uno vigente
- 2 planes de producción en estados distintos

Los casos incómodos —el stock en cero, el lote vencido, el que vence en 3 días— están
puestos a propósito. Son los que ejercitan HU-24 CA-05 y HU-27, y los que nadie recuerda
crear a mano cuando prueba.

**Antes de cada ronda de pruebas se restaura la base desde cero:** V1 → V2 → V3 →
`datos_prueba.sql`. Probar sobre una base con residuos de la sesión anterior produce
resultados que no se pueden reproducir ni explicar.
