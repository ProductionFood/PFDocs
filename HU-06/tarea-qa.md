# HU-06 · Gestión de proveedores — Casos de Prueba

Formato y criterios en `00-base/05-ESTANDARES-QA.md`.

---

## 1. Casos funcionales

| ID | CA | Descripción | Entrada | Resultado esperado | Tipo |
|---|---|---|---|---|---|
| CP-01 | CA-01, CA-03 | Crear con solo el nombre | `{"nombre":"Harinas del Caribe"}` | `201`, `activo: true`, resto `null` | Positivo |
| CP-02 | CA-01 | Nombre vacío | `{"nombre":""}` | `400 VALIDACION_FALLIDA` | Negativo |
| CP-03 | CA-01 | Nombre de 101 caracteres | `"A"×101` | `400` | Borde |
| CP-04 | CA-02 | Crear con todos los campos | nombre, contacto, teléfono, correo | `201` con los cuatro | Positivo |
| CP-05 | CA-02 | Correo con formato inválido | `{"correo":"no-es-correo"}` | `400`, error en `correo` | Negativo |
| CP-06 | CA-02 | Correo omitido | Sin la clave `correo` | `201`, `correo: null` | Borde |
| CP-07 | CA-02 | Correo como cadena vacía | `{"correo":""}` | `201`; se guarda `null` (R-03) | Borde |
| CP-08 | CA-02 | Dos proveedores con el mismo correo | Repetir correo | `201` — no hay UNIQUE (R-03) | Borde |
| CP-09 | CA-03 | Estado por defecto | Crear sin `estado` | `activo: true` | Positivo |
| CP-10 | CA-04 | Listar paginado | `?page=0&size=5` | `200`, máx. 5 | Positivo |
| CP-11 | CA-04 | Filtrar por nombre | `?nombre=Hari` | Solo los que empiezan por "Hari" | Positivo |
| CP-12 | CA-04 | Comodín en el filtro | `?nombre=%` | No devuelve todo | Seguridad |
| CP-13 | CA-04 | Orden fuera de lista blanca | `?sort=correo,asc` | `400 CAMPO_ORDEN_INVALIDO` | Seguridad |
| CP-14 | — | Editar proveedor | `PUT` con datos nuevos | `200` | Positivo |
| CP-15 | — | Editar inexistente | `PUT /proveedores/9999` | `404` | Negativo |
| CP-16 | CA-05 | Desactivar | `PATCH .../estado` | `200`, `estado = 0`, la fila permanece | Positivo |
| CP-17 | CA-05 | **No existe DELETE** | `DELETE /proveedores/3` | `405` o `404` | Negativo |
| CP-18 | — | Rol VENTAS intenta crear | Token de VENTAS | `403 SIN_PERMISO` | Seguridad |
| CP-19 | — | Rol COMPRAS puede crear | Token de COMPRAS | `201` | Positivo |
| CP-20 | — | Inyección SQL en el filtro | `?nombre=' OR 1=1 --` | Sin error 500 | Seguridad |

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


---

## 3. Verificación en base de datos

```sql
-- CP-07: el correo vacío se guarda como NULL, no como cadena vacía
SELECT id_proveedor, nombre, correo FROM proveedores WHERE correo = '';
-- Debe devolver 0 filas

-- CP-16/CP-17: desactivación lógica
SELECT COUNT(*) FROM proveedores;   -- comparar antes/después

-- El índice de V2 se usa en el filtro
EXPLAIN SELECT id_proveedor, nombre FROM proveedores WHERE nombre LIKE 'Hari%';
-- Se espera: key = idx_proveedores_nombre
```

---

## 4. Criterios de cierre

- [ ] Todos los casos ejecutados; los positivos pasan.
- [ ] Los negativos devuelven el HTTP y el `code` esperados.
- [ ] SEC-01 a SEC-05 pasan en todos los endpoints.
- [ ] Ningún endpoint supera 2 s (RNF-02).
- [ ] Cero defectos críticos o altos abiertos.
- [ ] Colección de Postman actualizada y versionada.
