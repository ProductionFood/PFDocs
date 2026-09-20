# 12 — Uso de Floci para Pruebas Locales de AWS

Guia completa de Floci: emulador local de servicios AWS para desarrollo y pruebas
sin necesidad de una cuenta real en la nube.

> **Cuando leer este documento?** Cuando una historia de usuario necesite consumir
> servicios AWS (S3, SQS, Lambda, etc.) y se quiera probar sin costos ni dependencia
> de internet. Ver para decidir si Floci aplica a tu caso.

---

## 1. Que es Floci y cuando usarlo?

**Floci** es un emulador local, open source (MIT), que replica servicios de AWS en
`localhost:4566`. No necesita cuenta de AWS, token de autenticacion ni conexion a
internet. Cualquier SDK o cliente AWS que apunte a `http://localhost:4566` funciona
sin cambios de codigo.

### Comparacion con LocalStack

Floci reemplazo a LocalStack Community (dejo de ser gratuita en marzo 2026):

| Aspecto | Floci | LocalStack Community |
|---|---|---|
| Token de auth requerido | Nunca | Obligatorio |
| Startup | ~24 ms | ~3.300 ms |
| Memoria idle | ~13 MiB | ~143 MiB |
| Imagen Docker | ~90 MB | ~1.000 MB |
| License | MIT (siempre gratis) | Restringida |
| Actualizaciones de seguridad | Activas | Congeladas |
| Servicios soportados | 100+ | Limitados |

### Cuando SI instalar Floci en ProductionFood?

Cuando una HU necesite consumir servicios AWS en desarrollo. Casos tipicos:

| Servicio AWS | Caso de uso en ProductionFood |
|---|---|
| **S3** | Almacenar imagenes de productos, reportes PDF, documentos de compra |
| **SQS** | Colas de mensajes para procesamiento asincrono (notificaciones, reportes) |
| **Lambda** | Funciones serverless para tareas programadas (alertas de stock bajo) |
| **DynamoDB** | Cache de sesiones, datos flexibles de auditoria |
| **CloudWatch** | Logging y metricas en el cloud |

### Cuando NO instalar Floci?

Si el proyecto solo usa MySQL (ya sea local o RDS) y no consume ningun otro servicio
AWS, Floci no es necesario. MySQL 8.4 en Docker ya es suficiente para desarrollo local.

---

## 2. Instalacion

### 2.1 Opcion A — Docker Compose (recomendado)

Agregar el servicio `floci` al `docker-compose.yml` existente:

```yaml
services:
  # ... servicios existentes (mysql, phpmyadmin) ...

  # ==========================================================================
  # Floci — Emulador local de AWS (solo cuando se necesita)
  # Acceder en: http://localhost:4566
  # ==========================================================================
  floci:
    image: floci/floci:latest
    container_name: productionfood-floci
    restart: unless-stopped
    ports:
      - "4566:4566"
    volumes:
      - floci-data:/app/data

# Agregar el volumen al final del archivo
volumes:
  mysql-data:
  floci-data:
```

Iniciar solo Floci:

```bash
docker compose up -d floci
```

Iniciar todo (MySQL + phpMyAdmin + Floci):

```bash
docker compose up -d
```

### 2.2 Opcion B — Docker directo

```bash
docker run --rm -d \
  --name productionfood-floci \
  -p 4566:4566 \
  -v floci-data:/app/data \
  floci/floci:latest
```

### 2.3 Opcion C — CLI de Floci

```bash
# Instalar el CLI (Linux/macOS)
curl -fsSL https://floci.io/install.sh | sh

# Iniciar
floci start

# Exportar variables de entorno
eval $(floci env)
```

### Verificacion

```bash
# Verificar que Floci responde
curl -s http://localhost:4566/_localstack/health | head -5
```

---

## 3. Variables de entorno

Floci acepta credenciales ficticias. No se necesita una cuenta real de AWS.

### Variables del host (para AWS CLI y SDKs)

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

Agregar al `.env` del proyecto (sin versionar):

```bash
# AWS Local (Floci) — solo para desarrollo
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

### Variables de Docker Compose

Si Floci corre como servicio en Docker, las variables del host apuntan al contenedor:

```bash
# En el .env (sin versionar)
export AWS_ENDPOINT_URL=http://localhost:4566
```

---

## 4. Integracion con Spring Boot

### 4.1 Dependencias en `pom.xml`

Para usar servicios AWS con el SDK v2:

```xml
<!-- AWS SDK v2 — agregar solo los modulos que se necesiten -->

