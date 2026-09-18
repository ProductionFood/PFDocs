# 11 — Despliegue de la Base de Datos en AWS RDS

Puesta en marcha de la instancia MySQL 8.4 que usa ProductionFood.

> ⚠️ **Antes de empezar: esto cuesta dinero si se configura mal.**
> La capa gratuita de AWS cubre 750 horas/mes de `db.t4g.micro` durante 12 meses, pero
> solo si se eligen exactamente las opciones indicadas. Multi-AZ, una clase de instancia
> mayor o almacenamiento aprovisionado se facturan desde el primer minuto. Hay una lista
> de verificación de costos al final.

---

## 1. Creación de la instancia

Consola de AWS → RDS → **Create database**.

| Opción | Valor | Por qué |
|---|---|---|
| Creation method | Standard create | *Easy create* elige opciones que no son las de capa gratuita |
| Engine | **MySQL** | — |
| Version | **MySQL 8.4.x** | 8.0 salió de soporte estándar el 31-jul-2026 y factura Extended Support |
| Templates | **Free tier** | Fija automáticamente las opciones sin costo |
| DB instance identifier | `productionfood-db` | — |
| Master username | `admin` | No usar `root` |
| Master password | Generada, guardada en gestor de contraseñas | **Nunca en el repositorio** |
| Instance class | `db.t4g.micro` | Incluida en capa gratuita |
| Storage type | General Purpose SSD (gp3), 20 GB | Mínimo de capa gratuita |
| Storage autoscaling | **Desactivado** | Activo, puede crecer y facturar sin aviso |
| Multi-AZ | **No** | Duplica el costo. Innecesario para un proyecto académico |
| Public access | **Yes** (solo durante el desarrollo) | Ver §2 antes de aceptar esto |
| VPC security group | Nuevo: `productionfood-sg` | — |
| Initial database name | `productionfood` | Crea la base; evita un paso manual |
| Backup retention | 7 días | Incluido |
| Deletion protection | **Activado** | Evita borrar la instancia por accidente |

---

## 2. Acceso público y grupo de seguridad

`Public access = Yes` hace la instancia alcanzable desde Internet. Es la forma práctica de
que cinco integrantes trabajen desde sus casas sin montar una VPN, pero **la regla del
security group es lo único que separa la base de datos del mundo**.

### Lo que no se debe hacer

```
Tipo: MYSQL/Aurora  ·  Puerto: 3306  ·  Origen: 0.0.0.0/0     ❌
```

Esto expone el puerto 3306 a Internet entero. Los escáneres automáticos encuentran
instancias RDS abiertas **en cuestión de horas**, no de días, y empiezan a probar
credenciales. No es un riesgo teórico ni exagerado: es el motivo más común de bases de
datos académicas comprometidas.

### Lo correcto

Una regla por integrante, con su IP pública:

```
Tipo: MYSQL/Aurora  ·  Puerto: 3306  ·  Origen: <IP pública>/32  ·  Descripción: "Carlos - casa"
```

