# Thymeleaf — Basics with Spring Boot

A beginner-friendly guide to Thymeleaf, the server-side HTML template engine most commonly used with Spring Boot.

> **Related Guides:**
> - [springboot.md](springboot.md) — Spring Boot fundamentals (controllers, config, testing)

---

## Table of Contents

- [What is Thymeleaf?](#what-is-thymeleaf)
- [How It Fits Into Spring Boot](#how-it-fits-into-spring-boot)
- [Setup](#setup)
- [Your First Page](#your-first-page)
- [Expression Types](#expression-types)
- [Common Attributes](#common-attributes)
- [Loops and Conditions](#loops-and-conditions)
- [Links, Images, and Static Files](#links-images-and-static-files)
- [Forms and Validation](#forms-and-validation)
- [Fragments: Reusing Header, Footer, Layout](#fragments-reusing-header-footer-layout)
- [Utility Objects](#utility-objects)
- [Security Notes](#security-notes)
- [Common Pitfalls](#common-pitfalls)

---

## What is Thymeleaf?

Thymeleaf turns an HTML template plus data from your Java code into a finished HTML page **on the server**, then sends that page to the browser.

Its key idea is **natural templates**: templates are valid HTML. You can open them directly in a browser and see a sensible prototype, because Thymeleaf only adds `th:*` attributes that browsers ignore.

```html
<!-- Browser opened directly: shows "John Doe" -->
<!-- Served by Spring:        shows the real user name -->
<p th:text="${user.name}">John Doe</p>
```

| | Thymeleaf (server-side) | React/Angular (client-side) |
|---|---|---|
| Where HTML is built | Server | Browser (JavaScript) |
| Backend returns | Full HTML page | JSON |
| Good for | Admin panels, internal tools, forms, SEO-friendly pages | Highly interactive single-page apps |
| Build tooling | None beyond Maven/Gradle | Node, npm, bundlers |

Thymeleaf replaces older JSP pages in modern Spring apps.

---

## How It Fits Into Spring Boot

```
Browser ──GET /orders──▶ @Controller
                           │ adds data to Model
                           │ returns "orders/list"
                           ▼
                 Thymeleaf renders
                 templates/orders/list.html
                           │
Browser ◀──── HTML page ───┘
```

The controller returns a **view name**. Spring Boot looks for it at:

```
src/main/resources/templates/<view-name>.html
```

> **Important:** Use `@Controller`, not `@RestController`. A `@RestController` would send the string `"orders/list"` as the response body instead of rendering the template.

---

## Setup

Add the starter (Maven):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

Or Gradle:

```kotlin
implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
```

Folder layout:

```
src/main/resources/
├── templates/          # Thymeleaf templates (.html)
│   ├── fragments/
│   │   └── layout.html
│   └── orders/
│       ├── list.html
│       └── form.html
└── static/             # CSS, JS, images, served as-is
    ├── css/app.css
    └── js/app.js
```

Useful properties (defaults shown except `cache`):

```properties
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
# Disable caching in dev so template edits show on refresh (devtools does this automatically)
spring.thymeleaf.cache=false
```

---

## Your First Page

Controller:

```java
@Controller
public class HomeController {

    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("name", "Simran");
        model.addAttribute("items", List.of("Apple", "Banana", "Cherry"));
        return "home";   // → templates/home.html
    }
}
```

Template `templates/home.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>
<body>
    <h1 th:text="'Hello, ' + ${name} + '!'">Hello, Guest!</h1>

    <ul>
        <li th:each="item : ${items}" th:text="${item}">Sample item</li>
    </ul>
</body>
</html>
```

The `xmlns:th` declaration is optional but gives IDE autocompletion.

---

## Expression Types

| Syntax | Name | Use | Example |
|---|---|---|---|
| `${...}` | Variable | Read model data | `${user.email}` |
| `*{...}` | Selection | Field of the object selected with `th:object` | `*{email}` |
| `#{...}` | Message | Text from `messages.properties` (i18n) | `#{welcome.title}` |
| `@{...}` | Link URL | Build URLs with context path and params | `@{/orders/{id}(id=${o.id})}` |
| `~{...}` | Fragment | Reference a reusable template piece | `~{fragments/layout :: header}` |

Handy inline operators:

```html
<!-- String concatenation, or literal substitution with |...| -->
<p th:text="'Total: ' + ${total}"></p>
<p th:text="|Total: ${total}|"></p>

<!-- Ternary and default (Elvis) -->
<span th:text="${active} ? 'Active' : 'Inactive'"></span>
<span th:text="${nickname} ?: 'No nickname'"></span>

<!-- Safe navigation: no error if address is null -->
<span th:text="${user.address?.city}"></span>
```

Inline text without a tag attribute:

```html
<p>Hello, [[${name}]]!</p>
```

---

## Common Attributes

| Attribute | Does | Example |
|---|---|---|
| `th:text` | Set text (HTML-escaped) | `<p th:text="${msg}">` |
| `th:utext` | Set text **unescaped** (renders HTML) | `<div th:utext="${trustedHtml}">` |
| `th:value` | Set input value | `<input th:value="${q}">` |
| `th:href` | Set link URL | `<a th:href="@{/orders}">` |
| `th:src` | Set image/script source | `<img th:src="@{/img/logo.png}">` |
| `th:action` | Set form action | `<form th:action="@{/orders}">` |
| `th:each` | Loop | `<tr th:each="o : ${orders}">` |
| `th:if` / `th:unless` | Show / hide element | `<p th:if="${orders.isEmpty()}">` |
| `th:switch` / `th:case` | Multiple choices | see below |
| `th:classappend` | Add CSS class | `th:classappend="${o.late} ? 'text-danger'"` |
| `th:attr` | Set any attribute | `th:attr="data-id=${o.id}"` |
| `th:object` / `th:field` | Form binding | see [Forms](#forms-and-validation) |
| `th:fragment` / `th:replace` / `th:insert` | Reuse template pieces | see [Fragments](#fragments-reusing-header-footer-layout) |
| `th:block` | Invisible wrapper (no tag output) | `<th:block th:each="...">` |

---

## Loops and Conditions

### Loop over a list

```html
<table>
    <tr>
        <th>#</th><th>Product</th><th>Qty</th>
    </tr>
    <tr th:each="order, stat : ${orders}" th:classappend="${stat.odd} ? 'odd'">
        <td th:text="${stat.count}">1</td>
        <td th:text="${order.product}">Pen</td>
        <td th:text="${order.quantity}">2</td>
    </tr>
</table>
```

The optional second variable (`stat`) gives loop status:

| Property | Meaning |
|---|---|
| `index` | 0-based position |
| `count` | 1-based position |
| `size` | Total items |
| `first` / `last` | Is first / last item |
| `even` / `odd` | Row parity |

### Conditions

```html
<p th:if="${orders.isEmpty()}">No orders yet.</p>
<p th:unless="${orders.isEmpty()}" th:text="|${orders.size()} orders|"></p>

<div th:switch="${order.status}">
    <span th:case="'NEW'">New</span>
    <span th:case="'SHIPPED'">Shipped</span>
    <span th:case="*">Unknown</span>
</div>
```

`th:if` treats `null`, `false`, `0`, `"false"`, `"off"`, and `"no"` as false.

---

## Links, Images, and Static Files

Always build URLs with `@{...}` so the app's context path is added automatically.

```html
<!-- /orders -->
<a th:href="@{/orders}">All orders</a>

<!-- /orders/42 (path variable) -->
<a th:href="@{/orders/{id}(id=${order.id})}">View</a>

<!-- /orders?page=2&size=20 (query params) -->
<a th:href="@{/orders(page=${page + 1}, size=20)}">Next</a>

<!-- Files from src/main/resources/static/ -->
<link rel="stylesheet" th:href="@{/css/app.css}">
<script th:src="@{/js/app.js}"></script>
<img th:src="@{/images/logo.png}" alt="Logo">
```

---

## Forms and Validation

Form DTO (a mutable class works best with `th:field`):

```java
public class OrderForm {
    @NotBlank(message = "Product is required")
    private String product;

    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity = 1;

    // getters and setters
}
```

Controller:

```java
@Controller
@RequestMapping("/orders")
public class OrderPageController {

    private final OrderService orderService;

    public OrderPageController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/new")
    public String showForm(Model model) {
        model.addAttribute("orderForm", new OrderForm());
        return "orders/form";
    }

    @PostMapping
    public String submit(@Valid @ModelAttribute("orderForm") OrderForm form,
                         BindingResult result,
                         RedirectAttributes redirect) {
        if (result.hasErrors()) {
            return "orders/form";        // redisplay with error messages
        }
        orderService.create(form);
        redirect.addFlashAttribute("message", "Order created");
        return "redirect:/orders";       // Post/Redirect/Get: avoids resubmit on refresh
    }
}
```

> **Note:** `BindingResult` must come **immediately after** the `@Valid` parameter, or Spring throws an exception instead of letting you handle errors.

Template `templates/orders/form.html`:

```html
<form th:action="@{/orders}" th:object="${orderForm}" method="post">

    <label for="product">Product</label>
    <input type="text" th:field="*{product}"
           th:classappend="${#fields.hasErrors('product')} ? 'is-invalid'">
    <p th:if="${#fields.hasErrors('product')}" th:errors="*{product}">Product error</p>

    <label for="quantity">Quantity</label>
    <input type="number" th:field="*{quantity}">
    <p th:if="${#fields.hasErrors('quantity')}" th:errors="*{quantity}">Quantity error</p>

    <button type="submit">Save</button>
</form>
```

What `th:field="*{product}"` does for you: sets `id="product"`, `name="product"`, and `value` from the object, so binding works both ways.

Show the flash message after redirect:

```html
<div th:if="${message}" th:text="${message}" class="alert">Saved</div>
```

Dropdown from a list:

```html
<select th:field="*{category}">
    <option th:each="c : ${categories}" th:value="${c.id}" th:text="${c.name}">Category</option>
</select>
```

---

## Fragments: Reusing Header, Footer, Layout

Define fragments once in `templates/fragments/layout.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head th:fragment="head(title)">
    <meta charset="UTF-8">
    <title th:text="${title}">App</title>
    <link rel="stylesheet" th:href="@{/css/app.css}">
</head>
<body>
    <nav th:fragment="navbar">
        <a th:href="@{/}">Home</a>
        <a th:href="@{/orders}">Orders</a>
    </nav>

    <footer th:fragment="footer">
        <p>&copy; 2026 Demo App</p>
    </footer>
</body>
</html>
```

Use them in any page:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Orders')}"></head>
<body>
    <nav th:replace="~{fragments/layout :: navbar}"></nav>

    <main>
        <h1>Orders</h1>
    </main>

    <footer th:replace="~{fragments/layout :: footer}"></footer>
</body>
</html>
```

| Attribute | Result |
|---|---|
| `th:replace` | Host tag is **replaced** by the fragment (most common) |
| `th:insert` | Fragment is placed **inside** the host tag |

> **Tip:** For a single master layout with "content slots", many teams use the separate *Thymeleaf Layout Dialect* library (`layout:decorate`). Plain fragments are enough for most apps.

---

## Utility Objects

Built-in helpers start with `#`:

| Object | Example | Output |
|---|---|---|
| `#strings` | `${#strings.toUpperCase(name)}` | `SIMRAN` |
| `#strings` | `${#strings.abbreviate(desc, 20)}` | Shortened text with `...` |
| `#numbers` | `${#numbers.formatDecimal(price, 1, 2)}` | `12.50` |
| `#temporals` | `${#temporals.format(order.createdAt, 'dd-MM-yyyy')}` | `29-09-2026` (for `java.time` types) |
| `#lists` | `${#lists.isEmpty(orders)}` | `true` / `false` |
| `#fields` | `${#fields.hasErrors('email')}` | Form validation checks |

---

## Security Notes

- **`th:text` escapes HTML** and is safe for user input. **`th:utext` does not**; only use it for content you fully trust, or you open an XSS hole.
- With **Spring Security**, forms using `th:action` automatically get a hidden CSRF token field. Plain `action="..."` does not, and those POSTs will be rejected.
- Don't put secrets in the `Model`; everything in a rendered page can be seen by the user (view source).

---

## Common Pitfalls

| Symptom | Likely cause | Fix |
|---|---|---|
| Page shows the text `home` instead of HTML | `@RestController` used | Use `@Controller` |
| `Error resolving template [x]` | File not at `templates/x.html`, or wrong case/path | Check the path and name (case-sensitive in a JAR) |
| Template edits don't show | Template caching on | `spring.thymeleaf.cache=false` or use devtools |
| `Neither BindingResult nor plain target object for bean name 'orderForm'` | Form object not added to the model | `model.addAttribute("orderForm", new OrderForm())` in the GET handler |
| Validation errors cause a 400 instead of showing on the page | `BindingResult` missing or not right after `@Valid` param | Place `BindingResult` immediately after the form param |
| CSS/JS 404 | File not under `static/`, or hardcoded URL | Put it in `static/` and use `@{/css/...}` |
| POST rejected with 403 | Missing CSRF token with Spring Security | Use `th:action` on the form |
