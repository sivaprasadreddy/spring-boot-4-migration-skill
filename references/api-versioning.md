# API Versioning Reference

Spring Framework 7 / Spring Boot 4 introduces first-class, native API
versioning support. This eliminates the need for custom versioning
solutions like `RequestCondition` implementations or manual header parsing.

## Contents

- [Core Feature: `version` Attribute](#core-feature-version-attribute)
- [Versioning Strategies](#versioning-strategies)
- [Default Version and Optional Versioning](#default-version-and-optional-versioning)
- [Java Configuration](#java-configuration)
- [WebFlux Configuration](#webflux-configuration)
- [Semantic Versioning and Ranges](#semantic-versioning-and-ranges)
- [Deprecation Handling (RFC 9745 / RFC 8594)](#deprecation-handling-rfc-9745--rfc-8594)
- [Functional Endpoints](#functional-endpoints)
- [Client-Side Versioning](#client-side-versioning)
- [Error Handling](#error-handling)
- [Testing Versioned APIs](#testing-versioned-apis)
- [Migration from Custom Versioning](#migration-from-custom-versioning)
- [Migration Checklist](#migration-checklist)

## Core Feature: `version` Attribute

All `@RequestMapping` variants support a new `version` attribute:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping(value = "/{id}", version = "1.0")
    public UserV1 getUserV1(@PathVariable Long id) { ... }

    @GetMapping(value = "/{id}", version = "2.0")
    public UserV2 getUserV2(@PathVariable Long id) { ... }
}
```

Works with `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`,
`@PatchMapping`, and `@RequestMapping`.

## Versioning Strategies

Configure ONE strategy via properties or Java config.

### Header-Based Versioning

```yaml
spring:
  mvc:
    apiversion:
      use:
        header: API-Version
```

```bash
curl -H "API-Version: 2.0" http://localhost:8080/api/users/1
```

### URI Path Versioning

```yaml
spring:
  mvc:
    apiversion:
      use:
        path-segment: 1   # 0-based index of version segment
```

```java
@GetMapping(value = "/{version}/users/{id}", version = "1.0")
public UserV1 getUserV1(@PathVariable Long id) { ... }
```

```bash
curl http://localhost:8080/api/v1.0/users/1
```

### Query Parameter Versioning

```yaml
spring:
  mvc:
    apiversion:
      use:
        query-param: version
```

```bash
curl http://localhost:8080/api/users/1?version=2.0
```

### Media Type (Content Negotiation) Versioning

```java
@Configuration
public class VersionConfig implements WebMvcConfigurer {
    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer.useMediaTypeParameter();
    }
}
```

```bash
curl -H "Accept: application/json;version=1.0" http://localhost:8080/api/users
```

## Default Version and Optional Versioning

```yaml
spring:
  mvc:
    apiversion:
      default: v1.0
      use:
        header: API-Version
```

When `default` is set, requests without a version use the default.

## Java Configuration

```java
@Configuration
public class ApiVersionConfig implements WebMvcConfigurer {

    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer
            .useRequestHeader("API-Version")
            .setVersionRequired(false)
            .setDefaultVersion("1.0")
            .setDeprecationHandler(deprecationHandler());
    }

    @Bean
    public ApiVersionDeprecationHandler deprecationHandler() {
        StandardApiVersionDeprecationHandler handler =
            new StandardApiVersionDeprecationHandler();
        handler.addDeprecation("1.0",
            OffsetDateTime.now().minusMonths(6),
            OffsetDateTime.now().plusMonths(6));
        return handler;
    }
}
```

## WebFlux Configuration

```yaml
spring:
  webflux:
    apiversion:
      default: v1.0
      use:
        header: X-API-Version
```

Or Java config:

```java
@Configuration
public class WebFluxVersionConfig implements WebFluxConfigurer {
    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer
            .useRequestHeader("X-API-Version")
            .setDefaultVersion("1.0");
    }
}
```

## Semantic Versioning and Ranges

```java
// Exact version
@GetMapping(version = "1.0")

// Version 1.1 and above
@GetMapping(version = "1.1+")

// Range: 1.0 inclusive to 2.0 exclusive
@GetMapping(version = "[1.0,2.0)")

// Range: 1.0 exclusive to 2.0 inclusive
@GetMapping(version = "(1.0,2.0]")
```

Uses `SemanticApiVersionParser` by default: `major.minor.patch` (minor and
patch default to 0 if omitted).

## Deprecation Handling (RFC 9745 / RFC 8594)

`StandardApiVersionDeprecationHandler` adds response headers when a
deprecated version is requested:

```
Deprecation: Fri, 30 Jun 2025 23:59:59 GMT
Sunset: Sun, 30 Jun 2026 23:59:59 GMT
Link: <https://api.example.com/docs/deprecation>; rel="deprecation"
```

## Functional Endpoints

```java
@Bean
public RouterFunction<ServerResponse> routes(ProductHandler handler) {
    return route()
        .GET("/api/products/{id}", version("1.0"), handler::getProductV1)
        .GET("/api/products/{id}", version("2.0"), handler::getProductV2)
        .build();
}
```

## Client-Side Versioning

### RestClient

```java
RestClient restClient = RestClient.builder()
    .baseUrl("http://localhost:8080")
    .apiVersionInserter(ApiVersionInserter.useHeader("API-Version"))
    .build();

User user = restClient.get()
    .uri("/api/users/{id}", id)
    .apiVersion("2.0")
    .retrieve()
    .body(User.class);
```

### WebClient

```java
WebClient webClient = WebClient.builder()
    .baseUrl("http://localhost:8080")
    .apiVersionInserter(ApiVersionInserter.useHeader("API-Version"))
    .build();

Mono<User> user = webClient.get()
    .uri("/api/users/{id}", id)
    .apiVersion("2.0")
    .retrieve()
    .bodyToMono(User.class);
```

Client-side inserter configuration via properties:

```properties
spring.http.client.restclient.apiversion.insert.header=API-Version
```

## Error Handling

Built-in exceptions:
- `MissingApiVersionException` — version required but not provided (400)
- `InvalidApiVersionException` — version format invalid or unsupported (400)

```java
@RestControllerAdvice
public class ApiVersionExceptionHandler {

    @ExceptionHandler(MissingApiVersionException.class)
    public ResponseEntity<ErrorResponse> handleMissingVersion(
            MissingApiVersionException ex) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("API version is required"));
    }
}
```

## Testing Versioned APIs

### MockMvc

```java
@SpringBootTest
@AutoConfigureMockMvc
class VersionedApiTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testHeaderVersioning() throws Exception {
        mockMvc.perform(get("/api/users/1")
                .header("API-Version", "2.0"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.enhancedField").exists());
    }

    @Test
    void testQueryParamVersioning() throws Exception {
        mockMvc.perform(get("/api/users/1?version=1.0"))
            .andExpect(status().isOk());
    }
}
```

### RestTestClient

```java
@SpringBootTest
@AutoConfigureRestTestClient
class VersionedApiTest {

    @Autowired
    private RestTestClient restTestClient;

    @Test
    void testVersionedEndpoint() {
        restTestClient.get()
            .uri("/api/users/{id}", 1L)
            .header("API-Version", "2.0")
            .exchange()
            .expectStatus().isOk()
            .expectBody(UserV2.class)
            .value(user -> assertThat(user.getEnhancedField()).isNotNull());
    }
}
```

## Migration from Custom Versioning

### Before (Custom Header Checking)

```java
@GetMapping("/api/users")
public ResponseEntity<?> getUsers(@RequestHeader("API-Version") String version) {
    if ("1.0".equals(version)) {
        return ResponseEntity.ok(userService.getUsersV1());
    } else if ("2.0".equals(version)) {
        return ResponseEntity.ok(userService.getUsersV2());
    }
    return ResponseEntity.badRequest().build();
}
```

### After (Native Versioning)

```java
@GetMapping(value = "/api/users", version = "1.0")
public List<UserV1> getUsersV1() { return userService.getUsersV1(); }

@GetMapping(value = "/api/users", version = "2.0")
public List<UserV2> getUsersV2() { return userService.getUsersV2(); }
```

### Before (Separate Controllers per Version)

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }
```

### After (Consolidated)

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping(version = "1.0") public List<UserV1> getUsersV1() { ... }
    @GetMapping(version = "2.0") public List<UserV2> getUsersV2() { ... }
}
```

## Migration Checklist

- [ ] Choose a versioning strategy (header, path, query, media type)
- [ ] Configure `spring.mvc.apiversion.*` or `spring.webflux.apiversion.*`
- [ ] Add `version` attribute to `@GetMapping`/`@PostMapping`/etc.
- [ ] Consolidate versioned endpoints into single controllers
- [ ] Remove custom `RequestCondition` or manual header parsing
- [ ] Update client code to use `ApiVersionInserter`
- [ ] Configure deprecation handler for legacy versions
- [ ] Update tests to use versioning headers/params
