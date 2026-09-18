# 08 — Desviaciones frente a la Entrega 1

El documento *Plantilla Entrega 1 — Requerimientos y Arquitectura* (fecha de entrega
09/09/2026) ya fue presentado. Este trabajo de preparación encontró diferencias entre ese
documento y las historias de usuario.

**Decisión del equipo: las historias de usuario son la fuente de verdad para el alcance
funcional. El documento de arquitectura lo es para la infraestructura de datos.**

Este registro existe para corregir la Entrega 1 en su próxima versión, no para reprocharla:
un desfase entre el documento de requerimientos y el backlog es normal cuando el segundo se
detalla después. Lo que no sería normal es dejarlo sin registrar.

---

## Resumen

| # | Tema | Entrega 1 | Historias de usuario | Decisión |
|---|---|---|---|---|
| D-01 | Alcance: compras y proveedores | Excluido explícitamente | HU-06, 13, 14, 15 | **Se incluye** |
| D-02 | Alcance: pedidos, ventas, devoluciones | No mencionado | HU-16, 17, 18, 25 | **Se incluye** |
| D-03 | Alcance: lotes y trazabilidad | No mencionado | HU-09, 19 | **Se incluye** |
| D-04 | Número de requerimientos | 10 RF | 27 HU | **27 HU** |
| D-05 | Actores | 3 | 5 implícitos | **5 roles** |
| D-06 | Base de datos | MySQL en AWS RDS | MariaDB 10.4 local | **MySQL 8.4 en AWS RDS** |
| D-07 | Versiones del stack | Sin versiones | Versiones fuera de soporte | **Actualizadas** |
| D-08 | Bitácora de auditoría | No mencionada | HU-04 | **Se incluye** |

---

## D-01 · Compras y proveedores

**Entrega 1, §3.2 Limitaciones:** *"El proyecto no incluirá en esta primera versión: [...]
Gestión de proveedores y compras."*

**Historias de usuario:** HU-06 (proveedores), HU-13 (registro de compras), HU-14 (detalle
de compra), HU-15 (historial de compras).

**Decisión: se incluyen.** No es solo que estén en el backlog — es que **sin compras el
sistema no funciona**.

La razón es concreta: HU-23 descuenta materia prima del inventario al producir, HU-10 lo
consulta y HU-27 alerta cuando está bajo. Pero ninguna historia definía cómo *entra* la
materia prima. La recepción de compra es el punto de entrada principal del stock
(ver `02-CORRECCIONES-DB.md` §C-03). Excluir compras deja el inventario permanentemente en
cero y vacía de sentido tres historias más.

**Acción en Entrega 2:** retirar esa línea de §3.2 y agregar el módulo a §3.1.

---

## D-02 · Pedidos, ventas y devoluciones

**Entrega 1:** no aparecen en el alcance, ni en los requerimientos funcionales, ni en los
módulos de la sección 4.

**Historias de usuario:** HU-16 a HU-18 y HU-25.

**Decisión: se incluyen.** Son un módulo completo con impacto sobre el inventario de
producto terminado. Su ausencia en la Entrega 1 parece un descuido de redacción más que una
decisión de alcance: el modelo entidad-relación de §8 **sí incluye** `pedidos`,
`detalle_pedido` y `devoluciones`. El esquema los contemplaba; el texto no los mencionó.

**Acción en Entrega 2:** agregar el módulo de pedidos a §3.1 y sus requerimientos a §4.

---

## D-03 · Lotes y control de vencimientos

**Entrega 1:** no se menciona el manejo de lotes ni de fechas de vencimiento.

**Historias:** HU-09 (lotes de materia prima), HU-19 (lotes de producto terminado),
HU-24 CA-05 (productos próximos a vencer).

**Decisión: se incluyen.** En una pyme de alimentos la trazabilidad por lote y el control
de vencimientos no son un extra: son el requisito sanitario básico del sector. Su omisión
en el documento debilita la justificación del proyecto.

La tabla `lotes` ya estaba en el modelo entidad-relación de §8.

---

## D-04 · 10 requerimientos vs. 27 historias

Los 10 RF de la Entrega 1 son un subconjunto de las 27 historias. Correspondencia:

| RF Entrega 1 | Historia |
|---|---|
| RF-01 Registrar usuarios | HU-01 |
| RF-02 Asignar roles | HU-01, HU-02 |
| RF-03 Registrar productos | HU-11 |
| RF-04 Registrar materias primas | HU-08 |
| RF-05 Consultar inventario de MP | HU-10 |
| RF-06 Registrar recetas | HU-12 |
| RF-07 Asociar materias primas a receta | HU-12 |
| RF-08 Crear planes de producción | HU-20 |
| RF-09 Actualizar estado del plan | HU-20 |
| RF-10 Registrar cantidad producida | HU-21, HU-22 |

