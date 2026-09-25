# 09 — Puesta en Marcha del Entorno

Pasos para dejar ProductionFood funcionando en una máquina nueva.

---

## 1. Requisitos

| Herramienta | Versión | Verificar con |
|---|---|---|
| JDK (Temurin) | 25 | `java -version` |
| Maven | 3.9.x | `mvn -version` |
| Node.js | 22 o 24 LTS | `node -v` |
| npm | 10+ | `npm -v` |
| MySQL | 8.4 | `mysql --version` |
| Git | cualquiera | `git --version` |
| Postman | actual | — |

`java -version` debe decir 25. Con varios JDK instalados, `JAVA_HOME` manda sobre el
`java` del `PATH`, y Maven usa `JAVA_HOME`: es la causa habitual de que el proyecto compile
en un equipo y no en otro.

---

## 2. Base de datos local

Con Docker (recomendado — la versión queda fijada y no interfiere con otras instalaciones).
El compose vive en el Backend y auto-carga su `.env` (§3):

```bash
cd Backend
docker compose up -d     # MySQL 8.4 + phpMyAdmin en http://localhost:8081
```

Requiere `Backend/.env` con `DB_NAME` y `DB_PASSWORD`. El servicio `mysql` crea la base
`MYSQL_DATABASE` al levantar el volumen; **el esquema lo construye Flyway** al arrancar el
backend (§2 *Migraciones*). Sin Docker: instalar MySQL 8.4 y crear la base a mano:

```sql
CREATE DATABASE productionfood
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

`--default-time-zone=-05:00` no es opcional: sin ella, `curdate()` devuelve UTC y los
pedidos registrados después de las 7 p.m. quedan con la fecha del día siguiente
(`02-CORRECCIONES-DB.md`, sección MySQL).

### Migraciones

El esquema **no se carga con SQL suelto**: el backend aplica sus migraciones de Flyway
automáticamente al arrancar. Viven junto al código, en
`Backend/src/main/resources/db/migration/` (`V1__esquema_base.sql`,
`V2__correcciones.sql`, `V3__datos_semilla.sql`), y su historial queda en la tabla
`flyway_schema_history`. `docs/00-base/migraciones/` queda como **guía de contenido**
(modelo E/R y correcciones), no como mecanismo de carga.

Verificación (tras el primer arranque del backend):

```sql
SELECT version, description, success FROM productionfood.flyway_schema_history
 ORDER BY installed_rank;             -- 3 filas: 1, 2, 3 — success = 1
SELECT COUNT(*) FROM roles;                 -- 5
SHOW TABLES LIKE 'movimientos_%';           -- 2 tablas
```

---

## 3. Backend

```bash
git clone <url-del-repositorio>
cd productionfood/Backend
```

Variables de entorno. Crear `.env` local (**está en `.gitignore`**):

```bash
export DB_HOST=localhost
export DB_NAME=productionfood
export DB_USER=root
export DB_PASSWORD=local
export JWT_SECRET=$(openssl rand -base64 48)
```

El mismo `.env` alimenta el compose de §2 (`DB_NAME`, `DB_PASSWORD`) y la aplicación.

```bash
source .env
mvn clean install
mvn spring-boot:run
```

Comprobaciones:

- API en `http://localhost:8080`
- Swagger en `http://localhost:8080/swagger-ui.html`
- Prueba de login:

```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
     -H "Content-Type: application/json" \
     -d '{"correo":"admin@productionfood.local","password":"Admin123*"}'
```

Debe devolver un token. Si devuelve `409 CREDENCIALES_INVALIDAS`, la semilla V3 no se
aplicó, el `PasswordEncoder` no es BCrypt, o el usuario está inactivo.

> ⚠️ **La contraseña `Admin123*` es pública**: está en el script de semilla, en este
> repositorio. Sirve para arrancar en local. Antes de exponer la aplicación fuera de
> `localhost`, crear un administrador real y desactivar este usuario
> (tarea de cierre en `HU-01/tarea-backend.md`).

---

## 4. Frontend

```bash
cd productionfood/frontend
npm ci        # no 'npm install': ci respeta package-lock.json exactamente
ng serve
```

Aplicación en `http://localhost:4200`.

`npm ci` en lugar de `npm install`: instala las versiones exactas del *lock file*. Con
`npm install`, cada integrante puede acabar con versiones menores distintas y aparecen
fallos que solo le ocurren a una persona.

---

## 5. Postman

1. Importar `qa/ProductionFood.postman_collection.json`.
2. Importar `qa/local.postman_environment.json`.
3. Seleccionar el entorno **local**.
4. Ejecutar la carpeta `00 - Setup` para obtener los tokens de los cinco roles.

Los tokens duran una hora; al expirar se vuelve a ejecutar esa carpeta.

---

## 6. Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `Communications link failure` al arrancar | MySQL no está corriendo o el puerto difiere | `docker ps` / revisar `DB_HOST` |
| `Unknown database 'productionfood'` | El volumen se creó sin `MYSQL_DATABASE` | Recrear: `docker compose down -v` en `Backend` y `up -d` (§2) |
| La app arranca sin crear `flyway_schema_history` | Falta el módulo `spring-boot-flyway` en el classpath (Boot 4) | Dependencia `spring-boot-flyway` en `pom.xml` del Backend |
| `Access denied for user 'root'` | Contraseña incorrecta | Revisar `DB_PASSWORD` |
| `WeakKeyException` de jjwt | `JWT_SECRET` con menos de 256 bits | Regenerar con `openssl rand -base64 48` |
| Todos los endpoints responden `403` | Falta el token, o falta `@EnableMethodSecurity` | Ver `06-ARQUITECTURA-BACKEND.md` §2 |
| Todos los endpoints responden `200` incluso con rol incorrecto | **Falta `@EnableMethodSecurity`** | Agregarla. La autorización está desactivada |
| CORS bloqueado en el navegador | El origen del frontend no está permitido | Revisar `corsConfigurationSource` |
| Las fechas aparecen con un día de diferencia | Zona horaria en UTC | §2 y `11-DESPLIEGUE-AWS-RDS.md` §3 |
| `Error 3819: Check constraint violated` | Un CHECK está funcionando | No es un fallo |
| El proyecto no compila tras actualizar | Versión de JDK distinta | `echo $JAVA_HOME` debe apuntar al JDK 25 |

---

## 7. Estructura del repositorio

```
productionfood/
├── backend/          Proyecto Maven (Spring Boot)
├── frontend/         Proyecto Angular
├── docs/             Esta documentación
│   ├── 00-base/
│   ├── HU-NN/
│   └── previo/       Documentos originales de la Entrega 1
├── qa/               Colecciones de Postman y datos de prueba
└── README.md
```

### `.gitignore` — lo que nunca se versiona

```gitignore
.env
*.env.local
application-local.properties
backend/target/
frontend/node_modules/
frontend/dist/
.idea/
.vscode/
*.log
```

**`.env` en el repositorio es una fuga de credenciales.** Los bots que rastrean GitHub
buscando cadenas de conexión y secretos las encuentran en minutos, no en días. Si llega a
subirse una contraseña, no basta con borrarla en un commit posterior: queda en el historial
y **hay que rotarla**.
