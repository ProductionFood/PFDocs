# 00 — Stack Tecnológico y Versiones

Revisión de versiones a **septiembre de 2026**. Varias de las declaradas originalmente
están fuera de soporte; aquí está el stack corregido y por qué.

---

## 1. Auditoría del stack original

| Componente | Versión original | Estado a sep-2026 | Acción |
|---|---|---|---|
| Java | 21 (LTS) | ✅ LTS con soporte, pero ya no es el LTS actual | **Actualizar a 25** |
| Spring Boot | 3.3.x | ❌ Fuera de soporte OSS (3.3 terminó en 2025) | **Actualizar a 4.1.x** |
| Base de datos | MySQL 8.0 en AWS RDS | 🔴 **Fin de soporte estándar en RDS: 31-jul-2026** | **Actualizar a MySQL 8.4 LTS** |
| Angular | 17 | ❌ Fuera de soporte (nov 2023, ciclo de 18 meses) | **Actualizar a 22** |
| SpringDoc OpenAPI | 2.6.0 | ❌ Incompatible con Spring Boot 4 | **Actualizar a 3.1.1** |
| jjwt | 0.12.6 | ⚠️ Funciona, hay versión más reciente | **Actualizar a 0.13.0** |
| Maven | 3.8.x | ⚠️ Antiguo | **Actualizar a 3.9.x** |
| Angular Material | — | Sigue la versión de Angular | 22.x |

### El caso urgente: MySQL 8.0 en RDS

**Amazon RDS para MySQL 8.0 alcanzó el fin de soporte estándar el 31 de julio de 2026**,
mes y medio antes del inicio de este proyecto. La comunidad MySQL lo declaró EOL en abril
de 2026.

Qué implica en la práctica:

- Una instancia RDS que siga en 8.0 pasa automáticamente a **Extended Support**, que
  **se factura aparte y encarece la instancia**. En una cuenta de estudiante con crédito
  limitado, eso consume el presupuesto sin que nadie se dé cuenta hasta ver la factura.
- AWS puede forzar la actualización mayor en una ventana de mantenimiento.
- Crear una instancia nueva en 8.0 hoy es empezar pagando soporte extendido por una versión
  obsoleta.

**Se crea la instancia directamente en MySQL 8.4 LTS.** No es una preferencia técnica:
es la única versión que no incurre en cargos de soporte extendido.

> **Nota sobre MariaDB.** La tabla de stack de las historias de usuario mencionaba
> MariaDB 10.4 local. El documento de arquitectura (Entrega 1) declara MySQL en AWS RDS,
> y esa es la decisión vigente confirmada por el equipo. Toda la documentación, las
> migraciones y el driver apuntan a **MySQL 8.4 en AWS RDS**.

---

## 2. Stack adoptado

### Backend

| Capa | Tecnología | Versión | Soporte hasta |
|---|---|---|---|
| Lenguaje | Java (OpenJDK / Temurin) | **25 LTS** | Sep 2030 (Premier) |
| Framework | Spring Boot | **4.1.x** | Jul 2027 (OSS) |
| Seguridad | Spring Security (incluido en Boot 4.1) | 7.x | Con Boot 4.1 |
| JWT | `io.jsonwebtoken:jjwt` | **0.13.0** | Activo |
| Acceso a datos | Spring `JdbcTemplate` (sin ORM) | Con Boot 4.1 | — |
| Base de datos | **MySQL en AWS RDS** | **8.4 LTS** | Abr 2032 (comunidad) |
| Driver | `com.mysql:mysql-connector-j` | 9.x | Activo |
| Build | Maven | **3.9.x** | Activo |
| Documentación API | SpringDoc OpenAPI | **3.1.1** | Activo |
| Validación | Jakarta Validation | Con Boot 4.1 | — |
| Pruebas | JUnit 5 + Mockito + MockMvc | Con Boot 4.1 | — |

### Frontend

| Capa | Tecnología | Versión |
|---|---|---|
| Framework | Angular | **22.x** |
| Lenguaje | TypeScript | 5.9.x (la que fije Angular 22) |
| UI | Angular Material | 22.x |
| Node.js | Node | 22 LTS o 24 LTS |
| Gestor de paquetes | npm | 10+ |

### Herramientas

| Uso | Herramienta |
|---|---|
| Pruebas de API | Postman |
| Control de versiones | Git + GitHub |
| IDE backend | IntelliJ IDEA Community 2026.x |
| IDE frontend | VS Code |

---

## 3. Justificación de cada cambio

### Java 21 → 25 LTS

Java 21 **sigue soportado** y no es una elección incorrecta. Se actualiza porque:

- Java 25 es el LTS vigente desde septiembre de 2025, con Premier Support hasta 2030.
- El período de solapamiento de licencia permisiva de Java 21 termina en **octubre de 2026**,
  el mes siguiente al inicio de este proyecto.
- Spring Boot 4.1 soporta Java 17–26; no hay fricción.

Aporta al proyecto, sin ser determinante: *pattern matching* maduro para los `switch` de
transición de estado, y `record` para los DTO —que se usan en todo el proyecto— ya
plenamente asentados.

### Spring Boot 3.3 → 4.1

Spring Boot 3.3 terminó su soporte OSS en 2025, y **3.5 lo terminó en junio de 2026**.
Arrancar un proyecto nuevo sobre una rama sin soporte no tiene defensa razonable cuando
no hay código heredado que migrar: el costo de elegir bien es cero hoy y creciente después.