Cada quien obtiene su IP en [checkip.amazonaws.com](https://checkip.amazonaws.com).

Con IP dinámica del proveedor de Internet, la dirección cambia cada pocos días y hay que
actualizar la regla. Es molesto. Es también el precio de no dejar la base abierta, y
actualizar una regla toma treinta segundos.

---

## 3. Parameter Group — la zona horaria

**Este paso no es opcional y se olvida siempre.**

Una instancia RDS nueva está en **UTC**. Colombia es UTC-5. Un pedido registrado a las
7:30 p.m. se guarda con la fecha del día siguiente, porque `curdate()` lo evalúa el servidor
de base de datos.

Consecuencia: HU-24 CA-01 ("pedidos del día actual") y los filtros por fecha de HU-25 dejan
de coincidir con lo que el usuario ve. Y como solo ocurre después de las 7 p.m., las pruebas
de la mañana pasan sin problema y el defecto llega hasta la sustentación.

### Configuración

RDS → Parameter groups → **Create parameter group**

| Campo | Valor |
|---|---|
| Family | `mysql8.4` |
| Name | `productionfood-params` |

Parámetros a modificar:

| Parámetro | Valor | Motivo |
|---|---|---|
| `time_zone` | `America/Bogota` | Lo anterior |
| `character_set_server` | `utf8mb4` | Acentos y ñ |
| `collation_server` | `utf8mb4_0900_ai_ci` | Coincide con las migraciones |
| `max_connections` | Dejar por defecto | El pool de HikariCP lo gobierna desde la aplicación |

Luego: RDS → instancia → Modify → DB parameter group → `productionfood-params` →
**Apply immediately** → reiniciar la instancia (`time_zone` es estático y requiere reinicio).

Verificación:

```sql
SELECT @@time_zone, @@system_time_zone, NOW(), CURDATE();
-- Se espera: America/Bogota  ·  -05  ·  hora local de Colombia
```

En la aplicación, además:

```properties
spring.jackson.time-zone=America/Bogota
```
```bash
java -Duser.timezone=America/Bogota -jar productionfood-api.jar
```

Los dos lados. Ajustar solo la aplicación no cambia lo que `curdate()` devuelve en el
servidor.

---

## 4. Conexión desde la aplicación

### `application.properties`

```properties
spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/productionfood?useSSL=true&requireSSL=true&serverTimezone=America/Bogota&characterEncoding=utf8
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Pool: la capa gratuita es pequeña, no tiene sentido pedir 50 conexiones
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.connection-timeout=30000

spring.jackson.time-zone=America/Bogota
```

> 🔒 **Las credenciales van en variables de entorno, nunca en el archivo versionado.**
> Un `application.properties` con la contraseña de RDS subido a un repositorio público es
> una fuga de credenciales, y los bots que rastrean GitHub buscando cadenas de conexión
> encuentran ese patrón en minutos. `application.properties` se versiona con los `${...}`;
> los valores viven en el entorno local de cada integrante y **`.env` va en `.gitignore`**.

### Verificación de conectividad

```bash
mysql -h productionfood-db.xxxxxxxx.us-east-1.rds.amazonaws.com \
      -u admin -p --ssl-mode=REQUIRED productionfood -e "SELECT VERSION(), @@time_zone;"
```

---

## 5. Aplicación de las migraciones

En orden, desde `00-base/migraciones/`:

```bash
export RDS_HOST=productionfood-db.xxxxxxxx.us-east-1.rds.amazonaws.com

for f in V1__esquema_base.sql V2__correcciones.sql V3__datos_semilla.sql; do
  echo "Aplicando $f..."
  mysql -h "$RDS_HOST" -u admin -p --ssl-mode=REQUIRED productionfood < "$f" || break
done
```

**Antes de ejecutar sobre una base con datos: tomar una snapshot.** RDS → instancia →
Actions → Take snapshot. Tarda un minuto y es la diferencia entre un susto y una pérdida.
Recuérdese que el `ROLLBACK` no revierte sentencias DDL.

---

## 6. Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `Communications link failure` / cuelgue al conectar | El security group no permite tu IP | Agregar tu IP pública actual. Cambia sola con IP dinámica |
| `Access denied for user 'admin'` | Contraseña incorrecta, o se está conectando a la base equivocada | Revisar credenciales y nombre de base |
| `Public Key Retrieval is not allowed` | Driver MySQL 9 con la configuración SSL por defecto | Usar `useSSL=true` en la URL. **No** recurrir a `allowPublicKeyRetrieval=true`, que debilita la conexión |
| `Unknown database 'productionfood'` | No se indicó *Initial database name* al crear la instancia | `CREATE DATABASE productionfood CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;` |
| Las fechas aparecen con un día de diferencia | `time_zone` en UTC | §3 de este documento |
| `Error 3819: Check constraint violated` | Un CHECK está funcionando | No es un fallo: es la base rechazando un dato inválido |
| La factura llegó con cargos inesperados | Multi-AZ, autoscaling o Extended Support activos | §7 |

---

## 7. Control de costos

Lista de verificación al terminar de crear la instancia:

- [ ] Clase de instancia `db.t4g.micro` — no mayor.
- [ ] Multi-AZ **desactivado**.
- [ ] Storage autoscaling **desactivado**.
- [ ] Almacenamiento 20 GB gp3 — no aprovisionado (io1/io2 se facturan aparte).
- [ ] Versión **8.4**, no 8.0 — 8.0 incurre en cargos de Extended Support.
- [ ] **Una sola instancia.** Cinco integrantes comparten una, no crean una cada uno.
- [ ] Alarma de facturación configurada en CloudWatch (por ejemplo, USD 5).
- [ ] Recordatorio en el calendario: **la capa gratuita vence a los 12 meses**.

### Al terminar el semestre

```
RDS → instancia → Actions → Take final snapshot → Delete
```

La snapshot conserva los datos por si hay que retomar el proyecto y cuesta una fracción de
la instancia. **Una instancia RDS olvidada sigue facturando meses después de que el curso
terminó** — es el final más común de los proyectos académicos en AWS.

Recuérdese desactivar *Deletion protection* antes de borrar.

---

## 8. Desarrollo local

No hace falta —ni conviene— que los cinco integrantes trabajen contra RDS todo el tiempo:
consume la capa gratuita y cada prueba destructiva afecta a los demás.

**MySQL 8.4 local para desarrollar; RDS para integración y para la sustentación.**

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

La misma versión mayor en ambos entornos. Desarrollar sobre MySQL 5.7 o MariaDB y desplegar
en 8.4 hace que los CHECK se ignoren en local y se apliquen en el servidor: el código
funciona en la máquina de cada quien y falla en la demostración.
