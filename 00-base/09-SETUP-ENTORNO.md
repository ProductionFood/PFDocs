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

Con Docker (recomendado — la versión queda fijada y no interfiere con otras instalaciones):

```bash
docker run --name productionfood-mysql \
  -e MYSQL_ROOT_PASSWORD=local \
  -e MYSQL_DATABASE=productionfood \
  -e TZ=America/Bogota \
  -p 3306:3306 \
  -d mysql:8.4 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_0900_ai_ci \
  --default-time-zone=-05:00
```

Sin Docker: instalar MySQL 8.4 y crear la base:

```sql
CREATE DATABASE productionfood
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

`--default-time-zone=-05:00` no es opcional: sin ella, `curdate()` devuelve UTC y los
pedidos registrados después de las 7 p.m. quedan con la fecha del día siguiente
(`02-CORRECCIONES-DB.md`, sección MySQL).

### Migraciones

```bash
cd docs/00-base/migraciones
mysql -u root -p productionfood < V1__esquema_base.sql
mysql -u root -p productionfood < V2__correcciones.sql
mysql -u root -p productionfood < V3__datos_semilla.sql
```

Verificación:

```sql
SELECT VERSION(), @@time_zone;              -- 8.4.x · -05:00
SELECT COUNT(*) FROM roles;                 -- 5
SHOW TABLES LIKE 'movimientos_%';           -- 2 tablas
```

---

## 3. Backend

```bash
git clone <url-del-repositorio>
cd productionfood/backend
```

Variables de entorno. Crear `.env` local (**está en `.gitignore`**):

```bash
export DB_HOST=localhost
export DB_USER=root
export DB_PASSWORD=local
export JWT_SECRET=$(openssl rand -base64 48)
```

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

Debe devolver un token. Si devuelve `401`, la semilla V3 no se aplicó o el `PasswordEncoder`
no es BCrypt.

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
| `Unknown database 'productionfood'` | La base no se creó | Ver §2 |
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