**Sin cobertura en la Entrega 1:** HU-02 (gestión de usuarios), HU-03 (login), HU-04
(bitácora), HU-05 a HU-07 (maestros), HU-09, HU-13 a HU-19, HU-23 a HU-27.

Llama la atención que **HU-03, el inicio de sesión, no tenga un RF asociado**, aun cuando
RNF-01 exige restringir el acceso según el rol. No se puede restringir por rol sin
autenticar.

**Acción en Entrega 2:** ampliar la tabla de §4 con los requerimientos faltantes.

---

## D-05 · Tres actores vs. cinco roles

**Entrega 1, §6:** Administrador, Encargado de producción, Usuario de consulta.

**Historias de usuario:** además de administrador y productor, aparecen *comprador*
(HU-13, 14, 15) y *vendedor* (HU-16, 17, 18).

**Decisión: cinco roles** — `ADMIN`, `PRODUCCION`, `COMPRAS`, `VENTAS`, `CONSULTA`.
Detalle completo en `03-MATRIZ-ROLES.md`.

Con tres actores, las operaciones de compras y ventas caerían todas en `ADMIN`, y el
sistema perdería la separación de funciones que RNF-01 pretende.

**Acción en Entrega 2:** ampliar la tabla de actores de §6.

---

## D-06 · Base de datos

**Entrega 1, §7 y §9:** MySQL alojado en AWS RDS.
**Historias de usuario:** MariaDB 10.4 local.

**Decisión: MySQL 8.4 LTS en AWS RDS**, como indica la Entrega 1 y confirmó el equipo.

Se añade una precisión de versión que la Entrega 1 no daba: **RDS para MySQL 8.0 terminó
su soporte estándar el 31 de julio de 2026** y pasa a facturarse como Extended Support.
Crear la instancia en 8.0 hoy implica pagar por una versión obsoleta. Detalle en
`00-STACK-VERSIONES.md` y `11-DESPLIEGUE-AWS-RDS.md`.

**Acción en Entrega 2:** especificar "MySQL 8.4 LTS" en §9, y corregir la tabla de stack de
las historias de usuario, que decía MariaDB. *(Ya corregida en `previo/Historias_de_Usuario.md`.)*

---

## D-07 · Versiones del stack

La Entrega 1 nombra las tecnologías sin versión; las historias sí las daban, y varias
estaban fuera de soporte a septiembre de 2026:

| Componente | Declarado | Estado | Adoptado |
|---|---|---|---|
| Java | 21 | LTS vigente, pero ya no el actual | **25 LTS** |
| Spring Boot | 3.3.x | Fuera de soporte OSS | **4.1.x** |
| Angular | 17 | Fuera de soporte | **22** |
| SpringDoc | 2.6.0 | Incompatible con Spring Boot 4 | **3.1.1** |
| jjwt | 0.12.6 | Funcional, hay versión más reciente | **0.13.0** |

Justificación de cada cambio en `00-STACK-VERSIONES.md`.

**Acción en Entrega 2:** incluir la columna de versión en la tabla de §9.

---

## D-08 · Bitácora de auditoría

**Entrega 1:** no la menciona en el alcance ni en los requerimientos. La tabla `bitacora`
**sí aparece** en el modelo entidad-relación de §8, pero no se describe su propósito en la
tabla de entidades.

**Historias:** HU-04 la define completa.

**Decisión: se incluye.** Es lo que da trazabilidad a las operaciones y sostiene RNF-01.

---

## Requerimientos no funcionales: sin desviaciones

Los seis RNF de la Entrega 1 se mantienen y se trasladan a criterios verificables:

| RNF | Cómo se verifica |
|---|---|
| RNF-01 Seguridad | Batería SEC-01..SEC-05 en cada endpoint (`05-ESTANDARES-QA.md` §4) |
| RNF-02 Rendimiento (< 2 s) | Aserción de tiempo en cada petición de Postman |
| RNF-03 Usabilidad | Tres estados obligatorios por pantalla (`05-ESTANDARES-QA.md` §7) |
| RNF-04 Escalabilidad | Estructura por módulos (`01-CONVENCIONES.md` §2) |
| RNF-05 Disponibilidad | Fuera del alcance de pruebas automatizadas |
| RNF-06 Mantenibilidad | Separación en capas, revisión de PR obligatoria |

---

## Qué llevar a la Entrega 2

1. Corregir §3.1 y §3.2: incluir compras, proveedores, pedidos, devoluciones y lotes.
2. Ampliar §4 con los 17 requerimientos faltantes.
3. Ampliar §6 a cinco actores.
4. Especificar versiones en §9, con MySQL 8.4 LTS.
5. Documentar la bitácora en la tabla de entidades de §8.
6. Incorporar las correcciones de esquema de `02-CORRECCIONES-DB.md` al modelo
   entidad-relación: `precio_unitario` en `detalle_pedido`, las dos tablas de kardex y las
   restricciones de unicidad.