> ⚠️ **Riesgo asumido, y cómo se mitiga.**
> Spring Boot 4 trae Spring Framework 7 y Spring Security 7. La mayoría de tutoriales de
> JWT con Spring que se encuentran en línea son para Boot 3.x, y la configuración de
> seguridad cambió. El equipo va a encontrar ejemplos que no compilan.
>
> Por eso `06-ARQUITECTURA-BACKEND.md` incluye la configuración de seguridad **escrita
> para Spring Security 7**, no un enlace a un tutorial. Ese documento es la referencia;
> si un ejemplo de internet lo contradice, el ejemplo está desactualizado.
>
> **Plan B:** si el equipo se bloquea más de dos días con la configuración de seguridad,
> es aceptable retroceder a **Spring Boot 3.5.x**, que tiene soporte comercial hasta 2032
> y material de aprendizaje abundante. Se documentaría como deuda técnica. Es preferible
> un proyecto que avanza en 3.5 a uno detenido en 4.1.

### MySQL 8.0 → 8.4 LTS (AWS RDS)

Explicado arriba: 8.0 ya está fuera de soporte estándar en RDS y entra en facturación
extendida. MySQL 8.4 es LTS, con soporte de la comunidad hasta abril de 2032 — cubre el
proyecto con margen sobrado.

Lo que cambia entre 8.0 y 8.4 **no afecta a este proyecto**: las diferencias están en
replicación, plugins de autenticación por defecto y variables de sistema retiradas.
El SQL de las migraciones es idéntico en ambas.

Sí hay dos detalles de sintaxis MySQL que importan y que difieren de MariaDB
(ver `02-CORRECCIONES-DB.md` §Notas MySQL):

- Las expresiones en `DEFAULT` van **entre paréntesis**: `DEFAULT (curdate())`.
  Sin los paréntesis, MySQL rechaza la sentencia.
- El ancho de visualización `int(10)` está **obsoleto desde MySQL 8.0.17**. El esquema
  usa `int unsigned`.

### Driver: `mariadb-java-client` → `mysql-connector-j`

Consecuencia directa del cambio de motor. Cambia también la URL JDBC:

```
jdbc:mysql://<endpoint-rds>:3306/productionfood
```

### Angular 17 → 22

Angular 17 es de noviembre de 2023 y quedó fuera del ciclo de soporte (18 meses) a mediados
de 2025. Angular 22 (junio de 2026) es el actual.

Lo que cambia para este proyecto, en concreto:

- **Signals estables** — el estado local por componente que pide `01-CONVENCIONES.md` §4
  se resuelve sin `BehaviorSubject`.
- **Componentes standalone por defecto** — desde Angular 19 no hay que declarar
  `standalone: true`; los `NgModule` por feature ya no se usan.
- **Nueva sintaxis de control de flujo** (`@if`, `@for`, `@switch`) en lugar de
  `*ngIf` / `*ngFor`. Es la forma actual y la que se usa en los ejemplos de estos documentos.
- **Angular ARIA** (estable en 22) — accesibilidad sin trabajo extra.

Igual que con la base de datos: el proyecto no tiene código todavía, así que el costo de
migración es exactamente cero.

### SpringDoc 2.6.0 → 3.1.1

Obligado: SpringDoc 2.x **no funciona con Spring Boot 4**. La rama 3.x existe precisamente
para dar soporte a Boot 4. No es una preferencia, es la única versión que arranca.

---

## 4. Dependencias Maven

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.1</version>
    <relativePath/>
</parent>

<properties>
    <java.version>25</java.version>
    <springdoc.version>3.1.1</springdoc.version>
    <jjwt.version>0.13.0</jjwt.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>${springdoc.version}</version>
    </dependency>

    <!-- jjwt: api en compile, impl y jackson en runtime (es el patrón del proyecto) -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>${jjwt.version}</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>${jjwt.version}</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>${jjwt.version}</version>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 5. Dependencias npm

```bash
npm install -g @angular/cli@22
ng new productionfood-frontend --routing --style=scss --ssr=false
cd productionfood-frontend
ng add @angular/material@22
```

`--ssr=false`: es una aplicación interna tras autenticación, no un sitio público. El
renderizado del lado del servidor añade complejidad sin beneficio aquí.

---

## 6. Compromiso de mantenimiento

1. **Nadie actualiza una versión mayor sin acuerdo del equipo.** Las versiones de este
   documento se fijan en `pom.xml` y `package.json`, y se respetan.
2. **`package-lock.json` y las versiones exactas del `pom.xml` se versionan en Git.**
   Sin esto, cada integrante compila contra dependencias distintas y aparecen fallos que
   solo le ocurren a una persona.
3. **Se revisa este documento al inicio de cada fase.** Si sale una versión de parche con
   arreglo de seguridad, se aplica; las versiones mayores esperan al final del proyecto.

---

## 7. Fuentes

Versiones verificadas en septiembre de 2026:

- Spring Boot — [endoflife.date/spring-boot](https://endoflife.date/spring-boot)
- Angular — [endoflife.date/angular](https://endoflife.date/angular)
- MySQL en RDS — [Versiones de MySQL en Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Concepts.VersionMgmt.html) · [Estrategias de actualización 8.0 → 8.4](https://aws.amazon.com/blogs/database/upgrade-strategies-for-amazon-rds-for-mysql-8-0-to-8-4/) · [RDS Extended Support](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/extended-support.html)
- Java — [OpenJDK 25](https://openjdk.org/projects/jdk/25/) · [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)
- SpringDoc — [springdoc.org/v4](https://springdoc.org/v4/)
- jjwt — [github.com/jwtk/jjwt](https://github.com/jwtk/jjwt/releases)
