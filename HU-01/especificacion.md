# HU-01 · Registro de usuarios — Especificación

**Fase 1** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** administrador,
> **quiero** registrar usuarios del sistema con nombre, correo, contraseña y rol,
> **para que** controle quién tiene acceso al sistema.

**Depende de:** _Ninguna. Es la primera historia del proyecto._

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Nombre obligatorio, máx. 100 caracteres | `@NotBlank` + `@Size(max=100)` en el DTO |
| CA-02 | Correo obligatorio y **único** | `@NotBlank` + `@Email`; unicidad verificada en el servicio **y** garantizada por `UNIQUE KEY correo` |
| CA-03 | Contraseña obligatoria, máx. 200, almacenada como hash | `@NotBlank` + `@Size(min=8,max=72)`; se guarda `BCryptPasswordEncoder.encode()` |
| CA-04 | El rol debe existir en `roles` | El servicio valida contra `RolRepository` antes de insertar |
| CA-05 | Estado por defecto activo (`1`) | `DEFAULT 1` añadido en V2 (§C-06). El DTO no recibe `estado` |
| CA-06 | Se retorna el usuario creado con su `id_usuario` | `201` + `Location` + `UsuarioResponse` |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `POST` | `/api/v1/usuarios` | Registrar un usuario | ADMIN |
| `GET` | `/api/v1/roles` | Listar roles disponibles | ADMIN |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · La contraseña nunca se devuelve
`UsuarioResponse` **no tiene campo de contraseña**. Ni el hash. Un DTO de respuesta que
arrastra el hash lo expone en cada listado, en los logs del navegador y en las capturas de
pantalla que se pegan en los reportes de defectos.

### R-02 · Hash con BCrypt, nunca texto plano ni MD5/SHA
`BCryptPasswordEncoder` incorpora *salt* automático y es deliberadamente lento, lo que
encarece los ataques por fuerza bruta. Un hash SHA-256 de una contraseña corta se rompe con
tablas precalculadas en segundos.

### R-03 · Longitud de contraseña
Mínimo 8 caracteres. **Máximo 72**: BCrypt ignora todo lo que exceda ese límite, así que
aceptar 200 caracteres daría una falsa sensación de seguridad — las contraseñas de 80 y de
150 caracteres con el mismo prefijo serían equivalentes. La columna es `varchar(200)` porque
almacena el hash (60 caracteres), no la contraseña.

### R-04 · Unicidad del correo: doble comprobación
El servicio consulta antes de insertar para devolver un `409` legible, pero **la garantía
real es el `UNIQUE KEY`**: dos peticiones simultáneas con el mismo correo pasan ambas la
consulta previa. Se captura `DuplicateKeyException` y se traduce a `CORREO_DUPLICADO`.

### R-05 · Comparación de correos y acentos
La collation `utf8mb4_0900_ai_ci` es *accent-insensitive*: `maria@x.com` y `maría@x.com` se
consideran el mismo correo. Es el comportamiento deseado. El correo se normaliza a
minúsculas antes de guardar.

---

## 4. Modelo de datos

**Tabla `usuarios`**

| Columna | Tipo | Notas |
|---|---|---|
| `id_usuario` | `int unsigned` PK AI | |
| `nombre` | `varchar(100)` NOT NULL | CA-01 |
| `correo` | `varchar(100)` NOT NULL **UNIQUE** | CA-02 |
| `password` | `varchar(200)` NOT NULL | Hash BCrypt (60 car.) |
| `estado` | `tinyint(1)` NOT NULL **DEFAULT 1** | CA-05 · corregido en V2 §C-06 |
| `id_rol` | `int unsigned` NOT NULL FK → `roles` | CA-04 |

**Roles disponibles** (semilla V3): `ADMIN`, `PRODUCCION`, `COMPRAS`, `VENTAS`, `CONSULTA`.
Ver `00-base/03-MATRIZ-ROLES.md`.

```json
// POST /api/v1/usuarios
{ "nombre": "María Gómez", "correo": "maria@productionfood.local",
  "password": "Clave2026*", "idRol": 4 }

// 201 Created · Location: /api/v1/usuarios/12
{ "idUsuario": 12, "nombre": "María Gómez", "correo": "maria@productionfood.local",
  "activo": true, "rol": { "idRol": 4, "nombre": "VENTAS" } }
```

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `CORREO_DUPLICADO` | 409 | Ya existe un usuario con ese correo |
| `ROL_INEXISTENTE` | 409 | El `idRol` no existe en la tabla `roles` |
| `PASSWORD_DEBIL` | 400 | Menos de 8 caracteres |

Los transversales (`VALIDACION_FALLIDA`, `NO_AUTENTICADO`, `SIN_PERMISO`,
`RECURSO_NO_ENCONTRADO`) están en `00-base/04-CONTRATO-API.md` §5.

---

## 6. Fuera de alcance

Lo que **no** cubre esta historia y no debe implementarse aquí:
se limita estrictamente a los criterios de aceptación listados arriba.
Cualquier funcionalidad adicional se propone como historia nueva, no se agrega en silencio.

---

## 7. Definición de Terminado

Se aplica la DoD de `00-base/01-CONVENCIONES.md` §7.

## 8. Notas

### El administrador semilla es una credencial pública

`V3__datos_semilla.sql` crea `admin@productionfood.local` con contraseña `Admin123*`, que
está escrita en el repositorio. Resuelve el problema del huevo y la gallina —sin un usuario
inicial nadie puede entrar a crear el primero— pero **es una puerta abierta conocida**.

Tarea de cierre obligatoria antes de exponer la aplicación fuera de `localhost`:
crear un administrador real y desactivar el semilla (`estado = 0`).
