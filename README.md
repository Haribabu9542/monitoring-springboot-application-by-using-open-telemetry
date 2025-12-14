# monitoring-springboot-application-by-using-open-telemetry

[![Build Status](https://img.shields.io/badge/build-pending-lightgrey)](https://github.com/Haribabu9542/monitoring-springboot-application-by-using-open-telemetry/actions) 
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-Enabled-orange)](https://opentelemetry.io/)

An opinionated, hands-on demo that shows how to monitor a Spring Boot application using OpenTelemetry for distributed tracing and Prometheus (via Micrometer) for metrics. The project demonstrates both auto-instrumentation (Java agent) and in-app instrumentation (Micrometer + OpenTelemetry SDK), plus sample Docker Compose stacks with an OTLP collector, Jaeger, Prometheus and Grafana.

This README includes quickstart steps, configuration examples (application.yml), a docker-compose for observability backends, and sample commands to generate traces & metrics.

Table of Contents
- Quickstart (recommended)
- Architecture & Components
- Prerequisites
- Run with Docker Compose (collector + backends)
- Run the Spring Boot app with OpenTelemetry Java Agent
- In-app configuration (application.yml) — Micrometer & OTLP
- Verify traces & metrics
- Helpful tips & troubleshooting
- Development & CI
- Contributing
- License

Quickstart (recommended)
1. Start observability backends:
   - docker-compose up -d
2. Run the Spring Boot app with the OpenTelemetry Java agent:
   - java -javaagent:./opentelemetry-javaagent.jar -jar target/*.jar
3. Generate traffic (curl, browser).
4. View traces in Jaeger and metrics in Prometheus/Grafana.

Architecture & Components
- Spring Boot application instrumented for traces and metrics
  - Traces: exported via OTLP to an OpenTelemetry Collector or directly to Jaeger
  - Metrics: Micrometer (Prometheus registry) exposed at /actuator/prometheus
- OpenTelemetry Collector (receives OTLP, exports to Jaeger / Prometheus)
- Jaeger UI for traces
- Prometheus for scraping metrics
- Grafana for dashboards

Prerequisites
- Java 17+ (or the version used in the project)
- Maven or Gradle (build tool used in the repo)
- Docker & docker-compose (for local observability backends)
- (Optional) curl, hey,wrk for load generation
- Download OpenTelemetry Java Agent (opentelemetry-javaagent.jar) from the official GitHub releases:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases

Run with Docker Compose (collector + backends)
A sample docker-compose is provided in this README and in the repo (if present). It starts:
- otel-collector (OTLP receiver)
- jaeger (traces UI)
- prometheus (for metrics)
- grafana (dashboard)

Create a file docker-compose.yml or use the repo's one. Example snippet:

```yaml
version: "3.8"
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.78.0
    command: ["--config=/conf/otel-collector-config.yml"]
    volumes:
      - ./otel-collector-config.yml:/conf/otel-collector-config.yml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "55681:55681" # OTLP http (if needed)

  jaeger:
    image: jaegertracing/all-in-one:1.41
    ports:
      - "16686:16686" # Jaeger UI
      - "14268:14268"

  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:9.5.3
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

Minimal otel-collector-config.yml (example):

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

exporters:
  jaeger:
    endpoint: "http://jaeger:14268/api/traces"
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
```

Run:
```bash
docker-compose up -d
# Wait a few seconds for services to become healthy
```

Run the Spring Boot app with the OpenTelemetry Java Agent
1. Download the agent jar:
   - curl -L -o opentelemetry-javaagent.jar https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v1.32.0/opentelemetry-javaagent.jar
   (replace version with the current stable release)

2. Start the app with environment variables pointing to the collector:
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_RESOURCE_ATTRIBUTES=service.name=my-springboot-app,service.version=0.1.0 \
java -javaagent:./opentelemetry-javaagent.jar -jar target/monitoring-springboot-app.jar
```

Notes:
- If using OTLP/http, set OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf and endpoint accordingly.
- The Java agent auto-instruments common libraries (Spring MVC, RestTemplate, JDBC, HTTP clients, etc.) and captures traces out-of-the-box.

In-app configuration (application.yml)
If you prefer in-app instrumentation (Micrometer + OpenTelemetry SDK) or to customize spans, add examples like the following.

application.yml (example for Micrometer-prometheus + actuator endpoints):

```yaml
server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,httptrace
  endpoint:
    prometheus:
      enabled: true
  metrics:
    export:
      simple:
        enabled: false

spring:
  application:
    name: monitoring-springboot-app
```

Add dependencies (Maven example):
- Spring Boot starter
- spring-boot-starter-actuator
- micrometer-registry-prometheus
- opentelemetry-sdk (if doing in-app tracing) or rely on the Java agent

Maven snippets:
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- Optional in-app OpenTelemetry SDK -->
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-sdk-extension-autoconfigure</artifactId>
  <version>1.32.0</version>
</dependency>
```

Exposing Prometheus metrics
- With Micrometer + Prometheus registry enabled and actuator, metrics are available at:
  http://localhost:8080/actuator/prometheus
- Configure Prometheus (prometheus.yml) to scrape:
```yaml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['host.docker.internal:8080']  # when running app on host
```

Verify traces & metrics
- Jaeger UI (http://localhost:16686)
  - Search for "my-springboot-app" service and inspect traces.
- Prometheus (http://localhost:9090)
  - Query metrics like jvm_memory_used_bytes, process_cpu_seconds_total, http_server_requests_seconds_count
- Grafana (http://localhost:3000, login admin/admin)
  - Import a Prometheus datasource and use or import example dashboards (Micrometer / JVM dashboards)

Generate test traffic (examples)
- Simple curl:
```bash
curl http://localhost:8080/api/hello
curl http://localhost:8080/api/orders/123
```
- Looped load to generate metrics/traces:
```bash
for i in {1..200}; do curl -s http://localhost:8080/api/hello > /dev/null; done
```

Helpful tips & troubleshooting
- No traces in Jaeger?
  - Ensure OTEL_EXPORTER_OTLP_ENDPOINT is set to the collector and collector is reachable on port 4317.
  - Check Java agent logs (set -Dotel.javaagent.debug=true).
  - Confirm collector config exports to Jaeger.
- No metrics in Prometheus?
  - Ensure /actuator/prometheus is reachable from Prometheus container (networking issues when app runs on host).
  - Use host.docker.internal or run the app as a container in the same compose network.
- Using Spring Cloud Sleuth?
  - Prefer OpenTelemetry and migrate tracing features to OpenTelemetry-based tooling.

Development & CI
- Build:
  - Maven: mvn clean package
  - Gradle: ./gradlew build
- Suggested CI: run build, unit tests, and optionally run containerized observability stack for integration smoke tests (example: GitHub Actions workflow can run docker-compose and run integration tests).

Contributing
- Open issues and PRs are welcome.
- Suggested workflow:
  1. Fork the repo
  2. Create feature branch
  3. Add tests and documentation
  4. Open a PR with a clear description

License
This project is licensed under the MIT License — see the LICENSE file for details.

Contact
Maintainer: Haribabu9542
Repo: https://github.com/Haribabu9542/monitoring-springboot-application-by-using-open-telemetry

Appendix — Useful links
- OpenTelemetry Java instrumentation: https://github.com/open-telemetry/opentelemetry-java-instrumentation
- OpenTelemetry Collector: https://opentelemetry.io/docs/collector/
- Micrometer: https://micrometer.io/
- Jaeger: https://www.jaegertracing.io/
- Prometheus: https://prometheus.io/
- Grafana: https://grafana.com/