<!-- S3 (almacenamiento de archivos) -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.31.0</version>
</dependency>

<!-- SQS (colas de mensajes) -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>sqs</artifactId>
    <version>2.31.0</version>
</dependency>

<!-- DynamoDB (NoSQL) -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>dynamodb</artifactId>
    <version>2.31.0</version>
</dependency>
```

### 4.2 Configuracion en `application.properties`

```properties
# AWS Local (Floci) — desarrollo
app.aws.endpoint-url=http://localhost:4566
app.aws.region=us-east-1
app.aws.access-key=test
app.aws.secret-key=test
```

### 4.3 Configuracion del cliente S3

```java
@Configuration
public class AwsConfig {

    @Value("${app.aws.endpoint-url:}")
    private String endpointUrl;

    @Value("${app.aws.region:us-east-1}")
    private String region;

    @Value("${app.aws.access-key:test}")
    private String accessKey;

    @Value("${app.aws.secret-key:test}")
    private String secretKey;

    @Bean
    public S3Client s3Client() {
        var builder = S3Client.builder()
                .region(Region.of(region))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(accessKey, secretKey)));

        // En desarrollo, apuntar a Floci
        if (endpointUrl != null && !endpointUrl.isBlank()) {
            builder.endpointOverride(URI.create(endpointUrl))
                   .forcePathStyle(true);  // Obligatorio para Floci/LocalStack
        }

        return builder.build();
    }
}
```

> **`forcePathStyle(true)` es obligatorio** para Floci. Sin esto, el SDK intenta
> usar subdominios virtual-hosted (`bucketname.s3.amazonaws.com`) que no resuelven
> en localhost.

### 4.4 Ejemplo: subir archivo a S3

```java
@Service
public class ArchivoService {

    private final S3Client s3;
    private final String bucketName;

    public ArchivoService(S3Client s3,
                          @Value("${app.s3.bucket:productionfood-archivos}") String bucketName) {
        this.s3 = s3;
        this.bucketName = bucketName;
    }

    public String subirArchivo(String clave, InputStream contenido, String contentType) {
        s3.putObject(b -> b.bucket(bucketName).key(clave),
                     RequestBody.fromInputStream(contenido, -1));
        return clave;
    }

    public byte[] descargarArchivo(String clave) {
        return s3.getObjectAsBytes(b -> b.bucket(bucketName).key(clave)).asByteArray();
    }

    public void eliminarArchivo(String clave) {
        s3.deleteObject(b -> b.bucket(bucketName).key(clave));
    }
}
```

### 4.5 Ejemplo: enviar mensaje a SQS

```java
@Service
public class NotificacionService {

    private final SqsClient sqs;
    private final String queueUrl;

    public NotificacionService(SqsClient sqs,
                               @Value("${app.sqs.queue-notificaciones:}") String queueUrl) {
        this.sqs = sqs;
        this.queueUrl = queueUrl;
    }

    public void enviarNotificacion(String mensaje) {
        sqs.sendMessage(b -> b.queueUrl(queueUrl).messageBody(mensaje));
    }
}
```

---

## 5. Testing con Testcontainers

Testcontainers levanta un contenedor Floci aislado por cada suite de pruebas. No
necesitas Floci corriendo manualmente — el contenedor arranca y se destruye solo.

### 5.1 Dependencias de testing en `pom.xml`

```xml
<!-- Testcontainers (solo scope test) -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.20.4</version>
    <scope>test</scope>
</dependency>

<!-- Modulo Floci para Testcontainers -->
<dependency>
    <groupId>io.floci</groupId>
    <artifactId>testcontainers-floci</artifactId>
    <version>1.14.0</version>
    <scope>test</scope>
</dependency>

<!-- Spring Boot + Testcontainers (para @ServiceConnection) -->
<dependency>
    <groupId>io.floci</groupId>
    <artifactId>spring-boot-testcontainers-floci</artifactId>
    <version>1.14.0</version>
    <scope>test</scope>
</dependency>
```

### 5.2 Test de integracion con S3

```java
@Testcontainers
class S3IntegrationTest {

    @Container
    static FlociContainer floci = new FlociContainer();

