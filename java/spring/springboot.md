# Spring Boot — Simple Guide

A beginner-friendly guide to what Spring Boot is, how it works under the hood, and how to build a clean REST or web application with it.

> **Related Guides:**
> - [thymeleaf.md](thymeleaf.md) — Server-side HTML pages with Thymeleaf
> - [../build/maven/README.md](../build/maven/README.md) — Maven builds
> - [../build/gradle/README.md](../build/gradle/README.md) — Gradle builds
> - [../tomcat/README.md](../tomcat/README.md) — Tomcat, the default embedded server

---

## Table of Contents

- [What is Spring Boot?](#what-is-spring-boot)
- [Spring vs Spring Boot](#spring-vs-spring-boot)
- [Creating a Project](#creating-a-project)
- [Project Structure](#project-structure)
- [The Main Class](#the-main-class)
- [Core Idea: Beans and Dependency Injection](#core-idea-beans-and-dependency-injection)
- [Layered Architecture](#layered-architecture)
- [Building a REST API](#building-a-rest-api)
- [Validation and Error Handling](#validation-and-error-handling)
- [Configuration and Profiles](#configuration-and-profiles)
- [Talking to a Database (Spring Data JPA)](#talking-to-a-database-spring-data-jpa)
- [Logging](#logging)
- [Actuator: Health and Metrics](#actuator-health-and-metrics)
- [Testing](#testing)
- [Running and Packaging](#running-and-packaging)
- [Common Starters](#common-starters)
- [Annotation Cheat Sheet](#annotation-cheat-sheet)
- [Common Pitfalls](#common-pitfalls)

---

## What is Spring Boot?

**Spring** is a large Java framework for building applications. It gives you dependency injection, web handling, data access, security, and more.

**Spring Boot** sits on top of Spring and removes most of the setup work. In simple terms:

> Spring is a box of powerful parts. Spring Boot is the same box, pre-assembled with sensible defaults, so you can start writing business code in minutes.

What Spring Boot gives you:

| Feature | What it means in practice |
|---|---|
| **Starters** | One dependency (e.g. `spring-boot-starter-web`) pulls in everything needed for a feature, with compatible versions |
| **Auto-configuration** | Boot looks at your classpath and configures things for you (e.g. sees a JDBC driver → creates a `DataSource`) |
| **Embedded server** | Tomcat (by default) runs *inside* your app. No WAR deployment; you run a plain `java -jar` |
| **Externalized config** | Settings live in `application.properties`/`application.yml` and can be overridden by env vars |
| **Production features** | Health checks, metrics, and info endpoints via Actuator |

---

## Spring vs Spring Boot

| | Plain Spring | Spring Boot |
|---|---|---|
| Setup | Manual XML or Java config for every component | Auto-configured from classpath |
| Dependencies | Pick and align every version yourself | Starters + managed versions (BOM) |
| Server | Build a WAR, deploy to an external Tomcat/JBoss | Embedded server, run as a JAR |
| Time to "Hello World" | Hours | Minutes |

Spring Boot is **not** a different framework; it still runs Spring. Anything you can do in Spring you can do in Boot.

---

## Creating a Project

The easiest way is **Spring Initializr** at <https://start.spring.io>. Pick:

- Project: Maven or Gradle
- Language: Java
- Java version: the LTS your team uses (Spring Boot 3.x and later require Java 17+)
- Dependencies: e.g. *Spring Web*, *Validation*, *Spring Data JPA*, *Thymeleaf*

From the terminal:

```bash
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d dependencies=web,validation,actuator \
  -d groupId=<com.example> \
  -d artifactId=<demo> \
  -o <demo>.zip
```

IntelliJ IDEA and VS Code (Spring Boot Extension Pack) can also create projects from Initializr directly.

---

## Project Structure

```
demo/
├── mvnw, mvnw.cmd, .mvn/            # Maven Wrapper (use ./mvnw, no local Maven needed)
├── pom.xml                          # Dependencies and build config
└── src/
    ├── main/
    │   ├── java/com/example/demo/
    │   │   ├── DemoApplication.java # Entry point
    │   │   ├── controller/          # HTTP layer
    │   │   ├── service/             # Business logic
    │   │   ├── repository/          # Database access
    │   │   └── dto/                 # Request/response objects
    │   └── resources/
    │       ├── application.properties
    │       ├── static/              # CSS, JS, images (served as-is)
    │       └── templates/           # Thymeleaf HTML templates
    └── test/java/com/example/demo/  # Tests
```

> **Tip:** Keep all your packages *under* the package of the main class. Component scanning starts from there; classes outside it are not found.

---

## The Main Class

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`@SpringBootApplication` is a shortcut for three annotations:

| Annotation | Job |
|---|---|
| `@SpringBootConfiguration` | Marks this as a configuration class |
| `@EnableAutoConfiguration` | Turns on Boot's auto-configuration |
| `@ComponentScan` | Finds your `@Component`, `@Service`, `@Controller`, etc. in this package and below |

What happens at startup:

```
main() → SpringApplication.run()
       → load application.properties / env vars
       → scan packages, create beans, wire dependencies
       → apply auto-configuration
       → start embedded Tomcat on port 8080
       → app is ready
```

---

## Core Idea: Beans and Dependency Injection

A **bean** is just an object that Spring creates and manages for you. Instead of writing `new OrderService(new OrderRepository(...))` yourself, you declare what you need and Spring hands it over. This is **Dependency Injection (DI)**.

Mark a class so Spring manages it:

| Annotation | Use for |
|---|---|
| `@Component` | Generic bean |
| `@Service` | Business logic |
| `@Repository` | Data access (also translates DB exceptions) |
| `@Controller` / `@RestController` | Web layer |
| `@Configuration` + `@Bean` | Creating beans from classes you don't own (e.g. a third-party client) |

**Prefer constructor injection:**

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    // With a single constructor, @Autowired is not needed
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

Why constructor injection over `@Autowired` on fields:

- Fields can be `final` (immutable, thread-safe).
- Dependencies are obvious from the constructor.
- Easy to unit test: just call `new OrderService(mockRepo)`.

Creating a bean for a class you don't own:

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestClient paymentClient(RestClient.Builder builder) {
        return builder.baseUrl("https://<payments-host>").build();
    }
}
```

---

## Layered Architecture

```
HTTP request
    │
    ▼
┌──────────────┐   thin: parse input, validate, return response
│  Controller  │
└──────┬───────┘
       ▼
┌──────────────┐   business rules, transactions
│   Service    │
└──────┬───────┘
       ▼
┌──────────────┐   database queries only
│  Repository  │
└──────┬───────┘
       ▼
    Database
```

| Layer | Should | Should not |
|---|---|---|
| Controller | Map HTTP ↔ DTOs, call services | Contain business logic or queries |
| Service | Hold business rules, use `@Transactional` | Know about HTTP status codes |
| Repository | Query the database | Contain business rules |

Use **DTOs** (Data Transfer Objects) at the API boundary instead of exposing database entities. Java `record`s make great DTOs:

```java
public record CreateOrderRequest(
        @NotBlank String product,
        @Min(1) int quantity) {
}

public record OrderResponse(Long id, String product, int quantity) {
}
```

---

## Building a REST API

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable Long id) {
        return orderService.findById(id);
    }

    @GetMapping
    public List<OrderResponse> search(@RequestParam(defaultValue = "") String product) {
        return orderService.search(product);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse create(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        orderService.delete(id);
    }
}
```

| Annotation | Reads from | Example |
|---|---|---|
| `@PathVariable` | URL path | `/api/orders/42` → `id = 42` |
| `@RequestParam` | Query string | `/api/orders?product=pen` |
| `@RequestBody` | JSON body | `{"product":"pen","quantity":2}` |
| `@RequestHeader` | HTTP header | `@RequestHeader("X-Request-Id")` |

Return objects are converted to JSON automatically (by Jackson). Use `ResponseEntity<T>` when you need full control over status and headers.

> **`@RestController` vs `@Controller`:** `@RestController` returns data (JSON). `@Controller` returns a *view name* that is rendered as HTML (e.g. with Thymeleaf). See [thymeleaf.md](thymeleaf.md).

Try it:

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product":"pen","quantity":2}'
```

---

## Validation and Error Handling

Add `spring-boot-starter-validation`, then annotate DTO fields (`@NotBlank`, `@NotNull`, `@Size`, `@Min`, `@Email`, `@Pattern`) and put `@Valid` on the controller parameter. Invalid input is rejected before your code runs.

Handle errors in **one place** with `@RestControllerAdvice`:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setDetail("Validation failed");
        problem.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .toList());
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Something went wrong");
    }
}
```

`ProblemDetail` produces a standard RFC 9457 error JSON (`type`, `title`, `status`, `detail`).

> **Warning:** Never send stack traces or internal messages to clients in the generic handler. Log them, return a safe message.

---

## Configuration and Profiles

Settings go in `src/main/resources/application.properties` (or `application.yml`):

```properties
server.port=8080
spring.application.name=demo
app.payments.base-url=https://<payments-host>
app.payments.timeout=5s
```

Read them with a type-safe `@ConfigurationProperties` record:

```java
@ConfigurationProperties(prefix = "app.payments")
public record PaymentProperties(String baseUrl, Duration timeout) {
}
```

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class DemoApplication { ... }
```

For a single value, `@Value("${app.payments.base-url}")` also works.

### Profiles

Profiles let you have different settings per environment:

```
application.properties         # shared defaults
application-dev.properties     # used when profile "dev" is active
application-prod.properties    # used when profile "prod" is active
```

Activate one:

```bash
java -jar app.jar --spring.profiles.active=prod
# or
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

### Override order (simplified, highest wins)

1. Command-line arguments (`--server.port=9090`)
2. Environment variables (`SERVER_PORT=9090`)
3. Profile-specific files (`application-prod.properties`)
4. `application.properties`

Environment variable names are the property name in upper case, with `.` replaced by `_` and `-` removed (e.g. `app.payments.base-url` → `APP_PAYMENTS_BASEURL`).

> **Security:** Never commit passwords or API keys to `application.properties`. Inject them via environment variables or a secret manager.

---

## Talking to a Database (Spring Data JPA)

Add `spring-boot-starter-data-jpa` and a driver (e.g. PostgreSQL). Configure the connection:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/<db>
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=validate
```

Entity:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String product;
    private int quantity;
    // getters, setters, no-arg constructor
}
```

Repository: just an interface; Spring writes the implementation:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Query generated from the method name
    List<Order> findByProductContainingIgnoreCase(String product);

    // Pagination for large tables
    Page<Order> findByQuantityGreaterThan(int quantity, Pageable pageable);
}
```

Service with transactions:

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional(readOnly = true)
    public OrderResponse findById(Long id) {
        return orderRepository.findById(id)
                .map(o -> new OrderResponse(o.getId(), o.getProduct(), o.getQuantity()))
                .orElseThrow(() -> new OrderNotFoundException(id));
    }

    @Transactional
    public OrderResponse create(CreateOrderRequest request) {
        Order order = new Order();
        order.setProduct(request.product());
        order.setQuantity(request.quantity());
        Order saved = orderRepository.save(order);
        return new OrderResponse(saved.getId(), saved.getProduct(), saved.getQuantity());
    }
}
```

> **Tips:**
> - Use **Flyway** or **Liquibase** for schema changes; keep `ddl-auto=validate` (or `none`) outside local dev.
> - Watch for the **N+1 problem**: loading a list, then lazily loading a relation for each row. Fix with `JOIN FETCH` or `@EntityGraph`.
> - Use `Pageable` for anything that could return many rows.

---

## Logging

Spring Boot ships with **SLF4J + Logback** configured. Just use it:

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

log.info("Order created id={} product={}", saved.getId(), saved.getProduct());
log.debug("Search params product={}", product);
log.warn("Payment retry attempt={} orderId={}", attempt, orderId);
log.error("Payment failed orderId={}", orderId, ex);
```

Change levels in config:

```properties
logging.level.root=INFO
logging.level.com.example.demo=DEBUG
logging.level.org.hibernate.SQL=DEBUG
```

> **Warning:** Don't use `System.out.println` and never log passwords, tokens, or personal data.

---

## Actuator: Health and Metrics

Add `spring-boot-starter-actuator` to get production endpoints:

| Endpoint | Shows |
|---|---|
| `/actuator/health` | `UP` / `DOWN` (used by load balancers and Kubernetes probes) |
| `/actuator/info` | App info (version, git commit) |
| `/actuator/metrics` | JVM, HTTP, DB pool metrics |
| `/actuator/prometheus` | Metrics for Prometheus (needs `micrometer-registry-prometheus`) |

Only `health` is exposed over HTTP by default. Expose more carefully:

```properties
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.probes.enabled=true
```

> **Security:** Don't expose `env`, `beans`, or `heapdump` publicly; they can leak secrets.

---

## Testing

Add `spring-boot-starter-test` (included by Initializr: JUnit 5, Mockito, AssertJ).

| Type | Annotation | Loads | Use for |
|---|---|---|---|
| Unit test | *(none)* | Nothing; plain `new` | Service business logic (fastest) |
| Web slice | `@WebMvcTest` | Controllers + MVC only | Request mapping, validation, JSON |
| Data slice | `@DataJpaTest` | JPA + repositories | Custom queries |
| Full app | `@SpringBootTest` | Entire context | End-to-end wiring |

Unit test (no Spring):

```java
class OrderServiceTest {

    private final OrderRepository repo = mock(OrderRepository.class);
    private final OrderService service = new OrderService(repo);

    @Test
    void throwsWhenOrderMissing() {
        when(repo.findById(1L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.findById(1L))
                .isInstanceOf(OrderNotFoundException.class);
    }
}
```

Web slice test:

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    OrderService orderService;

    @Test
    void rejectsBlankProduct() throws Exception {
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"product":"","quantity":1}
                                """))
                .andExpect(status().isBadRequest());
    }
}
```

> **Note:** `@MockitoBean` replaced the older `@MockBean` (deprecated since Spring Boot 3.4).

---

## Running and Packaging

| Task | Maven | Gradle |
|---|---|---|
| Run in dev | `./mvnw spring-boot:run` | `./gradlew bootRun` |
| Run tests | `./mvnw test` | `./gradlew test` |
| Build executable JAR | `./mvnw clean package` | `./gradlew bootJar` |
| Build container image | `./mvnw spring-boot:build-image` | `./gradlew bootBuildImage` |

Run the built JAR:

```bash
java -jar target/<demo>-<version>.jar
java -jar target/<demo>-<version>.jar --spring.profiles.active=prod --server.port=9090
```

The JAR is a **"fat JAR"**: it contains your code, all dependencies, and the embedded server. Only a JDK/JRE is needed to run it.

> **Tip:** Add `spring-boot-devtools` for automatic restart when classes change during development. It is disabled automatically in a packaged JAR.

---

## Common Starters

| Starter | Adds |
|---|---|
| `spring-boot-starter-web` | Spring MVC, REST, embedded Tomcat, Jackson |
| `spring-boot-starter-validation` | Jakarta Bean Validation (Hibernate Validator) |
| `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate |
| `spring-boot-starter-thymeleaf` | Thymeleaf templates ([thymeleaf.md](thymeleaf.md)) |
| `spring-boot-starter-security` | Authentication and authorization |
| `spring-boot-starter-actuator` | Health, metrics, info endpoints |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Spring test support |
| `spring-boot-devtools` | Auto-restart and live reload in dev |

You don't specify versions for starters; the Spring Boot parent/BOM manages them.

> **Note:** Spring Boot 4 splits starters into smaller modules (for example, `spring-boot-starter-webmvc` alongside the older `spring-boot-starter-web` name). Check start.spring.io for the names that match your Boot version.

Add OpenAPI/Swagger UI with `springdoc-openapi-starter-webmvc-ui`; the UI is then available at `/swagger-ui.html`.

---

## Annotation Cheat Sheet

| Annotation | Where | Purpose |
|---|---|---|
| `@SpringBootApplication` | Main class | Enable Boot, auto-config, scanning |
| `@RestController` | Class | REST controller returning JSON |
| `@Controller` | Class | MVC controller returning views |
| `@RequestMapping` | Class/method | Base URL path |
| `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` | Method | Map HTTP verbs |
| `@PathVariable`, `@RequestParam`, `@RequestBody` | Parameter | Read request data |
| `@Valid` | Parameter | Trigger validation |
| `@Service`, `@Repository`, `@Component` | Class | Register a bean |
| `@Configuration`, `@Bean` | Class/method | Define beans manually |
| `@Transactional` | Service method | Run in a DB transaction |
| `@ConfigurationProperties` | Record/class | Bind config to an object |
| `@Value` | Field/parameter | Inject a single config value |
| `@Profile("dev")` | Class/method | Only active in a profile |
| `@RestControllerAdvice`, `@ExceptionHandler` | Class/method | Global error handling |

---

## Common Pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| `Port 8080 was already in use` | Another app on the port | Stop it or set `server.port=<port>` |
| `No qualifying bean of type ...` | Class not annotated, or outside the main class's package | Add `@Service`/`@Component`; move under the main package |
| `Failed to configure a DataSource` | JPA/JDBC on classpath but no DB URL | Set `spring.datasource.url`, or remove the dependency |
| `@Transactional` has no effect | Method called from the same class, or not `public` | Call through another bean; make the method public |
| Whitelabel Error Page | No mapping for the URL, or unhandled exception | Check the path; add error handling |
| `LazyInitializationException` | Accessing a lazy relation outside a transaction | Fetch what you need in the service (`JOIN FETCH`, DTO projection) |
| Circular dependency error | Bean A needs B, B needs A | Redesign; extract shared logic into a third bean |
