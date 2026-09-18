# Migraciones — ProductionFood

Scripts SQL versionados para **MySQL 8.4 LTS** (Amazon RDS). Se ejecutan **en orden** y
**una sola vez** cada uno.

| Script | Contenido |
|---|---|
| `V1__esquema_base.sql` | Esquema original adaptado a MySQL 8.4. El dump tal como se entregó está en `docs/previo/db.sql` |
| `V2__correcciones.sql` | Correcciones de `02-CORRECCIONES-DB.md`: columnas, DEFAULTs, UNIQUEs, CHECKs, kardex, vistas, índices |
| `V3__datos_semilla.sql` | Roles, unidades de medida y administrador inicial |

---

## Ejecución en desarrollo local

Se trabaja contra **MySQL 8.4 local** y se despliega en RDS. Misma versión en ambos lados:
un `CHECK` que funciona en local y falla en el servidor por diferencia de versión es una
tarde perdida.

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS productionfood CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;"
mysql -u root -p productionfood < V1__esquema_base.sql
mysql -u root -p productionfood < V2__correcciones.sql
mysql -u root -p productionfood < V3__datos_semilla.sql
```

## Ejecución contra AWS RDS

```bash
mysql -h productionfood.xxxxxxxx.us-east-1.rds.amazonaws.com \
      -u admin -p \
      --ssl-mode=REQUIRED \
      productionfood < V1__esquema_base.sql
```

`--ssl-mode=REQUIRED` no es opcional: sin él la contraseña y los datos viajan en claro por
Internet hasta la instancia. Ver `11-DESPLIEGUE-AWS-RDS.md`.

---

## Verificación posterior

```sql
-- 5 roles
SELECT id_rol, nombre FROM roles;

-- 3 vistas
SHOW FULL TABLES IN productionfood WHERE Table_type = 'VIEW';

-- 2 tablas de kardex
SHOW TABLES LIKE 'movimientos_%';

-- Los CHECK deben estar activos: esta sentencia DEBE fallar con error 3819
INSERT INTO compras (id_proveedor, fecha, estado) VALUES (1, curdate(), 'inventado');

-- Los DEFAULT de fecha deben existir
SHOW CREATE TABLE pedidos\G
-- Se espera: `fecha_pedido` date NOT NULL DEFAULT (curdate())
```

Si el `INSERT` de prueba **tiene éxito**, los CHECK no se aplicaron: se está ejecutando
sobre MySQL 5.7 o sobre MariaDB anterior a 10.2, donde los CHECK se analizan pero se
ignoran en silencio. Verificar la versión con `SELECT VERSION();` antes de continuar.

---

## Reglas

1. **Nunca se edita un script ya ejecutado.** Si V2 tiene un error y ya corrió en la
   máquina de alguien, se crea `V4__fix_xxx.sql`. Editar un script aplicado produce bases
   divergentes entre integrantes, y el síntoma aparece días después como "a mí sí me
   funciona".
2. **Un tema por migración**, con nombre descriptivo.
3. **El `ROLLBACK` no protege el DDL.** MySQL hace *commit* implícito en cada
   `CREATE`/`ALTER`/`DROP`, así que la transacción del script solo cubre los `UPDATE` de
   datos. Por eso: **respaldar antes de ejecutar sobre una base con datos**
   (`mysqldump`, o una snapshot de RDS, que tarda un minuto).
4. **Se prueba primero sobre una base desechable**, nunca directo sobre la compartida ni
   sobre RDS.

---

## Sintaxis: diferencias MySQL vs. MariaDB

Si alguien copia SQL de un tutorial de MariaDB, esto es lo que va a fallar:

| Construcción | MySQL 8.4 | MariaDB |
|---|---|---|
| Expresión en `DEFAULT` | `DEFAULT (curdate())` — **paréntesis obligatorios** | `DEFAULT curdate()` |
| `CURRENT_TIMESTAMP` en datetime | Sin paréntesis (excepción histórica) | Ambas formas |
| Ancho de entero | `int unsigned` — `int(10)` obsoleto desde 8.0.17 | `int(10) unsigned` aceptado |
| Collation por defecto | `utf8mb4_0900_ai_ci` | `utf8mb4_general_ci` |
| `ORDER BY` dentro de una vista | **Se ignora** | Se respeta |

Las dos primeras producen errores de sintaxis inmediatos y son fáciles de detectar. La
última es traicionera: la vista se crea sin protestar y devuelve filas en orden arbitrario.
Por eso `v_lotes_disponibles_fefo` documenta que el orden FEFO se aplica en la consulta que
la usa, no dentro de la vista.

---

## Nota sobre herramientas

Las migraciones se ejecutan a mano porque el stack acordado no incluye Flyway ni Liquibase.
La convención de nombres (`V<n>__<descripcion>.sql`) es deliberadamente la de Flyway: si más
adelante se adopta, los scripts sirven sin tocarlos.