    @Test
    void shouldCreateBucketAndUploadFile() {
        S3Client s3 = S3Client.builder()
                .endpointOverride(URI.create(floci.getEndpoint()))
                .region(Region.of(floci.getRegion()))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(floci.getAccessKey(), floci.getSecretKey())))
                .forcePathStyle(true)
                .build();

        // Crear bucket
        s3.createBucket(b -> b.bucket("test-productos"));

        // Subir archivo
        s3.putObject(b -> b.bucket("test-productos").key("imagen.jpg"),
                     RequestBody.fromString("contenido-falso"));

        // Verificar
        var objetos = s3.listObjectsV2(b -> b.bucket("test-productos")).contents();
        assertThat(objetos).hasSize(1);
        assertThat(objetos.get(0).key()).isEqualTo("imagen.jpg");

        // Limpiar
        s3.deleteObject(b -> b.bucket("test-productos").key("imagen.jpg"));
        s3.deleteBucket(b -> b.bucket("test-productos"));
    }
}
```

### 5.3 Test con Spring Boot y `@ServiceConnection`

Con `@ServiceConnection`, Spring Boot auto-configura el endpoint, region y credenciales
para todos los beans de SDK AWS del contexto. No necesitas `application-test.yml`.

```java
@SpringBootTest
@Testcontainers
class ArchivoServiceIntegrationTest {

    @Container
    @ServiceConnection
    static FlociContainer floci = new FlociContainer();

    @Autowired
    private ArchivoService archivoService;

    @Test
    void shouldUploadAndDownloadFile() {
        var clave = archivoService.subirArchivo(
                "productos/pan.jpg",
                new ByteArrayInputStream("fake-image".getBytes()),
                "image/jpeg");

        byte[] contenido = archivoService.descargarArchivo(clave);
        assertThat(contenido).isNotNull();
    }
}
```

### 5.4 Reusar contenedor entre tests

Para evitar levantar un contenedor por cada clase de test, declarar en una clase base:

```java
public abstract class FlociTestBase {

    @Container
    protected static FlociContainer floci = new FlociContainer();

    protected static S3Client s3;

    @BeforeAll
    static void setUpClients() {
        s3 = S3Client.builder()
                .endpointOverride(URI.create(floci.getEndpoint()))
                .region(Region.of(floci.getRegion()))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(floci.getAccessKey(), floci.getSecretKey())))
                .forcePathStyle(true)
                .build();
    }
}

@Testcontainers
class ProductoServiceTest extends FlociTestBase {

    @Test
    void shouldManageProductImages() {
        s3.createBucket(b -> b.bucket("test-productos"));
        // ... usar el client s3 compartido
    }
}
```

---

## 6. Casos de uso en ProductionFood

### 6.1 HU-11/19 — Imagenes de productos y lotes

Cuando se implemente gestion de imagenes de productos:

```java
// Guardar imagen de un producto
String clave = "productos/" + idProducto + "/imagen-principal.jpg";
archivoService.subirArchivo(clave, inputStream, "image/jpeg");

// URL publica para el frontend
// En S3 real: https://bucket.s3.region.amazonaws.com/productos/1/imagen.jpg
// En Floci:  http://localhost:4566/test-bucket/productos/1/imagen.jpg
```

### 6.2 HU-24/25/26 — Reportes PDF

Cuando se generen reportes exportables:

```java
// Generar PDF y subirlo
byte[] pdf = generarReportePedidos(fechaInicio, fechaFin);
String clave = "reportes/pedidos-" + LocalDate.now() + ".pdf";
archivoService.subirArchivo(clave, new ByteArrayInputStream(pdf), "application/pdf");
```

### 6.3 HU-27 — Alertas de stock bajo por SQS

Cuando se necesite notificacion asincrona:

```java
// Cuando el stock baja del minimo
if (stockActual.compareTo(stockMinimo) < 0) {
    var alerta = new AlertaStock(idMateriaPrima, nombreMateriaPrima, stockActual, stockMinimo);
    notificacionService.enviarNotificacion(objectMapper.writeValueAsString(alerta));
}
```

---

## 7. Pruebas con AWS CLI contra Floci

Si necesitas verificar que Floci funciona desde la linea de comandos:

```bash
# Configurar variables
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

# S3 — crear bucket y subir archivo
aws s3 mb s3://productionfood-archivos --endpoint-url $AWS_ENDPOINT_URL
echo "test" | aws s3 cp - s3://productionfood-archivos/test.txt --endpoint-url $AWS_ENDPOINT_URL
aws s3 ls s3://productionfood-archivos --endpoint-url $AWS_ENDPOINT_URL

# SQS — crear cola y enviar mensaje
aws sqs create-queue --queue-name alertas-stock --endpoint-url $AWS_ENDPOINT_URL
aws sqs send-message \
  --queue-url $AWS_ENDPOINT_URL/000000000000/alertas-stock \
  --message-body '{"tipo":"STOCK_BAJO","materia_prima":"Harina"}' \
  --endpoint-url $AWS_ENDPOINT_URL

