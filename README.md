# ProductionFood — Documentación de Proyecto

Sistema Web para la Planificación y Gestión de la Producción en Pymes de Alimentos.
Corporación Universitaria Antonio José de Sucre — Electiva Profesional II.

---

## Cómo usar esta documentación

**Antes de escribir una sola línea de código**, todo el equipo debe leer `00-base/`.
Ahí viven las decisiones transversales: convenciones de API, correcciones al esquema
de base de datos, matriz de roles y estándares de prueba. Si una tarea de HU
contradice a `00-base/`, gana `00-base/` y se reporta la discrepancia.

## Estructura

```
docs/
├── 00-base/                      ← LEER PRIMERO. Decisiones transversales.
│   ├── 00-STACK-VERSIONES.md         Versiones y por qué se actualizaron
│   ├── 01-CONVENCIONES.md            Nomenclatura, capas, estructura de paquetes
│   ├── 02-CORRECCIONES-DB.md         Huecos del esquema y cómo se corrigen
│   ├── 03-MATRIZ-ROLES.md            Los 5 roles y qué puede hacer cada uno
│   ├── 04-CONTRATO-API.md            Request/response, errores, paginación, estados
│   ├── 05-ESTANDARES-QA.md           Cómo se prueba y cuándo se cierra una historia
│   ├── 06-ARQUITECTURA-BACKEND.md    Spring Boot 4.1, Security 7, JdbcTemplate
│   ├── 07-ARQUITECTURA-FRONTEND.md   Angular 22 standalone, signals, interceptores
│   ├── 08-DESVIACIONES-PDF.md        Diferencias vs. la Entrega 1 ya presentada
│   ├── 09-SETUP-ENTORNO.md           Puesta en marcha local paso a paso
│   ├── 10-ROADMAP.md                 Orden de ejecución y dependencias entre HU
│   ├── 11-DESPLIEGUE-AWS-RDS.md      Instancia MySQL 8.4, seguridad y costos
│   ├── 12-FLOCI-USO.md               Emulador local de AWS para pruebas
│   └── migraciones/                  Scripts SQL versionados
│       ├── V1__esquema_base.sql          Esquema original adaptado a MySQL 8.4
│       ├── V2__correcciones.sql          Correcciones, kardex, vistas, índices
│       └── V3__datos_semilla.sql         Roles, unidades, administrador inicial
│
├── HU-NN/                        ← Una carpeta por historia de usuario
│   ├── especificacion.md             Qué se construye, reglas de negocio, contrato
│   ├── diseno.md                     Pantallas, flujos, modelo de datos implicado
│   ├── tarea-backend.md              Checklist de implementación para backend
│   ├── tarea-frontend.md             Checklist de implementación para frontend
│   └── tarea-qa.md                   Casos de prueba y criterios de aceptación
│
└── previo/                       Documentos originales de la Entrega 1
    ├── Historias_de_Usuario.md       (tabla de stack actualizada a sep-2026)
    ├── db.sql                        Esquema tal como se entregó
    └── Plantilla Entrega1-...pdf
```

## Antes de empezar: lo que esta revisión encontró

La preparación detectó tres cosas que conviene conocer antes de escribir código:

1. **El esquema tiene tres huecos bloqueantes** — no hay precio en el detalle del pedido,
   no hay stock de producto terminado por producto, y **nada suma materia prima al
   inventario**. Detalle y corrección en `00-base/02-CORRECCIONES-DB.md`.
2. **Varias versiones del stack están fuera de soporte**, incluida la base de datos:
   RDS para MySQL 8.0 terminó soporte estándar el 31-jul-2026 y factura Extended Support.
   Ver `00-base/00-STACK-VERSIONES.md`.
3. **La Entrega 1 y las historias no coinciden** en alcance ni en actores.
   Registro completo en `00-base/08-DESVIACIONES-PDF.md`, con lo que hay que corregir
   para la Entrega 2.

## Índice de historias de usuario

| Fase | HU | Título | Depende de |
|---|---|---|---|
| **1. Infraestructura y seguridad** | [HU-01](HU-01/) | Registro de usuarios | — |
| | [HU-02](HU-02/) | Gestión de usuarios | HU-01 |
| | [HU-03](HU-03/) | Inicio de sesión | HU-01 |
| | [HU-04](HU-04/) | Bitácora de auditoría | HU-03 |
| **2. Datos maestros** | [HU-05](HU-05/) | Gestión de clientes | HU-03, HU-04 |
| | [HU-06](HU-06/) | Gestión de proveedores | HU-03, HU-04 |
| | [HU-07](HU-07/) | Unidades de medida | HU-03, HU-04 |
| **3. Inventario de materias primas** | [HU-08](HU-08/) | Gestión de materias primas | HU-07 |
| | [HU-09](HU-09/) | Lotes de materia prima | HU-08 |
| | [HU-10](HU-10/) | Consulta de inventario de MP | HU-08 |
| **4. Catálogo y recetas** | [HU-11](HU-11/) | Gestión de productos | HU-07 |
| | [HU-12](HU-12/) | Gestión de recetas | HU-08, HU-11 |
| **5. Compras** | [HU-13](HU-13/) | Registro de compras | HU-06 |
| | [HU-14](HU-14/) | Detalle de compra | HU-13, HU-08 |
| | [HU-15](HU-15/) | Historial de compras | HU-14 |
| **6. Pedidos y ventas** | [HU-16](HU-16/) | Creación de pedidos | HU-05 |
| | [HU-17](HU-17/) | Productos en el pedido | HU-16, HU-19 |
| | [HU-18](HU-18/) | Devoluciones | HU-17 |
| | [HU-19](HU-19/) | Lotes de producto terminado | HU-11 |
| **7. Producción** | [HU-20](HU-20/) | Plan de producción | HU-01 |
| | [HU-21](HU-21/) | Productos del plan | HU-20, HU-11 |
| | [HU-22](HU-22/) | Consumo de materia prima | HU-21, HU-12 |
| | [HU-23](HU-23/) | Inventario automático por producción | HU-22, HU-10 |
| **8. Reportes** | [HU-24](HU-24/) | Dashboard resumen | HU-13, HU-16, HU-20 |
| | [HU-25](HU-25/) | Consulta de pedidos | HU-17 |
| | [HU-26](HU-26/) | Eficiencia de producción | HU-22 |
| | [HU-27](HU-27/) | Alerta de stock bajo | HU-10 |

Ver `00-base/10-ROADMAP.md` para el orden de ejecución sugerido y el grafo de dependencias.

## Reglas de trabajo

1. **No se empieza una HU cuyas dependencias no estén cerradas.** El roadmap lo indica.
2. **El backend cierra antes que el frontend.** El contrato en `especificacion.md` es el
   punto de sincronización: frontend puede trabajar contra mocks derivados de ese contrato.
3. **QA valida contra `tarea-qa.md`, no contra la implementación.** Si la implementación
   difiere del contrato, es defecto aunque "funcione".
4. **Toda corrección al esquema pasa por una migración versionada** en `00-base/migraciones/`.
   Nadie toca la base de datos con SQL suelto.
5. **Commits atómicos**: un commit por unidad lógica, bajo ~50 líneas. Un PR por HU o por
   capa de HU, bajo ~250 líneas.
