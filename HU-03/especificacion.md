# HU-03 · Inicio de sesión — Especificación

**Fase 1** · Fuente: `previo/Historias_de_Usuario.md`

> **Como** usuario,
> **quiero** iniciar sesión con correo y contraseña,
> **para que** acceda a las funcionalidades según mi rol.

**Depende de:** [HU-01](../HU-01/) (debe existir al menos un usuario)

---

## 1. Criterios de aceptación

| CA | Enunciado | Cómo se implementa |
|---|---|---|
| CA-01 | Se recibe correo y contraseña | `POST /auth/login` con `LoginRequest` |
| CA-02 | Credenciales correctas → token JWT | `200` con `token`, `tipo` y datos del usuario |
| CA-03 | Usuario desactivado → `403` | Se verifica `estado = 1` **después** de validar la contraseña |
| CA-04 | Credenciales incorrectas → `401` | Mismo mensaje para correo inexistente y contraseña errónea |
| CA-05 | El token contiene `id_usuario`, `correo` y `rol` | Claims `sub`, `correo`, `nombre`, `rol` — `04-CONTRATO-API.md` §2 |

---

## 2. Endpoints

| Método | Ruta | Descripción | Roles |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Autenticar y obtener token | Público |
| `GET` | `/api/v1/auth/perfil` | Datos del usuario autenticado | Autenticado |

Reglas transversales (paginación, formato de error, autenticación) en
`00-base/04-CONTRATO-API.md`.

---

## 3. Reglas de negocio

### R-01 · Mensaje idéntico para correo inexistente y contraseña incorrecta

Ambos casos devuelven exactamente `401` con `"Correo o contraseña incorrectos."`

Si el sistema respondiera "el correo no existe" en un caso y "contraseña incorrecta" en el
otro, cualquiera podría **enumerar los correos registrados** probando direcciones: el
mensaje revela si la cuenta existe. Es una fuga de información que además facilita el
ataque dirigido por fuerza bruta, porque permite descartar correos antes de empezar.

### R-02 · El orden de las comprobaciones importa

1. Buscar el usuario por correo.
2. **Verificar la contraseña.**
3. Solo entonces, verificar `estado = 1` → `403 USUARIO_INACTIVO`.

Invertir 2 y 3 convierte el `403` en un oráculo: quien reciba "usuario inactivo" sabe que
el correo existe, sin conocer la contraseña. Se responde `401` genérico hasta haber probado
que quien pregunta conoce la credencial.

### R-03 · Comparación en tiempo constante
La verificación se hace con `passwordEncoder.matches()`, que compara en tiempo constante.
Comparar hashes con `equals()` permitiría deducir información por el tiempo de respuesta.

Cuando el correo no existe, se ejecuta igualmente un `matches()` contra un hash ficticio
para que ambos caminos tarden lo mismo. Sin eso, la diferencia de tiempo entre "correo
inexistente" (respuesta inmediata) y "contraseña incorrecta" (BCrypt tarda ~100 ms) delata
qué correos están registrados, aunque el mensaje sea idéntico.

### R-04 · Vigencia del token
1 hora (`app.jwt.expiration-ms=3600000`). **No hay refresh token** en este alcance: al
expirar se vuelve a iniciar sesión.

### R-05 · El estado se verifica en cada petición
El token es válido criptográficamente hasta que expira, incluso si el usuario se desactiva
después. Por eso `JwtAuthenticationFilter` comprueba `estado` **en cada llamada**, no solo
al autenticar (ver HU-02 R-05).

---

## 4. Modelo de datos

No requiere cambios de esquema. Consulta `usuarios` con `JOIN roles`.

```json
// POST /api/v1/auth/login
{ "correo": "maria@productionfood.local", "password": "Clave2026*" }

// 200 OK
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tipo": "Bearer",
  "expiraEn": 3600,
  "usuario": {
    "idUsuario": 12, "nombre": "María Gómez",
    "correo": "maria@productionfood.local",
    "rol": "VENTAS"
  }
}

// 401 Unauthorized
{ "code": "CREDENCIALES_INVALIDAS", "message": "Correo o contraseña incorrectos.", ... }

// 403 Forbidden
{ "code": "USUARIO_INACTIVO",
  "message": "Su usuario está desactivado. Contacte al administrador.", ... }
```

**Claims del token** (CA-05):
```json
{ "sub": "12", "correo": "maria@productionfood.local",
  "nombre": "María Gómez", "rol": "VENTAS", "iat": ..., "exp": ... }
```

---

## 5. Códigos de error

| `code` | HTTP | Cuándo |
|---|---|---|
| `CREDENCIALES_INVALIDAS` | 401 | Correo inexistente **o** contraseña incorrecta (mismo mensaje) |
| `USUARIO_INACTIVO` | 403 | Credenciales correctas pero `estado = 0` |

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

### Lo que esta historia deja fuera, y conviene saberlo

No se implementa, por estar fuera del alcance acordado:

- **Bloqueo tras N intentos fallidos.** Sin él, un atacante puede probar contraseñas
  indefinidamente. BCrypt lo hace lento (unos 100 ms por intento), lo que limita el ritmo,
  pero no lo impide.
- **Refresh token.** La sesión caduca a la hora sin previo aviso.
- **Recuperación de contraseña.** Un usuario que olvide la suya necesita que un
  administrador se la cambie — y HU-02 R-03 tampoco implementa el cambio de contraseña.
  **Es un hueco real del backlog**, no de esta historia: conviene proponerlo como historia
  nueva.

Se registran aquí para que sean decisiones conscientes y no descubrimientos durante la
sustentación.
