# Taller de Monitoreo y Observabilidad con Prometheus, Grafana y Jaeger

Este taller tiene como objetivo aprender a **instrumentar una aplicación**, **recolectar sus métricas y trazas** y **visualizarlas** para entender qué está pasando en un sistema que ya está desplegado en contenedores.
Se asume que conoces Docker y Docker Compose. En la **entrega 3 del proyecto del curso** la observabilidad es una **bonificación (+5)**: métricas de la aplicación con un tablero en Grafana ligado al SLO (latencia p95, tasa de error y *throughput*), o logs centralizados con búsqueda.

---

## Integrantes

| Nombre | Correo institucional |
|---|---|
| María José Almanza Caviedes | mariaalmca@unisabana.edu.co |

La documentación del taller (evidencias, análisis y reflexión) está en el [Wiki del repositorio](https://github.com/Majo7-613/DYAS-Monitoreo/wiki).

---

## Objetivo General

Comprender, configurar y aplicar las prácticas de **monitoreo y observabilidad** sobre una aplicación **Java + Spring Boot** desplegada con **Docker Compose**, usando **Spring Boot Actuator y Micrometer** para publicar métricas, **Prometheus** para recolectarlas y evaluar alertas, **Grafana** para construir dashboards, y **OpenTelemetry con Jaeger** para seguir una petición a través de varios servicios, de forma que el equipo pueda detectar y diagnosticar fallas a partir de datos y no de suposiciones.

---

## Índice

- [Conceptos clave](#conceptos-clave)
- [CONOCE EL TALLER](#conoce-el-taller)
  - [Arquitectura](#arquitectura)
  - [Estructura del Proyecto](#estructura-del-proyecto)
  - [Configuración de Dependencias](#configuración-de-dependencias)
  - [Configuración de la observabilidad](#configuración-de-la-observabilidad)
  - [Métricas propias en el código](#métricas-propias-en-el-código)
- [PASO A PASO](#paso-a-paso)
  - [Paso 1: Levantar el stack](#paso-1-levantar-el-stack)
  - [Paso 2: Probar los servicios y generar tráfico](#paso-2-probar-los-servicios-y-generar-tráfico)
  - [Paso 3: Métricas con Prometheus](#paso-3-métricas-con-prometheus)
  - [Paso 4: Dashboards en Grafana](#paso-4-dashboards-en-grafana)
  - [Paso 5: Trazas distribuidas con Jaeger](#paso-5-trazas-distribuidas-con-jaeger)
  - [Paso 6: Provocar errores y latencia, y ver las alertas](#paso-6-provocar-errores-y-latencia-y-ver-las-alertas)
  - [Paso 7: Correlacionar un log con su traza](#paso-7-correlacionar-un-log-con-su-traza)
  - [Paso 8: Apagar el stack](#paso-8-apagar-el-stack)
  - [Retos opcionales](#retos-opcionales)
- [Buenas prácticas](#buenas-prácticas)
- [Para entregar](#para-entregar-con-este-taller)
- [Cómo usar esta guía para tu proyecto](#cómo-usar-esta-guía-para-tu-proyecto)
- [Resumen del Taller](#hagamos-un-resumen)
- [Conclusión](#conclusión)
- [Recursos recomendados](#recursos-recomendados)
- [Créditos y uso académico](#créditos-y-uso-académico)
- [Licencia](#licencia-de-uso)

---

## Conceptos clave

- **Monitoreo**
Recolectar de forma continua datos sobre el estado de un sistema y avisar cuando algo se sale de lo normal.
Ejemplo: una alerta que avisa cuando más del 5% de las peticiones terminan en error.

- **¿Qué es la observabilidad?**
Es la capacidad de entender **qué está pasando dentro de un sistema a partir de los datos que produce**, incluso ante fallas que nadie anticipó. El monitoreo responde preguntas que se conocen de antemano (¿está arriba el servicio?, ¿cuántos errores hay?); la observabilidad permite hacer preguntas nuevas (¿por qué este pedido tardó 3 segundos?, ¿en qué servicio se perdió el tiempo?).

- **Los tres pilares de la observabilidad**

  | Pilar | Qué es | Pregunta que responde | Herramienta en este taller |
  |---|---|---|---|
  | **Métricas** | Valores numéricos agregados en el tiempo (contadores, tiempos, porcentajes). Ocupan poco espacio y sirven para alertar. | ¿Cuántas peticiones por segundo hay? ¿Qué porcentaje falla? | Micrometer + Prometheus + Grafana |
  | **Logs** | Registros de texto de eventos puntuales, con fecha y nivel (INFO, WARN, ERROR). | ¿Qué pasó exactamente en este momento? ¿Qué mensaje de error se produjo? | Logs de Spring Boot (`docker compose logs`) |
  | **Trazas** | El recorrido completo de **una** petición a través de los servicios, con el tiempo que tomó cada paso. | ¿Por qué servicios pasó esta petición y dónde se demoró? | OpenTelemetry + Jaeger |

  Los tres se complementan: la **métrica** avisa que algo anda mal, la **traza** muestra en qué servicio y en qué paso, y el **log** explica el detalle del error.

- **Tipos de métricas**
  - **Counter (contador):** solo aumenta. Ejemplo: `pedidos_total`, número de pedidos procesados.
  - **Gauge (medidor):** sube y baja. Ejemplo: `inventario_stock`, unidades disponibles; o la memoria usada.
  - **Timer / Histogram:** registra duraciones y su distribución en rangos (*buckets*). Ejemplo: `http_server_requests_seconds`, duración de las peticiones HTTP.

- **Etiquetas (labels)**
Pares clave-valor que acompañan a una métrica y permiten filtrarla o agruparla.
Ejemplo: `http_server_requests_seconds_count{application="pedidos", method="POST", uri="/api/pedidos", status="201"}`.

- **Método RED (para servicios)**
Las tres métricas mínimas de un servicio que atiende peticiones: **R**ate (peticiones por segundo), **E**rrors (peticiones que fallan) y **D**uration (cuánto tardan).

- **Método USE (para recursos)**
Para cada recurso (CPU, memoria, disco, conexiones): **U**tilization (qué tan ocupado está), **S**aturation (cuánto trabajo está en espera) y **E**rrors.

- **Percentiles (p50, p95, p99)**
El percentil 95 (p95) es el valor por debajo del cual queda el 95% de las mediciones. Si el p95 de la latencia es 300 ms, el 95% de las peticiones tardó menos de 300 ms y el 5% más lento tardó más.
El **promedio esconde los casos lentos**: 95 peticiones de 10 ms y 5 de 2 segundos dan un promedio de 110 ms, que no describe a ningún usuario real. Por eso los objetivos de servicio (SLO) se expresan con percentiles.

- **Modelo pull (scrape)**
Prometheus **va a buscar** las métricas: cada cierto intervalo hace un `GET` al endpoint de métricas de cada servicio (a esto se le llama *scrape*). La aplicación solo publica sus métricas en `/actuator/prometheus`; no necesita saber dónde está Prometheus.
En el modelo **push** es la aplicación la que envía los datos a un colector. Las trazas de este taller funcionan así: cada servicio las envía a Jaeger.

- **Traza y span**
Un **span** es una operación con inicio y fin (por ejemplo, "atender `POST /api/pedidos`" o "llamar a inventario"). Una **traza** es el árbol de spans de una misma petición, identificada por un **traceId**. Cada span tiene su propio **spanId** y conoce el span que lo originó.

- **Propagación de contexto**
Para que los spans de varios servicios queden en la misma traza, el servicio que llama envía el traceId en el encabezado HTTP `traceparent` (estándar W3C Trace Context). El servicio que recibe lo lee y crea sus spans como hijos.

- **OpenTelemetry y OTLP**
**OpenTelemetry** es el estándar abierto (de la CNCF) para generar métricas, trazas y logs sin atarse a un proveedor. **OTLP** (*OpenTelemetry Protocol*) es su protocolo de envío: por HTTP en el puerto **4318** o por gRPC en el **4317**. Jaeger, Grafana Tempo y la mayoría de los servicios comerciales reciben OTLP.

- **Regla de alerta**
Una consulta que Prometheus evalúa periódicamente. Si la condición se cumple pasa a estado **pending**; si se sigue cumpliendo durante el tiempo indicado en `for`, pasa a **firing** (disparada).

---

## CONOCE EL TALLER

La aplicación es intencionalmente pequeña: una **tienda** con dos servicios.

- **pedidos** recibe los pedidos y, antes de confirmarlos, llama al servicio de inventario para reservar las unidades.
- **inventario** guarda el stock en memoria y tiene un **simulador de fallas** que permite agregar latencia y errores en caliente.

Los dos servicios salen de **la misma imagen Docker**: el perfil de Spring (`SPRING_PROFILES_ACTIVE`) decide cuál se ejecuta. El foco del taller es la **observabilidad**, no la lógica de negocio.

### Arquitectura

```text
              POST /api/pedidos              POST /api/inventario/{producto}/reservas
  k6 / curl ──────────────────► pedidos :8081 ─────────────────────────────────────────► inventario :8082
                                                (con el encabezado traceparent)

  pedidos e inventario ──── envían sus spans por OTLP (HTTP, puerto 4318) ─────────► Jaeger :16686
  Prometheus :9090 ──────── lee GET /actuator/prometheus cada 5 s (scrape) ────────► pedidos e inventario
  Grafana :3000 ─────────── consulta métricas con PromQL ──────────────────────────► Prometheus
  Grafana :3000 ─────────── consulta trazas por traceId ───────────────────────────► Jaeger
```

Las flechas indican quién inicia la comunicación: las **trazas** las envía cada servicio (push) y las **métricas** las va a buscar Prometheus (pull).

| Servicio | Imagen | Puerto | Para qué |
|---|---|---|---|
| `pedidos` | `dyasmonitoreo/tienda:1.0` | 8081 | API de pedidos |
| `inventario` | `dyasmonitoreo/tienda:1.0` (se construye con el `Dockerfile` al levantar este servicio) | 8082 | API de inventario y simulador de fallas |
| `prometheus` | `prom/prometheus:v3.15.0` | 9090 | Recolecta métricas y evalúa alertas |
| `grafana` | `grafana/grafana:13.2.3` | 3000 | Dashboards (usuario `admin`, contraseña `admin`) |
| `jaeger` | `jaegertracing/jaeger:2.21.0` | 16686 (UI), 4317 y 4318 (OTLP) | Recibe y muestra las trazas |
| `k6` | `grafana/k6:2.3.0` | | Generador de tráfico (solo con `docker compose run`) |

### Estructura del Proyecto

```text
.
├─ .github/workflows/ci.yml                        # pipeline: pruebas + cobertura + build de la imagen
├─ Dockerfile                                      # imagen multi-stage, usuario sin privilegios
├─ docker-compose.yaml                             # pedidos, inventario, Prometheus, Grafana, Jaeger y k6
├─ pom.xml                                         # Spring Boot 4.1 + Actuator + Micrometer + OpenTelemetry + JaCoCo
├─ src/main/java/edu/unisabana/dyas/tienda/
│   ├─ TiendaApplication.java
│   ├─ comun/
│   │   ├─ ManejadorErrores.java                   # excepciones -> códigos HTTP; marca los errores 5xx en métricas y trazas
│   │   ├─ TrazasConfig.java                       # no envía a Jaeger las trazas de /actuator
│   │   └─ ...Exception.java                       # excepciones compartidas
│   ├─ pedidos/
│   │   ├─ PedidoController.java                   # POST /api/pedidos, GET /api/pedidos/{id}
│   │   ├─ PedidoService.java                      # caso de uso + métrica propia pedidos_total (Counter)
│   │   ├─ InventarioCliente.java                  # puerto hacia inventario
│   │   └─ InventarioClienteHttp.java              # llamada HTTP con RestClient (propaga la traza)
│   └─ inventario/
│       ├─ InventarioController.java               # GET /api/inventario/{producto}, reservas y /api/fallas
│       ├─ InventarioService.java                  # stock en memoria + métrica propia inventario_stock (Gauge)
│       └─ SimuladorFallas.java                    # latencia y errores configurables en caliente
├─ src/main/resources/
│   ├─ application.properties                      # Actuator, métricas y trazas (común a los dos servicios)
│   ├─ application-pedidos.properties              # puerto 8081, URL de inventario, timeouts
│   └─ application-inventario.properties           # puerto 8082, valores iniciales del simulador
├─ src/test/java/...                               # pruebas unitarias (JUnit 5, MockMvc, MockRestServiceServer)
├─ prometheus/
│   ├─ prometheus.yml                              # qué servicios leer (scrape) y cada cuánto
│   └─ alertas.yml                                 # reglas TasaDeErroresAlta y LatenciaP95Alta
├─ grafana/
│   ├─ provisioning/datasources/datasources.yml    # fuentes de datos Prometheus y Jaeger
│   ├─ provisioning/dashboards/dashboards.yml      # de dónde cargar los dashboards
│   ├─ provisioning/alerting/ y plugins/           # vacías (Grafana las busca al arrancar)
│   └─ dashboards/tienda.json                      # dashboard "Tienda - Observabilidad"
├─ k6/trafico.js                                   # generador de tráfico
└─ img/                                            # capturas usadas en esta guía
```

### Configuración de Dependencias

El [`pom.xml`](pom.xml) usa **Spring Boot 4.1** con **Java 21**:

| Dependencia | Para qué |
|---|---|
| `spring-boot-starter-webmvc` | API REST con Spring MVC. |
| `spring-boot-starter-restclient` | `RestClient.Builder` autoconfigurado. Es el que agrega el encabezado `traceparent` a las llamadas salientes. |
| `spring-boot-starter-actuator` | Endpoints de operación: `/actuator/health` (salud) y `/actuator/prometheus` (métricas). |
| `micrometer-registry-prometheus` | Publica las métricas de Micrometer en el formato de texto que lee Prometheus. |
| `spring-boot-starter-opentelemetry` | Micrometer Tracing con el puente a OpenTelemetry y el exportador OTLP: crea los spans y los envía a Jaeger. |
| `spring-boot-starter-test` | JUnit 5, AssertJ, Mockito, MockMvc y MockRestServiceServer. |
| `jacoco-maven-plugin` | Cobertura con umbral mínimo del 80%, igual que en el taller de Integración Continua. |

> **Micrometer** es a las métricas lo que SLF4J es a los logs: el código usa una sola API (`Counter`, `Gauge`, `Timer`)
> y la dependencia de registro decide a qué sistema se publican (Prometheus, Datadog, OTLP...).

### Configuración de la observabilidad

La instrumentación se activa con dependencias y propiedades en [`application.properties`](src/main/resources/application.properties):

```properties
# Endpoints expuestos por HTTP: salud, información y métricas en formato Prometheus.
management.endpoints.web.exposure.include=health,info,prometheus
management.endpoint.health.probes.enabled=true

# Etiqueta común en todas las métricas: permite filtrar por servicio.
management.metrics.tags.application=${spring.application.name}
# Publica los buckets del histograma de http.server.requests: necesarios para calcular el p95.
management.metrics.distribution.percentiles-histogram.http.server.requests=true
# Prometheus lee las métricas (pull), así que se desactiva el envío de métricas por OTLP (push).
management.otlp.metrics.export.enabled=false

# 1.0 = se registran todas las peticiones. Por defecto Spring Boot solo muestrea el 10%.
management.tracing.sampling.probability=1.0
# Destino OTLP por HTTP. En Docker Compose se reemplaza por http://jaeger:4318/v1/traces.
management.opentelemetry.tracing.export.otlp.endpoint=${OTLP_TRACES_ENDPOINT:http://localhost:4318/v1/traces}
```

| Propiedad | Qué hace |
|---|---|
| `management.endpoints.web.exposure.include` | Por defecto Actuator solo expone `health`. Aquí se agrega `prometheus`. |
| `management.metrics.tags.application` | Agrega la etiqueta `application="pedidos"` o `application="inventario"` a todas las métricas. |
| `percentiles-histogram.http.server.requests` | Sin esta línea solo se publican el conteo, la suma y el máximo; con ella se publican los *buckets* (`_bucket`) que usa `histogram_quantile` para calcular percentiles. |
| `management.tracing.sampling.probability` | Qué fracción de las peticiones se traza. 100% sirve para el taller; en producción con mucho tráfico se usa un valor menor. |
| `management.opentelemetry.tracing.export.otlp.endpoint` | A dónde se envían los spans. |

Los logs no necesitan configuración: mientras hay una traza activa, Spring Boot agrega a cada línea el bloque `[traceId-spanId]`.

Además hay dos clases pequeñas de configuración en el paquete `comun`:

- [`TrazasConfig`](src/main/java/edu/unisabana/dyas/tienda/comun/TrazasConfig.java) evita enviar a Jaeger las trazas de `/actuator`, que Prometheus y Docker consultan cada pocos segundos.
- [`ManejadorErrores`](src/main/java/edu/unisabana/dyas/tienda/comun/ManejadorErrores.java) traduce las excepciones a códigos HTTP y, para los errores 5xx, escribe el log de error y marca la petición como fallida en la métrica y en la traza (ver el paso 7).

### Métricas propias en el código

Las métricas de HTTP y de la JVM las genera Spring Boot. Las métricas **de negocio** las escribe el equipo, porque solo el equipo sabe qué es importante medir.

**Counter** en [`PedidoService`](src/main/java/edu/unisabana/dyas/tienda/pedidos/PedidoService.java): cuenta los pedidos según su resultado (`confirmado`, `sin_stock`, `producto_inexistente`, `error_inventario`).

```java
private void contar(String resultado) {
    Counter.builder("pedidos")
            .description("Pedidos procesados, por resultado")
            .tag("resultado", resultado)
            .register(registry)
            .increment();
}
```

**Gauge** en [`InventarioService`](src/main/java/edu/unisabana/dyas/tienda/inventario/InventarioService.java): publica las unidades disponibles de cada producto.

```java
Gauge.builder("inventario.stock", disponible, AtomicInteger::get)
        .description("Unidades disponibles por producto")
        .tag("producto", producto)
        .register(registry);
```

Micrometer traduce los nombres a la convención de Prometheus: `pedidos` se publica como **`pedidos_total`** (los contadores terminan en `_total`) e `inventario.stock` como **`inventario_stock`**.

La propagación de la traza entre servicios está en [`InventarioClienteHttp`](src/main/java/edu/unisabana/dyas/tienda/pedidos/InventarioClienteHttp.java). El cliente se construye con el `RestClient.Builder` que inyecta Spring Boot:

```java
public InventarioClienteHttp(RestClient.Builder builder, @Value("${inventario.url}") String inventarioUrl) {
    this.restClient = builder.baseUrl(inventarioUrl).build();
}
```

> Si se crea el cliente con `RestClient.create()` en lugar del builder inyectado, la llamada funciona igual,
> pero **no lleva el encabezado `traceparent`** y en Jaeger aparecen dos trazas separadas en lugar de una.

---

## PASO A PASO

### Prerrequisitos

- **Docker Desktop** (o Docker Engine con el plugin Compose) en ejecución.
- Git.
- Opcional, para correr las pruebas en tu máquina: **JDK 21** y **Maven 3.9**.
- Libres en tu máquina los puertos **8081, 8082, 9090, 3000, 16686, 4317 y 4318**.

> **Sobre los comandos:** están escritos para **Git Bash** en Windows o la terminal de macOS/Linux.
> En **PowerShell** hay tres diferencias:
>
> - Escribe `curl.exe` en lugar de `curl` (en Windows PowerShell 5.1, `curl` es un alias de `Invoke-WebRequest` y no acepta las mismas opciones).
> - Dentro del cuerpo JSON las comillas dobles van escapadas con `\`:
>   `curl.exe -s -X POST http://localhost:8081/api/pedidos -H "Content-Type: application/json" -d '{\"producto\":\"teclado\",\"cantidad\":2}'`
> - Usa `Select-String` en lugar de `grep`, por ejemplo `docker compose logs pedidos | Select-String ERROR`.

### Paso 1: Levantar el stack

1. Clona el repositorio y entra a la carpeta:

   ```sh
   git clone https://github.com/<tu-usuario>/DYAS-Monitoreo.git
   cd DYAS-Monitoreo
   ```

2. (Opcional) Corre las pruebas unitarias:

   ```sh
   mvn -B verify
   ```

   Debes ver `Tests run: 47, Failures: 0`, `All coverage checks have been met.` y `BUILD SUCCESS`.

3. Construye la imagen y levanta los cinco contenedores:

   ```sh
   docker compose up -d --build
   ```

   La primera vez tarda unos minutos (descarga las imágenes y las dependencias de Maven).

4. Verifica que todo esté arriba:

   ```sh
   docker compose ps
   ```

   Los servicios `pedidos`, `inventario`, `prometheus` y `grafana` deben aparecer como `(healthy)`; `jaeger` como `Up`
   (la imagen de Jaeger no trae herramientas para definirle un *healthcheck*):

   ```text
   SERVICE      IMAGE                         STATUS
   grafana      grafana/grafana:13.2.3        Up 43 seconds (healthy)
   inventario   dyasmonitoreo/tienda:1.0      Up 53 seconds (healthy)
   jaeger       jaegertracing/jaeger:2.21.0   Up 54 seconds
   pedidos      dyasmonitoreo/tienda:1.0      Up 47 seconds (healthy)
   prometheus   prom/prometheus:v3.15.0       Up 54 seconds (healthy)
   ```

   > El nombre del proyecto está fijado en el `docker-compose.yaml` (`name: dyasmonitoreo`), por eso los contenedores
   > se llaman `dyasmonitoreo-pedidos-1`, `dyasmonitoreo-grafana-1`, etc.

### Paso 2: Probar los servicios y generar tráfico

1. Consulta el stock de un producto directamente en el inventario:

   ```sh
   curl -s http://localhost:8082/api/inventario/teclado
   ```

   ```json
   {"producto":"teclado","stock":50000}
   ```

2. Crea un pedido. El servicio de pedidos llama a inventario para reservar las unidades:

   ```sh
   curl -s -X POST http://localhost:8081/api/pedidos -H "Content-Type: application/json" -d '{"producto":"teclado","cantidad":2}'
   ```

   Respuesta esperada (`201 Created`):

   ```json
   {"id":1,"producto":"teclado","cantidad":2,"fecha":"2026-10-05T17:46:06.109984185Z"}
   ```

3. Prueba los casos de error de negocio. `portatil` no tiene stock (`409 Conflict`) y `tablet` no existe (`404 Not Found`):

   ```sh
   curl -s -X POST http://localhost:8081/api/pedidos -H "Content-Type: application/json" -d '{"producto":"portatil","cantidad":1}'
   curl -s -X POST http://localhost:8081/api/pedidos -H "Content-Type: application/json" -d '{"producto":"tablet","cantidad":1}'
   ```

   ```json
   {"detail":"Stock insuficiente de 'portatil'","instance":"/api/pedidos","status":409,"title":"Conflict"}
   {"detail":"No existe el producto 'tablet'","instance":"/api/pedidos","status":404,"title":"Not Found"}
   ```

   Y consulta el pedido creado:

   ```sh
   curl -s http://localhost:8081/api/pedidos/1
   ```

4. Mira las métricas tal como las publica la aplicación (es lo mismo que lee Prometheus):

   ```sh
   curl -s http://localhost:8081/actuator/prometheus | grep pedidos_total
   ```

   ```text
   # HELP pedidos_total Pedidos procesados, por resultado
   # TYPE pedidos_total counter
   pedidos_total{application="pedidos",resultado="confirmado"} 1.0
   pedidos_total{application="pedidos",resultado="producto_inexistente"} 1.0
   pedidos_total{application="pedidos",resultado="sin_stock"} 1.0
   ```

   Cada línea es una **serie**: el mismo nombre de métrica con una combinación distinta de etiquetas.

5. Genera tráfico continuo con **k6** (el mismo de las pruebas de carga). Déjalo corriendo en una terminal aparte mientras haces los pasos siguientes:

   ```sh
   docker compose run --rm k6
   ```

   Por defecto genera 10 iteraciones por segundo durante 5 minutos (unas 28 peticiones HTTP por segundo, porque
   cada iteración consulta el stock, crea un pedido y lo consulta). Al terminar, k6 imprime un resumen. Se puede cambiar:

   ```sh
   docker compose run --rm -e DURACION=10m -e TASA=20 k6
   ```

   > El servicio `k6` tiene `profiles: ["trafico"]` en el `docker-compose.yaml`: por eso **no** arranca con
   > `docker compose up` y solo corre cuando se invoca con `docker compose run`.

### Paso 3: Métricas con Prometheus

1. Abre <http://localhost:9090/targets>. Los dos servicios de la tienda deben aparecer en estado **UP** dentro del job `tienda`.
   Si alguno aparece **DOWN**, Prometheus no logra leer su `/actuator/prometheus`.

2. Abre <http://localhost:9090/query> y ejecuta las siguientes consultas **PromQL** (pestaña *Table* o *Graph*):

| # | Consulta | Qué muestra |
|---|---|---|
| 1 | `up{job="tienda"}` | 1 si Prometheus logra leer las métricas del servicio, 0 si no. |
| 2 | `http_server_requests_seconds_count{application="pedidos"}` | El contador crudo de peticiones, una serie por combinación de etiquetas. Solo crece. |
| 3 | `sum by (application) (rate(http_server_requests_seconds_count{uri!~"/actuator.*"}[1m]))` | **Rate:** peticiones por segundo de cada servicio, sin contar las de Prometheus a `/actuator`. |
| 4 | `sum by (application, status) (rate(http_server_requests_seconds_count{uri!~"/actuator.*"}[1m]))` | Peticiones por segundo separadas por código HTTP. |
| 5 | `sum by (application) (rate(http_server_requests_seconds_count{uri!~"/actuator.*", status=~"5.."}[1m])) / sum by (application) (rate(http_server_requests_seconds_count{uri!~"/actuator.*"}[1m]))` | **Errors:** fracción de peticiones que terminan en 5xx (0.05 = 5%). |
| 6 | `histogram_quantile(0.95, sum by (application, le) (rate(http_server_requests_seconds_bucket{uri!~"/actuator.*"}[1m])))` | **Duration:** latencia p95 en segundos. |
| 7 | `sum by (resultado) (increase(pedidos_total[5m]))` | Métrica propia: pedidos de los últimos 5 minutos por resultado. |
| 8 | `inventario_stock` | Métrica propia: unidades disponibles por producto. |
| 9 | `sum by (application) (jvm_memory_used_bytes{area="heap"})` | Memoria heap usada por cada JVM, en bytes. |

> Mientras no haya errores 5xx, la consulta 5 responde **"Empty query result"**: no hay ninguna serie con
> `status=~"5.."` que dividir. Es normal; en el paso 6 se provocan errores y aparecerá el resultado.

Cómo leer estas consultas:

- `rate(x[1m])` calcula cuánto creció por segundo un contador en el último minuto. Los contadores **siempre** se consultan con `rate` o `increase`: el valor crudo (consulta 2) depende de cuándo arrancó el servicio y por sí solo no dice nada.
- `sum by (application)` suma todas las series y deja solo la etiqueta `application`.
- `{uri!~"/actuator.*"}` excluye las series cuyo `uri` empieza por `/actuator` (`!~` es "no coincide con la expresión regular").
- `histogram_quantile(0.95, ...)` estima el percentil 95 a partir de los *buckets* (`le` = *less or equal*, el límite superior de cada bucket).

> **Para discutir:** en la consulta 4 aparecen códigos 404 y 409. ¿Deberían contar como errores para la alerta?
> ¿Por qué la consulta 5 solo cuenta los 5xx?

### Paso 4: Dashboards en Grafana

1. Abre <http://localhost:3000>. El dashboard **Tienda - Observabilidad** aparece como página de inicio
   (también está en *Dashboards → Taller DYAS*). Puedes verlo sin iniciar sesión; para editar, entra con `admin` / `admin`.

   ![Dashboard Tienda - Observabilidad en Grafana](img/grafana-dashboard.png)

2. Identifica los paneles y la consulta que hay detrás de cada uno (menú del panel → *Edit*):

   | Fila | Panel | Consulta (resumida) |
   |---|---|---|
   | Tráfico HTTP (método RED) | Peticiones por segundo | consulta 3 del paso anterior |
   | | Tasa de errores 5xx | consulta 5 (con `or ... * 0` para mostrar 0 cuando no hay errores), con una línea roja en el umbral del 5% |
   | | Latencia p95 | consulta 6, con una línea roja en el umbral de 500 ms |
   | | Peticiones por endpoint y código de estado | consulta 4, agregando `method` y `uri` |
   | Métricas de negocio (propias) | Pedidos por resultado (por minuto) | `sum by (resultado) (increase(pedidos_total[1m]))` |
   | | Stock del inventario | `inventario_stock` |
   | JVM (método USE) | Memoria heap usada | consulta 9 |
   | | Uso de CPU del proceso | `process_cpu_usage` |

3. Nada de esto se configuró a mano: Grafana lo **aprovisiona** al arrancar a partir de archivos del repositorio.
   - [`grafana/provisioning/datasources/datasources.yml`](grafana/provisioning/datasources/datasources.yml) crea las fuentes de datos *Prometheus* y *Jaeger*.
   - [`grafana/provisioning/dashboards/dashboards.yml`](grafana/provisioning/dashboards/dashboards.yml) indica la carpeta de la que se cargan los dashboards.
   - [`grafana/dashboards/tienda.json`](grafana/dashboards/tienda.json) es el dashboard.

   Así el dashboard queda **versionado en Git** junto con el código, y cualquier integrante del equipo obtiene el mismo resultado con `docker compose up`.

4. Inicia sesión con `admin` / `admin`, entra en modo edición del dashboard y agrega una visualización nueva con la consulta
   `sum by (uri) (rate(http_server_requests_seconds_count{application="inventario", uri!~"/actuator.*"}[1m]))`.
   Para conservarla, exporta el dashboard como JSON (opción *Export* del dashboard) y reemplaza con ese contenido el archivo
   `grafana/dashboards/tienda.json`. Al recrear el stack (`docker compose down -v` y `docker compose up -d`) el panel debe seguir ahí.

### Paso 5: Trazas distribuidas con Jaeger

1. Abre <http://localhost:16686>. En *Service* selecciona **pedidos**, en *Span Name* selecciona `http post /api/pedidos` y pulsa **Find Traces**.
   Cada resultado indica cuántos spans tiene la traza (3) y por qué servicios pasó (`inventario` y `pedidos`).

2. Abre una de las trazas. Debes ver un árbol de **3 spans** de **los dos servicios**:

   ![Traza de POST /api/pedidos en Jaeger](img/jaeger-traza.png)

   | Span | Servicio | Qué representa |
   |---|---|---|
   | `http post /api/pedidos` | pedidos | La petición que recibió el servicio de pedidos (span raíz). |
   | `http post` | pedidos | La llamada saliente a inventario hecha con `RestClient`. |
   | `http post /api/inventario/{producto}/reservas` | inventario | La petición que llegó a inventario. Es hija de la anterior porque viajó con el encabezado `traceparent`. |

   Haz clic en un span para ver sus etiquetas (`method`, `uri`, `status`, `http.url`...).

3. Compara la duración de cada span. La diferencia entre el span `http post` (la llamada vista desde pedidos) y el span
   de `inventario` (la misma petición vista desde inventario) es el tiempo que se pierde en la red y en el manejo HTTP entre los dos contenedores.

4. En el menú *System Architecture* de Jaeger aparece el grafo `pedidos → inventario`, calculado a partir de las trazas.

> En Jaeger no aparecen trazas de `/actuator/prometheus` ni de `/actuator/health`, aunque Prometheus y Docker los
> consultan cada pocos segundos. [`TrazasConfig`](src/main/java/edu/unisabana/dyas/tienda/comun/TrazasConfig.java)
> filtra esos spans antes de enviarlos; sus métricas sí se siguen publicando.

> **Trazas desde Grafana.** Grafana tiene configurada la fuente de datos Jaeger, pero con Jaeger 2.21 solo permite abrir
> una traza a partir de su `traceId` (en *Explore → Jaeger → TraceID*). La búsqueda de trazas por servicio u operación
> se hace en la interfaz de Jaeger (<http://localhost:16686>), como en este paso.

> **Para discutir:** si un pedido tarda 2 segundos, ¿cómo sabrías **sin trazas** si el tiempo se fue en `pedidos` o en `inventario`?

### Paso 6: Provocar errores y latencia, y ver las alertas

Con k6 todavía generando tráfico:

1. Abre <http://localhost:9090/alerts>. Las reglas `TasaDeErroresAlta` y `LatenciaP95Alta` (definidas en [`prometheus/alertas.yml`](prometheus/alertas.yml)) deben estar en estado **Inactive**.

2. Activa errores en el inventario: el 30% de sus operaciones fallará con `500`.

   ```sh
   curl -s -X POST "http://localhost:8082/api/fallas?tasaError=0.3"
   ```

   ```json
   {"latenciaMs":0,"tasaError":0.3}
   ```

3. Observa la secuencia:
   - En Grafana, el panel **Tasa de errores 5xx** supera la línea roja del 5% para `inventario` y también para `pedidos`, que responde `503` cuando inventario falla.
   - En el panel **Pedidos por resultado** aparece la serie `error_inventario`.
   - En <http://localhost:9090/alerts> la alerta `TasaDeErroresAlta` pasa a **Pending** (unos 20 segundos después,
     cuando la ventana de un minuto ya refleja los errores) y, después de 30 segundos más en esa condición (`for: 30s`), a **Firing**.
     Hay una instancia de la alerta por servicio, porque la regla agrupa por `application`. La anotación `descripcion`
     muestra el valor medido, por ejemplo `30.79% de las peticiones terminan en error 5xx (umbral: 5%).`

   - La consulta 5 del paso 3 ya devuelve resultados. Prueba también esta, que muestra qué excepción produjo cada error:

     ```promql
     sum by (application, status, exception) (rate(http_server_requests_seconds_count{status=~"5.."}[1m]))
     ```

4. Ahora cambia los errores por latencia: cada operación del inventario tardará 800 ms.

   ```sh
   curl -s -X POST "http://localhost:8082/api/fallas?latenciaMs=800"
   ```

   ```json
   {"latenciaMs":800,"tasaError":0.0}
   ```

   Cada llamada a `/api/fallas` reemplaza la configuración completa: al enviar solo `latenciaMs`, la tasa de errores vuelve a 0.

   El panel **Latencia p95** sube por encima de 500 ms y la alerta `LatenciaP95Alta` pasa a **Pending** y luego a **Firing**.
   En Jaeger, las trazas nuevas muestran que el tiempo se va en el span de `inventario`.

5. Regresa a la normalidad:

   ```sh
   curl -s -X DELETE http://localhost:8082/api/fallas
   ```

   ```json
   {"latenciaMs":0,"tasaError":0.0}
   ```

   Después de uno o dos minutos las alertas vuelven a **Inactive** (la ventana `[1m]` de la consulta tarda en "olvidar" los datos anteriores).

> Los valores iniciales del simulador también se pueden fijar al arrancar con las variables `FALLAS_LATENCIA_MS`
> y `FALLAS_TASA_ERROR` del servicio `inventario` en el `docker-compose.yaml`.

> En este taller las alertas solo se ven en la interfaz de Prometheus. En un entorno real Prometheus las envía a
> **Alertmanager**, que se encarga de notificar por correo, Slack o Teams, agrupar alertas repetidas y silenciarlas
> durante un mantenimiento (ver los retos opcionales).

### Paso 7: Correlacionar un log con su traza

1. Con errores activados (`tasaError=0.3`), busca en los logs de pedidos las líneas con nivel `ERROR`:

   ```sh
   docker compose logs pedidos | grep ERROR
   ```

   ```text
   pedidos-1  | 2026-10-05T17:49:05.507Z ERROR 1 --- [pedidos] [nio-8081-exec-8] [1b742b56410a5d2017f63301e04d0478-82551770d2c9750d] e.u.dyas.tienda.comun.ManejadorErrores   : Error al atender la solicitud: El inventario respondió 500
   ```

   El bloque `[traceId-spanId]` lo agrega Spring Boot automáticamente. La primera parte, antes del guion, es el **traceId**
   (`1b742b56410a5d2017f63301e04d0478` en el ejemplo); la segunda es el **spanId**.

2. Copia el traceId y búscalo en los logs del **otro** servicio. Aparece la línea del error original en inventario:

   ```sh
   docker compose logs inventario | grep <traceId>
   ```

   ```text
   inventario-1  | 2026-10-05T17:49:05.503Z ERROR 1 --- [inventario] [nio-8082-exec-8] [1b742b56410a5d2017f63301e04d0478-de126e9e0ad76e48] e.u.dyas.tienda.comun.ManejadorErrores   : Error al atender la solicitud: Falla simulada en el servicio de inventario
   ```

   El traceId es el mismo en los dos servicios; el spanId cambia porque cada servicio registra su propio span.

3. Abre la traza en Jaeger: pega el traceId en el buscador de la parte superior (*Lookup by Trace ID*) o abre
   `http://localhost:16686/trace/<traceId>`. Los tres spans aparecen marcados con un ícono rojo de error y, al abrirlos,
   muestran la excepción (`FallaSimuladaException` en inventario, `InventarioNoDisponibleException` en pedidos):

   ![Traza fallida en Jaeger](img/jaeger-traza-error.png)

   > Los errores se manejan en [`ManejadorErrores`](src/main/java/edu/unisabana/dyas/tienda/comun/ManejadorErrores.java)
   > con `@ExceptionHandler`. Como la excepción no llega hasta Spring, ese manejador la registra a mano en la observación
   > HTTP y en el span actual; sin esas dos líneas, la traza no quedaría marcada con error y la etiqueta `exception`
   > de la métrica diría `none`.

Con esto se recorre el ciclo completo de diagnóstico: la **métrica** avisa (alerta de tasa de errores), el **log** dice qué pasó y en qué petición, y la **traza** muestra el recorrido de esa petición por los dos servicios.

> `docker compose logs` sirve para un equipo con pocos contenedores en una máquina. Con varios servidores se necesita
> **centralizar los logs** en un sistema que los reciba de todos los servicios y permita buscarlos, como Grafana Loki
> o Elasticsearch/OpenSearch. Ver los retos opcionales.

### Paso 8: Apagar el stack

```sh
docker compose down -v
```

`-v` borra también los volúmenes, es decir, los datos de Prometheus y Grafana. Para borrar además la imagen construida:

```sh
docker image rm dyasmonitoreo/tienda:1.0
```

### Retos opcionales

1. **Alertmanager:** agrega el servicio `prom/alertmanager` al `docker-compose.yaml`, configura `alerting:` en `prometheus.yml` y envía las alertas a un *webhook* de prueba.
2. **Centralización de logs con Loki:** agrega Grafana Loki y un agente (Grafana Alloy) que lea los logs de los contenedores; crea la fuente de datos en Grafana y busca los logs por `traceId`.
3. **De la métrica a la traza:** en la fuente de datos Prometheus de Grafana configura *Exemplars* apuntando a la fuente Jaeger, para saltar desde un punto de la gráfica de latencia a una traza concreta.
4. **Nueva métrica de negocio:** agrega un `Timer` o un `DistributionSummary` (por ejemplo, unidades por pedido) y un panel que la muestre.
5. **SLO:** define un objetivo de disponibilidad del 99% para `POST /api/pedidos` y construye un panel con el porcentaje de peticiones exitosas de los últimos 30 minutos.

---

## Buenas prácticas

1. **Instrumentar desde el inicio:** métricas, trazas y logs se configuran con el proyecto, no cuando ya hay un incidente.
2. **RED para cada servicio, USE para cada recurso:** es un buen punto de partida para decidir qué graficar.
3. **Percentiles, no promedios:** las alertas de latencia se definen sobre p95 o p99.
4. **Alertar sobre síntomas que afectan al usuario** (errores, latencia) y no sobre cada causa posible (CPU, memoria); cada alerta debe tener una acción clara.
5. **Usar `for:` en las alertas** para no avisar por picos de pocos segundos.
6. **Cuidar la cardinalidad:** nunca usar como etiqueta valores con muchas variantes (ids de usuario, de pedido, correos). Cada combinación de etiquetas es una serie nueva en Prometheus.
7. **Propagar el contexto:** usar los clientes HTTP autoconfigurados para que las trazas no se corten entre servicios.
8. **Logs con traceId y sin datos sensibles:** nunca registrar contraseñas, tokens ni datos personales.
9. **Configuración como código:** dashboards, fuentes de datos y reglas de alerta versionados en el repositorio.
10. **Proteger los endpoints de operación:** en producción, `/actuator` no se expone a Internet y Grafana no usa `admin`/`admin`.

---

## PARA ENTREGAR CON ESTE TALLER

### 1) Repositorio

- **Fork** de este repositorio con **URL pública o acceso por invitación**.
- Archivo **`integrantes.txt`** o sección en el README con nombres y correos institucionales.
- **Rama principal ejecutable:** `mvn verify` en verde y `docker compose up -d --build` funcionando sin configuraciones manuales adicionales.

### 2) Documentación en Wiki (obligatoria)

> Toda la documentación del taller se entrega en el **Wiki del repositorio**.
> No se requiere PDF; el Wiki es la entrega oficial.

Estructura mínima sugerida del Wiki:

- **Inicio:** integrantes y propósito del taller.
- **Arquitectura de observabilidad:** diagrama propio del stack y explicación, con sus palabras, de qué hace cada componente.
- **Métricas:** las consultas PromQL del paso 3 con su resultado y su interpretación.
- **Dashboard:** captura del dashboard con tráfico y explicación de cada panel.
- **Trazas:** captura de una traza con spans de los dos servicios.
- **Alertas:** evidencia de las alertas en *Pending* y *Firing*.
- **Correlación:** el recorrido métrica → log → traza del paso 7.
- **Reflexión final.**

### 3) Métricas y PromQL

- Captura de <http://localhost:9090/targets> con los dos servicios en **UP**.
- Resultado de las consultas 3, 5, 6 y 7 del paso 3, con una frase que explique cada una.

### 4) Dashboard en Grafana

- Captura del dashboard **Tienda - Observabilidad** con tráfico generado por k6.
- Un **panel nuevo** agregado por el equipo y guardado en `grafana/dashboards/tienda.json` (el panel debe aparecer al levantar el stack desde cero).

### 5) Trazas distribuidas

- Captura de una traza de `POST /api/pedidos` con spans de `pedidos` y de `inventario`.
- Captura de una traza **lenta** (con `latenciaMs`) en la que se vea en qué servicio se fue el tiempo.

### 6) Alertas

- Captura de `TasaDeErroresAlta` en **Firing** y del panel de errores del dashboard en el mismo intervalo.
- Captura de `LatenciaP95Alta` en **Firing**.
- Captura de las alertas de nuevo en **Inactive** después de `DELETE /api/fallas`.

### 7) Correlación de logs y trazas

- Una línea de log con nivel `ERROR` de `pedidos`, la línea correspondiente de `inventario` con el mismo traceId y la traza en Jaeger.

### 8) Reflexión final (en el Wiki)

- ¿Qué diferencia hay entre monitoreo y observabilidad? Den un ejemplo con este taller.
- ¿Por qué la latencia se mide con percentiles y no con el promedio?
- ¿Por qué los errores 404 y 409 no cuentan para la alerta de tasa de errores?
- ¿Qué pasaría con las trazas si `InventarioClienteHttp` usara `RestClient.create()`?
- ¿Qué tendrían que agregar a este stack para obtener la bonificación de observabilidad de la entrega 3 en su proyecto?

### 9) Rúbrica – Taller de Monitoreo y Observabilidad

| **Criterios de evaluación** | **Indicadores de cumplimiento** | **Excelente (5 pts)** | **Bueno (4 pts)** | **Necesita mejorar (3.5 pts)** | **Deficiente (2.5 pts)** | **No cumple (0 pts)** |
|-----------------------------|----------------------------------|------------------------|-------------------|-------------------------------|--------------------------|------------------------|
| **Repositorio y stack** | Fork organizado, con integrantes, pruebas en verde y stack que levanta con un comando. | `mvn verify` en verde y `docker compose up -d --build` deja todos los servicios saludables. | El stack levanta con detalles menores. | Requiere ajustes manuales para levantar. | Algunos servicios no arrancan. | No entrega o el stack no levanta. |
| **Métricas y PromQL** | Targets en UP y consultas RED y de negocio interpretadas. | Targets en UP, consultas 3, 5, 6 y 7 con resultado e interpretación correcta. | Consultas correctas con interpretación superficial. | Solo algunas consultas o sin interpretación. | Consultas con errores o sin datos. | No presenta consultas. |
| **Dashboard en Grafana** | Dashboard con tráfico y un panel nuevo versionado. | Dashboard con datos y panel nuevo guardado en el JSON aprovisionado. | Panel nuevo creado pero no versionado en el repositorio. | Solo captura del dashboard original. | Dashboard sin datos. | No presenta dashboard. |
| **Trazas distribuidas** | Trazas con spans de los dos servicios y análisis de latencia. | Traza normal y traza lenta, con el servicio responsable identificado. | Traza con los dos servicios, sin análisis de la latencia. | Traza de un solo servicio. | Capturas sin relación con el taller. | No presenta trazas. |
| **Alertas** | Ciclo completo Inactive → Pending → Firing → Inactive. | Ambas alertas en Firing y de vuelta en Inactive, con el panel correspondiente. | Una sola alerta con el ciclo completo. | Solo evidencia de la regla sin dispararla. | Reglas modificadas o desactivadas para "pasar". | No presenta alertas. |
| **Correlación de logs y trazas** | Recorrido métrica → log → traza con el mismo traceId. | Log de pedidos, log de inventario y traza con el mismo traceId, bien explicados. | Recorrido completo sin explicación. | Solo log y traza, sin el segundo servicio. | Identificadores que no coinciden. | No realiza el ejercicio. |
| **Documentación en Wiki** | Secciones completas con evidencias y explicación del stack. | Wiki completo, claro y con capturas de todos los pasos. | Wiki completo con leves omisiones. | Wiki incompleto o con poca claridad. | Wiki muy limitado o confuso. | No hay Wiki o está vacío. |
| **Reflexión técnica** | Análisis de resultados y aprendizajes. | Reflexión profunda, conectada con la entrega 3 del proyecto. | Reflexión correcta pero superficial. | Reflexión breve o poco argumentada. | Reflexión vaga o sin relación con el taller. | No presenta reflexión. |

> **Cómo suma**: 8 criterios × 5 pts = **40 puntos**.

| Rango de puntaje | Desempeño                                                |
| ---------------- | -------------------------------------------------------- |
| 36 – 40          | Excelente dominio técnico y metodológico.                |
| 29 – 35          | Buen trabajo con evidencias o documentación parcial.     |
| 24 – 28          | Cumple con lo básico pero sin profundidad.               |
| < 24             | No cumple con los criterios mínimos del taller.          |

---

## Propósito del taller

En este taller se instrumenta una aplicación de dos servicios y se monta, con un solo `docker compose up`, el stack con el que el equipo puede saber si su sistema funciona bien y, cuando no, encontrar la causa.

A través del caso de la tienda (`pedidos` e `inventario`) se conectan las prácticas ya vistas en el curso (**Docker**, **Docker Compose**, **pruebas unitarias**, **integración continua**) con la operación del software en ejecución, que es la parte de **DevOps** que empieza después del despliegue.

---

## Cómo usar esta guía para tu proyecto

1. **Agrega las dependencias** de Actuator, `micrometer-registry-prometheus` y `spring-boot-starter-opentelemetry` a cada servicio (en otros lenguajes: el SDK de OpenTelemetry y un cliente de Prometheus, por ejemplo `prom-client` en Node.js o `prometheus-client` en Python).
2. **Copia las propiedades** de [`application.properties`](src/main/resources/application.properties) y ajusta el nombre de cada servicio.
3. **Usa los clientes HTTP autoconfigurados** (`RestClient.Builder`, `WebClient.Builder`) en las llamadas entre servicios.
4. **Copia las carpetas `prometheus/` y `grafana/`** y los servicios correspondientes del `docker-compose.yaml`; cambia los *targets* de `prometheus.yml` por los de tu proyecto.
5. **Define al menos una métrica de negocio** propia de tu dominio (reservas creadas, pagos rechazados, mensajes procesados...).
6. **Liga el tablero al SLO** definido en el Corte 2: muestra la latencia p95, la tasa de error y el *throughput*, con el umbral del SLO como referencia.
7. **(Opcional) Centraliza los logs** con Loki, Elasticsearch u OpenSearch; la bonificación también se obtiene por esta vía.

Checklist para la bonificación de observabilidad del Proyecto 3 (sección 6 del enunciado):

- [ ] Cada servicio expone `/actuator/prometheus` (o el equivalente en su lenguaje) y aparece en **UP** en Prometheus.
- [ ] Un dashboard aprovisionado desde el repositorio con la **latencia p95**, la **tasa de error** y el ***throughput*** de cada servicio, ligados al SLO.
- [ ] Al menos una **métrica de negocio** propia, graficada.
- [ ] Al menos una **regla de alerta** sobre errores o latencia.
- [ ] Trazas en Jaeger que atraviesan **al menos dos** servicios o componentes.
- [ ] Logs con **traceId** (y, si se elige esa vía, centralizados con búsqueda).

---

> **Resultado esperado:**
> Al finalizar este taller, cada equipo contará con un **stack de observabilidad funcionando** (Prometheus, Grafana y Jaeger) que se levanta con Docker Compose junto a su aplicación, con **métricas RED y de negocio**, **dashboards y alertas versionados en el repositorio** y **trazas distribuidas** correlacionadas con los logs, listo para aplicarse al proyecto del curso como bonificación de observabilidad en la **entrega 3**.

---

## Hagamos un resumen

### Métricas con Prometheus

- **Qué son:** valores numéricos agregados en el tiempo, con etiquetas.
- **Cómo llegan:** la aplicación los publica en `/actuator/prometheus` y Prometheus los lee cada pocos segundos (modelo pull).
- **Cómo se consultan:** con PromQL; los contadores siempre con `rate` o `increase`, la latencia con `histogram_quantile`.

### Dashboards en Grafana

- **Qué son:** paneles que grafican consultas sobre una o varias fuentes de datos.
- **Buena práctica:** aprovisionar fuentes de datos y dashboards desde archivos del repositorio.

### Trazas distribuidas con Jaeger

- **Qué son:** el recorrido de una petición por todos los servicios, dividido en spans.
- **Cómo se conectan:** el encabezado `traceparent` lleva el traceId de un servicio a otro.
- **Cómo llegan a Jaeger:** por OTLP, el protocolo estándar de OpenTelemetry.

### Alertas

- **Qué son:** consultas PromQL que se evalúan periódicamente y pasan de *Inactive* a *Pending* y a *Firing*.
- **Buena práctica:** alertar sobre síntomas (errores, latencia) con un tiempo mínimo (`for`).

### Correlación

- **El traceId** une las tres señales: aparece en los logs y identifica la traza en Jaeger.

---

## Conclusión

En conjunto, estas prácticas permiten:

- Saber en todo momento cuántas peticiones atiende cada servicio, cuántas fallan y cuánto tardan (**métricas y dashboards**).
- Enterarse de un problema antes que los usuarios (**alertas**).
- Encontrar en qué servicio y en qué paso se origina una falla o una demora (**trazas distribuidas**).
- Llegar al detalle de un error concreto a partir de su traceId (**logs correlacionados**).

Con esto se logra **operar el software con datos**: detectar, diagnosticar y verificar la corrección de una falla sin depender de suposiciones, que es lo que exige un sistema desplegado en contenedores y compuesto por varios servicios.

---

## Recursos recomendados

- *Site Reliability Engineering* – Betsy Beyer, Chris Jones, Jennifer Petoff & Niall Richard Murphy (capítulo 6, *Monitoring Distributed Systems*): <https://sre.google/sre-book/monitoring-distributed-systems/>
- *Observability Engineering* – Charity Majors, Liz Fong-Jones & George Miranda
- Tom Wilkie – [The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- Brendan Gregg – [The USE Method](https://www.brendangregg.com/usemethod.html)
- Spring Boot Docs – [Metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html) y [Tracing](https://docs.spring.io/spring-boot/reference/actuator/tracing.html)
- Prometheus Docs – [Querying basics (PromQL)](https://prometheus.io/docs/prometheus/latest/querying/basics/) y [Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- Grafana Docs – [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- Jaeger Docs – [Getting started](https://www.jaegertracing.io/docs/latest/getting-started/)
- OpenTelemetry – [What is OpenTelemetry?](https://opentelemetry.io/docs/what-is-opentelemetry/)
- W3C – [Trace Context](https://www.w3.org/TR/trace-context/)

---

## Créditos y uso académico

**Autor:** César Augusto Vega Fernández
**Curso:** Diseño y Arquitectura de Software
**Programa:** Ingeniería Informática – Universidad de La Sabana
**Año:** 2026

Este taller y su contenido fueron diseñados por el profesor **César Augusto Vega Fernández** como material académico para el curso *Diseño y Arquitectura de Software*, impartido en el programa de **Ingeniería Informática de la Universidad de La Sabana**.

Su propósito es exclusivamente educativo y está orientado a fortalecer las competencias de los estudiantes en **monitoreo, observabilidad, métricas, trazas distribuidas** y prácticas **DevOps** aplicadas a la operación de software.

---

### Licencia de uso

Este material se distribuye bajo la licencia [Creative Commons Atribución-NoComercial-CompartirIgual 4.0 Internacional (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es).

Puedes **usar, adaptar o compartir** este contenido con fines educativos, siempre que:

1. Se reconozca la autoría del profesor **César Augusto Vega Fernández**.
2. No se utilice con fines comerciales.
3. Las obras derivadas se distribuyan bajo la misma licencia.

---

© Universidad de La Sabana – Facultad de Ingeniería
Ingeniería Informática – 2026
