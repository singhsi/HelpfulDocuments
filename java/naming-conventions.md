# Java Object Naming Conventions

A practical guide to naming the objects in a layered Java application (`LoginForm`, `UserDto`, `UserEntity`, `UserDao`, `UserMapper`, `UserCreatedEvent`, ...) and knowing when and where each one belongs.

> **Related Guides:**
> - [spring/springboot.md](spring/springboot.md) — Layered architecture, DTOs, and JPA in Spring Boot
> - [spring/thymeleaf.md](spring/thymeleaf.md) — Form-backing objects in server-rendered pages

---

## Table of Contents

- [Why Naming Matters](#why-naming-matters)
- [Quick Reference](#quick-reference)
- [How Objects Flow Through an Application](#how-objects-flow-through-an-application)
- [Entity — Database Row](#entity--database-row)
- [DTO — Data Crossing a Boundary](#dto--data-crossing-a-boundary)
- [Request and Response — API DTOs](#request-and-response--api-dtos)
- [Form — HTML Form Backing Object](#form--html-form-backing-object)
- [Repository and DAO — Data Access](#repository-and-dao--data-access)
- [Model — An Overloaded Word](#model--an-overloaded-word)
- [Messaging — Events, Commands, and Messages](#messaging--events-commands-and-messages)
- [Mapper — Converting Between Objects](#mapper--converting-between-objects)
- [Other Common Suffixes](#other-common-suffixes)
- [Package Layout](#package-layout)
- [Choosing a Name: Decision Guide](#choosing-a-name-decision-guide)
- [General Rules](#general-rules)
- [Anti-Patterns](#anti-patterns)

---

## Why Naming Matters

The same "user" shows up in many shapes: a database row, a JSON response, an HTML form, a Kafka message. Each shape has a **different job, different rules, and a different lifetime**. A suffix tells the reader which one they are holding:

- `UserEntity` → managed by JPA, changes may be written to the database.
- `UserResponse` → safe to send to a client, contains no password hash.
- `UserCreatedEvent` → a published fact; other services depend on its shape.

Without the suffix, a single `User` class ends up doing all these jobs, and a new field added for the database quietly leaks into your public API.

> **Tip:** The exact suffix matters less than being **consistent within one codebase**. Pick a convention, write it down, and follow it.

---

## Quick Reference

| Suffix / Pattern | Example | What it is | Where it lives | Typical Java type |
|---|---|---|---|---|
| *(none)* or `Entity` | `User`, `UserEntity` | Maps to a database table | Persistence layer | Class (`@Entity`) |
| `Dto` | `UserDto` | Generic data carrier between layers or systems | Service ↔ controller, service ↔ service | `record` |
| `Request` | `CreateUserRequest` | Incoming API payload | Controller input | `record` |
| `Response` | `UserResponse` | Outgoing API payload | Controller output | `record` |
| `Form` | `LoginForm`, `UserForm` | Backs an HTML form | MVC controller + template | Class (mutable) |
| `Repository` | `UserRepository` | Data access (Spring Data / DDD style) | Persistence layer | Interface |
| `Dao` | `UserDao` | Data access (JDBC / classic style) | Persistence layer | Interface + class |
| `Event` | `UserCreatedEvent` | Something that already happened | Messaging, domain events | `record` |
| `Command` | `CreateUserCommand` | A request to do something | Messaging, CQRS | `record` |
| `Message` | `UserNotificationMessage` | Raw payload on a queue/topic | Messaging adapters | `record` |
| `Mapper` | `UserMapper` | Converts between shapes | Next to the objects it maps | Class or MapStruct interface |
| `Service` | `UserService` | Business logic | Service layer | Class |
| `Controller` | `UserController` | HTTP entry point | Web layer | Class |
| `Client` | `PaymentClient` | Calls an external API | Integration layer | Class or interface |
| `Criteria` / `Filter` | `UserSearchCriteria` | Search parameters | Controller → service → repository | `record` |
| `Summary` / `View` | `UserSummary` | Read-only projection or subset | Query results, list screens | `record` or interface |
| `Properties` / `Config` | `MailProperties`, `SecurityConfig` | Settings / bean configuration | Config package | `record` / class |
| `Exception` | `UserNotFoundException` | An error condition | Anywhere | Class |

---

## How Objects Flow Through an Application

```
            ┌───────────────────────────── Web layer ─────────────────────────────┐
 Browser ──►│ LoginForm / UserForm        (HTML form, Thymeleaf)                  │
 API     ──►│ CreateUserRequest ──► Controller ──► UserResponse ──► JSON          │
            └──────────────┬──────────────────────────────▲───────────────────────┘
                           │ UserMapper                   │ UserMapper
            ┌──────────────▼──────── Service layer ───────┴───────────────────────┐
            │ UserService  (works with UserEntity / domain objects, UserDto)      │
            │      └── publishes ──► UserCreatedEvent ──► Kafka / RabbitMQ / JMS  │
            └──────────────┬──────────────────────────────────────────────────────┘
                           │
            ┌──────────────▼────── Persistence layer ─────────────────────────────┐
            │ UserRepository / UserDao ──► UserEntity ──► Database table `users`  │
            └─────────────────────────────────────────────────────────────────────┘
```

**Rule of thumb:** each layer has its own objects. **Mappers** sit on the borders and convert between them. Entities should not travel past the service layer.

---

## Entity — Database Row

**What:** A class mapped to a database table with JPA/Hibernate (`@Entity`). Its identity is its primary key.

**Naming options:**

| Style | Example | When to use |
|---|---|---|
| Plain noun | `User` | Small apps, or when the entity *is* your domain model. Most Spring/JPA tutorials use this. |
| `Entity` suffix | `UserEntity` | When a `User` already exists elsewhere (domain model, security `User`, generated API model) or you want the persistence role obvious. |

```java
@Entity
@Table(name = "users")
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String passwordHash;   // must never reach an API response

    protected UserEntity() { }     // JPA needs a no-arg constructor
    // getters, setters ...
}
```

**Use it:** in repositories and services.
**Don't:** return it from a controller or put it on a message queue. It may contain sensitive fields and lazy-loaded relations (`LazyInitializationException`), and tying your API to your table layout makes schema changes break clients.

> **Note:** JPA entities cannot be Java `record`s. JPA needs a no-arg constructor, mutable fields, and non-final classes it can proxy.

> **About `UserData` / `UserRecord`:** Some teams use `UserData`, but "data" describes every object, so it tells the reader nothing. `UserRecord` clashes with the Java `record` keyword and with jOOQ, which generates `*Record` classes. Prefer `User` or `UserEntity`.

---

## DTO — Data Crossing a Boundary

**What:** A **Data Transfer Object** is a plain carrier of data with no behavior and no persistence annotations. It exists to move data across a boundary (layer, process, network) in a shape chosen for the *consumer*.

```java
public record UserDto(Long id, String email, String displayName) { }
```

**Use it when:**
- A service returns data to a controller, another service, or a batch job.
- You call or expose an internal API and one shape is shared for reads.
- You want to decouple callers from the entity.

**Tips:**
- Make DTOs **immutable** — Java `record`s are ideal.
- No business logic, no JPA annotations. Validation annotations are fine.
- Prefer **specific names** (`UserResponse`, `CreateUserRequest`) over one giant `UserDto` reused for every case. A single DTO for create, update, and read ends up with fields that are "sometimes null", which is hard to validate.

---

## Request and Response — API DTOs

These are DTOs specialized for the HTTP boundary. Naming them after the **action** makes each endpoint's contract explicit.

| Name | Used for |
|---|---|
| `CreateUserRequest` | `POST /users` body |
| `UpdateUserRequest` | `PUT /users/{id}` body |
| `PatchUserRequest` | `PATCH /users/{id}` body (fields optional) |
| `UserResponse` | Single user returned to the client |
| `UserSummaryResponse` | Lightweight item in a list |
| `PageResponse<T>` | Paginated wrapper |
| `ErrorResponse` | Error body (or use Spring's `ProblemDetail`) |

```java
public record CreateUserRequest(
        @NotBlank @Email String email,
        @NotBlank @Size(min = 8) String password,
        @NotBlank String displayName) { }

public record UserResponse(Long id, String email, String displayName) { }
```

```java
@PostMapping("/users")
public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest request) {
    UserResponse created = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
```

> **Tip:** Notice `CreateUserRequest` has `password` but `UserResponse` does not. Separate input and output types make it impossible to echo a secret back by accident.

---

## Form — HTML Form Backing Object

**What:** An object bound to an HTML form in server-rendered apps (Spring MVC + Thymeleaf, JSP). Spring calls this a *form-backing object* or *command object*.

**Naming:** `<Purpose>Form` — `LoginForm`, `RegistrationForm`, `UserForm`, `ChangePasswordForm`.

```java
public class RegistrationForm {

    @NotBlank @Email
    private String email;

    @NotBlank @Size(min = 8)
    private String password;

    @NotBlank
    private String confirmPassword;

    // no-arg constructor, getters and setters
}
```

```java
@GetMapping("/register")
public String showForm(Model model) {
    model.addAttribute("registrationForm", new RegistrationForm());
    return "register";
}

@PostMapping("/register")
public String submit(@Valid @ModelAttribute RegistrationForm registrationForm,
                     BindingResult result) {
    if (result.hasErrors()) {
        return "register";   // redisplay the form with errors
    }
    userService.register(registrationForm.getEmail(), registrationForm.getPassword());
    return "redirect:/login";
}
```

**Why a separate `Form` instead of a `Request` DTO?**

| | `Form` | `Request` |
|---|---|---|
| Source | HTML `<form>` fields | JSON body |
| Mutability | Usually mutable (the page redisplays it with the user's input and errors) | Immutable `record` |
| UI-only fields | Yes (`confirmPassword`, `rememberMe`, checkbox flags) | No |
| Annotation | `@ModelAttribute` | `@RequestBody` |

> **Note:** Spring can bind forms to records too, but a mutable class is simpler when the form is redisplayed and prefilled. Never bind a form directly to an entity: an attacker can post extra fields (e.g. `role=ADMIN`) and they will be set on the entity ("mass assignment").

See [spring/thymeleaf.md](spring/thymeleaf.md) for the template side.

---

## Repository and DAO — Data Access

Both hide *how* data is stored. The difference is mostly style and history.

| | `UserRepository` | `UserDao` |
|---|---|---|
| Origin | Domain-Driven Design; Spring Data | Classic Java EE / J2EE pattern |
| Mental model | "A collection of users" (`findById`, `save`) | "Operations on a table" (`insert`, `update`, `selectByEmail`) |
| Typical implementation | Spring Data interface — no implementation code | Hand-written with JDBC, `JdbcTemplate`, MyBatis, or `EntityManager` |
| Use in new Spring code | **Default choice** | When writing SQL by hand, or in an existing codebase that uses DAOs |

```java
// Spring Data JPA — Spring generates the implementation
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findByEmail(String email);
}
```

```java
// Hand-written DAO
public interface UserDao {
    Optional<UserEntity> findByEmail(String email);
    void insert(UserEntity user);
}

@Repository
public class JdbcUserDao implements UserDao {
    private final JdbcTemplate jdbc;
    // ...
}
```

> **Tip:** Don't have both a `UserRepository` and a `UserDao` for the same table. Pick one term per codebase. Name implementations after their technology (`JdbcUserDao`, `JpaUserRepository`) rather than `UserDaoImpl` when there could be more than one.

---

## Model — An Overloaded Word

"Model" means different things in different contexts. Avoid `UserModel` unless your team has defined exactly which meaning it has.

| Meaning | Where you see it | Better name |
|---|---|---|
| **Domain model** — rich object with business behavior | DDD, hexagonal architecture | `User` (plain noun) in a `domain` package |
| **Spring MVC `Model`** — the map of attributes passed to a view | `org.springframework.ui.Model` | Name the *attributes*, e.g. `UserView`, `UserForm` |
| **View model** — data shaped for one screen | Thymeleaf pages, dashboards | `UserProfileView`, `DashboardView` |
| **Spring HATEOAS representation** | `RepresentationModel`, `EntityModel<T>` | `UserModel` is the conventional name here |
| **Generated API model** | OpenAPI Generator output | Keep the generated names; map to your own types |
| **Message payload** | Kafka / RabbitMQ / JMS | `UserCreatedEvent`, `...Message` (see next section) |

**If you must use `Model`:** reserve it for one meaning only. For example, "`*Model` = domain objects in the `domain` package" — and never use it for messaging payloads in the same project.

---

## Messaging — Events, Commands, and Messages

Objects sent over Kafka, RabbitMQ, JMS, or Spring's `ApplicationEventPublisher` are **contracts with other systems**. Name them for their intent.

| Kind | Naming | Tense | Example | Meaning |
|---|---|---|---|---|
| **Event** | `<Noun><PastVerb>Event` | Past | `UserCreatedEvent`, `OrderShippedEvent` | A fact: this already happened. Many listeners may react. |
| **Command** | `<Verb><Noun>Command` | Imperative | `SendWelcomeEmailCommand` | A request for one handler to do something. It may be rejected. |
| **Message** | `<Purpose>Message` | — | `EmailNotificationMessage` | Generic payload when it is neither clearly an event nor a command. |
| **Envelope** | `<Name>Envelope` | — | `EventEnvelope<T>` | Wrapper with metadata (id, timestamp, version, correlation id). |

```java
public record UserCreatedEvent(
        UUID eventId,
        Instant occurredAt,
        Long userId,
        String email) { }
```

```java
@Service
public class UserService {
    private final ApplicationEventPublisher events;
    // ...
    public UserResponse create(CreateUserRequest request) {
        UserEntity saved = userRepository.save(userMapper.toEntity(request));
        events.publishEvent(new UserCreatedEvent(UUID.randomUUID(), Instant.now(),
                saved.getId(), saved.getEmail()));
        return userMapper.toResponse(saved);
    }
}
```

**Tips:**
- Keep message types **immutable** and **free of entities**.
- Include an id and timestamp so consumers can deduplicate and order.
- Treat them like a public API: add fields in a backward-compatible way, and version (`UserCreatedEventV2` or a `version` field) when you must break.
- Listener classes: `UserCreatedListener`, `WelcomeEmailHandler`, `OrderEventsConsumer`.

---

## Mapper — Converting Between Objects

**What:** A class whose only job is converting one shape to another (`Request → Entity`, `Entity → Response`, `Entity → Event`).

**Naming:** `<Noun>Mapper` — `UserMapper`, `OrderMapper`. Method names follow the pattern `to<Target>`:

| Method | Converts |
|---|---|
| `toEntity(CreateUserRequest)` | Request → Entity |
| `toResponse(UserEntity)` | Entity → Response |
| `toDto(UserEntity)` | Entity → DTO |
| `toEvent(UserEntity)` | Entity → Event |
| `updateEntity(UpdateUserRequest, UserEntity)` | Copies changes onto an existing entity |

### Option 1: Hand-written mapper (no dependencies)

```java
@Component
public class UserMapper {

    public UserEntity toEntity(CreateUserRequest request) {
        UserEntity user = new UserEntity();
        user.setEmail(request.email());
        user.setDisplayName(request.displayName());
        return user;   // password hashing belongs in the service, not the mapper
    }

    public UserResponse toResponse(UserEntity user) {
        return new UserResponse(user.getId(), user.getEmail(), user.getDisplayName());
    }
}
```

### Option 2: Static factory on the DTO (small apps)

```java
public record UserResponse(Long id, String email, String displayName) {
    public static UserResponse from(UserEntity user) {
        return new UserResponse(user.getId(), user.getEmail(), user.getDisplayName());
    }
}
```

Simple, but it couples the DTO to the entity. Fine for small projects; move to a mapper class as the app grows.

### Option 3: MapStruct (generated at compile time)

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(UserEntity user);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "passwordHash", ignore = true)
    UserEntity toEntity(CreateUserRequest request);
}
```

MapStruct generates plain Java code at build time, so it is fast and errors show up during compilation. It needs the `mapstruct` dependency and annotation processor.

| Approach | Pros | Cons |
|---|---|---|
| Hand-written | No dependencies, fully explicit | Boilerplate |
| Static `from(...)` | Least code | DTO depends on entity |
| MapStruct | Little code, compile-time checks, fast | Extra dependency and build setup |
| ModelMapper / reflection-based | Almost no code | Runtime errors, silent mismatches, slower |

**Keep mappers dumb:** no database calls, no business rules, no password hashing. That belongs in services.

---

## Other Common Suffixes

| Suffix | Example | Purpose |
|---|---|---|
| `Service` | `UserService` | Business logic. Add an interface only when you really have multiple implementations. |
| `Controller` / `RestController` | `UserController` | HTTP endpoints. |
| `Client` | `PaymentClient`, `GitHubClient` | Wraps calls to an external HTTP API. |
| `Criteria` / `Filter` / `Query` | `UserSearchCriteria` | Bundle of search parameters. |
| `Summary` / `Projection` | `UserSummary` | Subset of columns returned from a query (Spring Data supports interface and record projections). |
| `View` | `UserProfileView` | Data shaped for one page or screen. |
| `Properties` | `MailProperties` | `@ConfigurationProperties` binding. |
| `Config` / `Configuration` | `SecurityConfig` | `@Configuration` class that defines beans. |
| `Exception` | `UserNotFoundException` | Error conditions. Name after the problem, not the layer. |
| `Handler` / `Advice` | `GlobalExceptionHandler` | `@RestControllerAdvice` for centralized errors. |
| `Validator` | `PasswordStrengthValidator` | Custom validation logic. |
| `Utils` / `Helper` | `DateUtils` | Stateless helpers. Use sparingly — often a sign the method belongs on another class. |
| `Test` / `IT` | `UserServiceTest`, `UserRepositoryIT` | Unit test / integration test (Maven Failsafe runs `*IT` by default). |

---

## Package Layout

### Package naming rules

| Rule | Example | Notes |
|---|---|---|
| Start with your reversed domain | `com.acme`, `org.example` | Keeps names globally unique (Java Language Specification convention). |
| Then the application or module | `com.acme.shop`, `com.acme.billing` | One root package per deployable app. |
| All lowercase, no separators | `com.acme.shop.useraccount` | No `camelCase`, no hyphens. If a domain has a hyphen (`my-co.com`), use an underscore: `com.my_co`. |
| Short, one-word segments | `dto`, `event`, `mapper` | Prefer `event` over `domainevents`. |
| Singular names | `controller`, `entity`, `mapper` | Matches the JDK style (`java.util`, `java.time`). Plural is also common; just be consistent. |
| No Java keywords | `com.acme.shop.enums`, not `enum` | Keywords such as `enum`, `new`, `class` are invalid package names. |
| Package name ≠ class suffix duplication | `...user.UserService`, not `...userservice.UserService` | The class suffix already states the role. |

> **Spring Boot note:** Put the `@SpringBootApplication` class in the **root package** (e.g. `com.acme.shop`). Component scanning, JPA entity scanning, and repository scanning all start there and include every sub-package. A class placed outside that tree (e.g. `com.acme.common`) will not be picked up without extra configuration.

### Where each object goes

Two common ways to organize packages. The table shows where each object type belongs in both styles (root package `com.acme.shop`):

| Object | Example | By layer | By feature |
|---|---|---|---|
| Controller | `UserController` | `com.acme.shop.controller` | `com.acme.shop.user` |
| Request / Response | `CreateUserRequest`, `UserResponse` | `com.acme.shop.dto` (or `dto.request`, `dto.response`) | `com.acme.shop.user.dto` |
| Form | `LoginForm`, `RegistrationForm` | `com.acme.shop.web.form` | `com.acme.shop.user.form` |
| View model | `UserProfileView` | `com.acme.shop.web.view` | `com.acme.shop.user.view` |
| Service | `UserService` | `com.acme.shop.service` | `com.acme.shop.user` |
| General DTO | `UserDto` | `com.acme.shop.dto` | `com.acme.shop.user.dto` |
| Mapper | `UserMapper` | `com.acme.shop.mapper` | `com.acme.shop.user` |
| Entity | `UserEntity` | `com.acme.shop.entity` (or `domain`, `model`) | `com.acme.shop.user` |
| Repository / DAO | `UserRepository`, `JdbcUserDao` | `com.acme.shop.repository` (or `dao`) | `com.acme.shop.user` |
| Projection | `UserSummary` | `com.acme.shop.repository.projection` | `com.acme.shop.user` |
| Search criteria | `UserSearchCriteria` | `com.acme.shop.dto` | `com.acme.shop.user.dto` |
| Event / Command | `UserCreatedEvent` | `com.acme.shop.event` | `com.acme.shop.user.event` |
| Listener / Consumer | `UserCreatedListener` | `com.acme.shop.messaging` (or `listener`) | `com.acme.shop.user.event` |
| External API client | `PaymentClient` | `com.acme.shop.client` (or `integration`) | `com.acme.shop.payment` or `com.acme.shop.integration.payment` |
| Feature exception | `UserNotFoundException` | `com.acme.shop.exception` | `com.acme.shop.user` |
| Global exception handler | `GlobalExceptionHandler` | `com.acme.shop.exception` | `com.acme.shop.common.exception` |
| Bean configuration | `SecurityConfig` | `com.acme.shop.config` | `com.acme.shop.config` |
| Configuration properties | `MailProperties` | `com.acme.shop.config` | `com.acme.shop.config` (or the feature that uses it) |
| Shared utilities | `DateUtils` | `com.acme.shop.util` | `com.acme.shop.common.util` |

> **Tip:** Cross-cutting classes (`config`, `security`, global exception handling, shared utilities) sit at the root level in **both** styles. Only feature-specific classes move into feature packages.

### By layer

Simple and common in tutorials and small apps. Each package holds one kind of object for **all** features:

```
com.acme.shop
├── ShopApplication.java          # @SpringBootApplication — root package
├── config/       SecurityConfig, MailProperties
├── controller/   UserController, OrderController
├── web/
│   ├── form/     LoginForm, RegistrationForm
│   └── view/     UserProfileView
├── dto/
│   ├── request/  CreateUserRequest, UpdateUserRequest
│   └── response/ UserResponse, OrderResponse
├── service/      UserService, OrderService
├── mapper/       UserMapper, OrderMapper
├── entity/       UserEntity, OrderEntity
├── repository/   UserRepository, OrderRepository
├── event/        UserCreatedEvent, OrderShippedEvent
├── messaging/    UserCreatedListener
├── client/       PaymentClient
├── exception/    UserNotFoundException, GlobalExceptionHandler
└── util/         DateUtils
```

### By feature

Scales better. Everything about one feature sits together, and classes that other features should not use can be **package-private** (no `public` modifier):

```
com.acme.shop
├── ShopApplication.java          # @SpringBootApplication — root package
├── config/           SecurityConfig, MailProperties
├── common/
│   ├── exception/    GlobalExceptionHandler, ErrorResponse
│   └── util/         DateUtils
├── user/
│   ├── UserController.java       # public — HTTP entry point
│   ├── UserService.java          # public — other features may call it
│   ├── UserMapper.java           # package-private
│   ├── UserEntity.java
│   ├── UserRepository.java       # package-private
│   ├── UserNotFoundException.java
│   ├── dto/          CreateUserRequest, UserResponse, UserSearchCriteria
│   ├── form/         RegistrationForm
│   └── event/        UserCreatedEvent, UserCreatedListener
├── order/
│   └── ...
└── integration/
    └── payment/      PaymentClient, PaymentRequest, PaymentResponse
```

> **Note:** Package-private only works *within the same package*. Classes in `user.dto` cannot see package-private classes in `user`, so keep the things you want to hide (mapper, repository) directly in the feature package.

### Which style to choose

| | By layer | By feature |
|---|---|---|
| Best for | Small apps, learning, few features | Growing apps, multiple teams, future microservice split |
| Finding "all controllers" | Easy | Spread across features |
| Finding "everything about users" | Spread across packages | Easy |
| Encapsulation | Everything must be `public` | Package-private hides internals |
| Deleting or extracting a feature | Touch many packages | Move one package |

> **Tip:** Start by layer if the app is small. Switch to by-feature once a single layer package has dozens of unrelated classes. Don't mix both styles for the same kind of object.

---

## Choosing a Name: Decision Guide

```
Is it mapped to a database table (@Entity)?
 └─ yes → User  or  UserEntity

Does it read/write the database?
 └─ yes → UserRepository  (Spring Data)   or  UserDao (hand-written SQL)

Is it bound to an HTML form?
 └─ yes → LoginForm, RegistrationForm

Is it a JSON body for your REST API?
 ├─ incoming → CreateUserRequest, UpdateUserRequest
 └─ outgoing → UserResponse, UserSummaryResponse

Is it sent over a queue/topic or published as an event?
 ├─ something happened  → UserCreatedEvent
 ├─ asks for an action  → SendWelcomeEmailCommand
 └─ neither             → EmailNotificationMessage

Does it convert one shape into another?
 └─ yes → UserMapper (methods: toEntity, toResponse, toDto, toEvent)

Is it a general data carrier between internal layers/services?
 └─ yes → UserDto
```

---

## General Rules

| Rule | Example |
|---|---|
| Classes are `PascalCase` nouns | `UserService`, not `ManageUsers` |
| Treat acronyms as words (Google Java Style) | `UserDto`, `HttpClient`, `XmlParser` — not `UserDTO`, `HTTPClient` |
| Suffix describes the **role**, prefix describes the **thing** | `Order` + `Response` → `OrderResponse` |
| Action-specific DTOs start with the verb | `CreateOrderRequest`, `CancelOrderCommand` |
| Events use past tense | `OrderPlacedEvent`, not `PlaceOrderEvent` |
| Use singular names | `UserEntity` (one row), table can still be `users` |
| One convention per codebase | Don't mix `UserDto` and `UserDTO`, or `Dao` and `Repository` |

---

## Anti-Patterns

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| Returning `@Entity` from a controller | Leaks sensitive fields, lazy-loading errors, API breaks when schema changes | Map to a `Response` DTO |
| Binding a form or request directly to an entity | Mass assignment: attackers set fields like `role` or `id` | Use a `Form` / `Request` with only allowed fields |
| One `UserDto` for create, update, and read | Fields are "sometimes null", validation gets messy | Separate `CreateUserRequest`, `UpdateUserRequest`, `UserResponse` |
| Vague suffixes: `UserData`, `UserInfo`, `UserBean`, `UserObject` | Says nothing about the role | Use the role-specific suffix |
| `UserModel` meaning different things in different packages | Readers can't tell what they're holding | Define `Model` once, or avoid it |
| Business logic inside mappers | Hidden side effects, hard to test | Keep mappers to field copying; logic goes in services |
| `UserServiceImpl` for the only implementation | Interface adds no value | Make `UserService` a class; add an interface when a second implementation appears |
| Entities inside Kafka/JMS messages | Consumers become coupled to your database schema | Publish dedicated `Event` records |