# Verificar colas
aws sqs list-queues --endpoint-url $AWS_ENDPOINT_URL
```

---

## 8. Persistencia de datos

Floci almacena datos en `/app/data` dentro del contenedor. Para que sobrevivan
reinicios:

### Docker Compose (volumen named)

```yaml
volumes:
  floci-data:
```

### Docker run (volumen named)

```bash
docker run --rm -d \
  -p 4566:4566 \
  -v floci-data:/app/data \
  floci/floci:latest
```

### Docker run (directorio local)

```bash
docker run --rm -d \
  -p 4566:4566 \
  -v $(pwd)/floci-data:/app/data \
  floci/floci:latest
```

> **Para CI/CD:** usar modo `memory` (sin persistencia). Los datos se pierden al
> detener el contenedor, que es exactamente lo que se quiere en pipelines.

---

## 9. Puertos

| Servicio | Puerto | Descripcion |
|---|---|---|
| Floci (AWS) | 4566 | Endpoint principal para todos los servicios AWS |
| MySQL | 3306 | Base de datos (ya existente) |
| phpMyAdmin | 8081 | Panel web de MySQL (ya existente) |
| Spring Boot | 8080 | API REST (ya existente) |
| Angular | 4200 | Frontend (ya existente) |

> **Floci usa el puerto 4566** — el mismo que LocalStack. Si migras de LocalStack,
> solo cambia la imagen y el nombre del servicio.

---

## 10. Troubleshooting

| Sintoma | Causa | Solucion |
|---|---|---|
| `Connection refused` en `localhost:4566` | Floci no esta corriendo | `docker compose up -d floci` |
| `403 Forbidden` al conectar a S3 | Falta `forcePathStyle(true)` en el cliente | Agregar `.forcePathStyle(true)` al builder |
| `Access Denied` en SQS | La cola no existe en Floci | Crear la cola primero con `aws sqs create-queue` |
| Lambda no responde (timeout) | UFW bloquea trafico docker bridge (Linux nativo) | `sudo ufw allow in on docker0` |
| Datos se pierden al reiniciar | No hay volumen persistente | Agregar volumen `floci-data:/app/data` |
| Puerto 4566 en uso | Otro servicio lo usa | Detener el otro servicio o cambiar el puerto en docker-compose |

### Errores comunes de Spring Boot

```properties
# Si ves "Endpoint URL must not be null" significa que la variable no se esta leyendo
# Verificar que application.properties tenga:
app.aws.endpoint-url=http://localhost:4566

# Si ves "software.amazon.awssdk.core.exception.SdkClientException: Unable to execute request"
# Verificar que Floci este corriendo:
docker compose ps floci
```

---

## 11. Buenas practicas

1. **No versionar Floci en docker-compose.yml si aun no se usa.** Agregarlo solo
   cuando una HU lo necesite. Si esta en el compose pero nadie lo usa, gasta
   memoria innecesariamente.

2. **Usar Testcontainers en pruebas de integracion.** No dependas de que Floci
   este corriendo manualmente para que los tests pasen. Testcontainers levanta y
   destruye el contenedor automaticamente.

3. **Credenciales ficticias en desarrollo, reales en produccion.** Las credenciales
   de Floci (`test`/`test`) nunca deben usarse contra AWS real. Usar perfiles
   de AWS CLI o variables de entorno distintas.

4. **`forcePathStyle(true)` siempre.** Es el error mas comun al integrar Floci con
   Spring Boot. Sin esto, el SDK intenta resolver bucket names como subdominios
   que no existen en localhost.

5. **Bucket names sin puntos ni guiones.** Floci es estricto con los nombres de
   buckets. Usar solo minusculas, numeros y guiones largos (`productionfood-archivos`,
   no `ProductionFood_Archivos`).

6. **Un bucket por entorno.** En desarrollo usar `productionfood-dev`, en produccion
   `productionfood-prod`. Nunca reutilizar el mismo bucket.

---

## 12. Referencias

- **Sitio oficial:** https://floci.io
- **GitHub:** https://github.com/floci-io/floci
- **Testcontainers Java:** https://github.com/floci-io/testcontainers-floci
- **Documentacion de servicios:** https://floci.io/floci/services/
- **Migracion desde LocalStack:** https://floci.io/floci/getting-started/migrate-from-localstack/
