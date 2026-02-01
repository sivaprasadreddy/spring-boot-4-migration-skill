# HTTP Clients and Interfaces Reference

Spring Boot 4 consolidates HTTP client support around RestClient
(synchronous) and WebClient (reactive), introduces declarative HTTP
interfaces with auto-configuration, and deprecates RestTemplate.

## RestTemplate Deprecation Timeline

| Version | Status |
|---------|--------|
| Spring Framework 7.0 (Nov 2025) | Intent to deprecate announced |
| Spring Framework 7.1 (Nov 2026) | Formally `@Deprecated` with marking for removal |
| Spring Framework 8.0 | Complete removal |
| OSS support | Until at least 2029 |

**Migration is not urgent** but RestClient is the recommended path for
synchronous HTTP in Spring MVC applications.

## RestClient (Recommended for Spring MVC)

### Auto-Configuration

Spring Boot 4 auto-configures `RestClient.Builder` with **prototype
scope** — each injection point gets a fresh clone.

```java
@Service
public class UserService {
    private final RestClient restClient;

    public UserService(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://api.example.com")
            .build();
    }

    public User getUser(Long id) {
        return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .body(User.class);
    }
}
```

Add the starter dependency:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-restclient</artifactId>
</dependency>
```

### Migrating from RestTemplate

```java
// Reuse existing RestTemplate configuration
RestTemplate oldRestTemplate = ...;
RestClient restClient = RestClient.create(oldRestTemplate);
```

| RestTemplate | RestClient |
|-------------|-----------|
| `restTemplate.getForObject(url, Class)` | `restClient.get().uri(url).retrieve().body(Class)` |
| `restTemplate.postForEntity(url, body, Class)` | `restClient.post().uri(url).body(body).retrieve().toEntity(Class)` |
| `restTemplate.exchange(url, method, entity, Class)` | `restClient.method(method).uri(url).body(body).retrieve().toEntity(Class)` |

### Error Handling

```java
restClient.get()
    .uri("/endpoint")
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
        throw new ClientException(response.getStatusCode());
    })
    .body(String.class);
```

## WebClient (Recommended for Reactive/WebFlux)

Add the starter:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webclient</artifactId>
</dependency>
```

WebClient remains the choice for:
- Reactive/non-blocking applications
- High concurrency with streaming
- WebFlux stack
- Backpressure handling

## Declarative HTTP Interfaces (@HttpExchange)

### Defining an Interface

```java
@HttpExchange(url = "https://api.github.com")
public interface RepositoryService {

    @GetExchange("/repos/{owner}/{repo}")
    Repository getRepository(@PathVariable String owner,
                           @PathVariable String repo);

    @PostExchange("/repos/{owner}/{repo}/issues")
    Issue createIssue(@PathVariable String owner,
                     @PathVariable String repo,
                     @RequestBody IssueRequest request);
}
```

Supported annotations: `@GetExchange`, `@PostExchange`, `@PutExchange`,
`@PatchExchange`, `@DeleteExchange`, `@HttpExchange`.

### Boot 3 vs Boot 4 Configuration

**Before (Boot 3 — manual factory):**
```java
@Bean
public RepositoryService repositoryService(RestClient.Builder builder) {
    RestClient restClient = builder.baseUrl("https://api.github.com/").build();
    RestClientAdapter adapter = RestClientAdapter.create(restClient);
    HttpServiceProxyFactory factory = HttpServiceProxyFactory
        .builderFor(adapter).build();
    return factory.createClient(RepositoryService.class);
}
```

**After (Boot 4 — declarative):**
```java
@Configuration
@ImportHttpServices(group = "github", types = RepositoryService.class)
public class ClientConfig { }
```

### Configuration Properties

```yaml
spring:
  http:
    client:
      service:
        connect-timeout: 5s
        read-timeout: 10s
        group:
          github:
            base-url: https://api.github.com
            read-timeout: 5s
          internal:
            base-url: http://internal-api:8080
            connect-timeout: 2s
```

### Java-Based Customization

```java
@Bean
RestClientHttpServiceGroupConfigurer githubConfigurer() {
    return groups -> {
        groups.filterByName("github")
            .forEachClient((group, clientBuilder) -> {
                clientBuilder
                    .defaultHeader("Accept", "application/vnd.github.v3+json")
                    .defaultHeader("Authorization", "Bearer " + token)
                    .defaultStatusHandler(
                        HttpStatusCode::is4xxClientError,
                        (request, response) -> {
                            throw new GitHubApiException(response);
                        }
                    );
            });
    };
}
```

### Multiple Groups

```java
@Configuration
@ImportHttpServices(group = "github", basePackages = "com.example.github")
@ImportHttpServices(group = "internal", types = {OrderService.class, PaymentService.class})
public class ApiClientsConfig { }
```

### WebClient Integration

```java
@Configuration
@ImportHttpServices(
    group = "reactive",
    types = ReactiveUserService.class,
    clientType = ClientType.WebClient
)
public class ReactiveConfig {

    @Bean
    WebClientHttpServiceGroupConfigurer reactiveConfigurer() {
        return groups -> {
            groups.filterByName("reactive")
                .forEachClient((group, builder) -> {
                    builder.baseUrl("https://api.example.com");
                });
        };
    }
}
```

### Reactive Return Types

```java
@HttpExchange
public interface ReactiveUserService {
    @GetExchange("/users")
    Flux<User> getAllUsers();

    @GetExchange("/users/{id}")
    Mono<User> getUserById(@PathVariable String id);

    @PostExchange("/users")
    Mono<User> createUser(@RequestBody User user);
}
```

### @HttpServiceClient — Removed

`@HttpServiceClient` was present in Boot 4 milestones but **removed
before the final release**. Use `@ImportHttpServices` instead.

### OAuth Integration

```java
@HttpExchange(url = "https://api.github.com")
public interface GitHubService {
    @GetExchange("/user")
    @ClientRegistrationId("github")   // Spring Security OAuth
    User getCurrentUser();
}
```

## Connection Pool Configuration

### Properties

```yaml
spring:
  http:
    clients:
      connect-timeout: 2s
      read-timeout: 1s
      redirects: dont-follow
```

### Programmatic

```java
@Bean
RestClient pooledRestClient() {
    PoolingHttpClientConnectionManager cm =
        new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(100);
    cm.setDefaultMaxPerRoute(20);

    CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm).build();

    return RestClient.builder()
        .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
        .build();
}
```

### Virtual Threads

```yaml
spring:
  threads:
    virtual:
      enabled: true   # JDK HttpClient uses virtual threads
```

## Observability

With `spring-boot-starter-opentelemetry`, RestClient and WebClient are
automatically instrumented — no code changes needed. HTTP method, URL,
status code, and timing are captured as traces and metrics.

## Feign/OpenFeign Migration

| Feign | HTTP Interfaces |
|-------|----------------|
| `@FeignClient("service")` | `@HttpExchange` + `@ImportHttpServices(group="service")` |
| `@RequestLine("GET /users")` | `@GetExchange("/users")` |
| `@Param("id")` | `@PathVariable("id")` |
| `@Body` | `@RequestBody` |
| Custom decoder/encoder | `HttpMessageConverter` config |
| Request interceptors | `ClientHttpRequestInterceptor` |
| Error decoder | `defaultStatusHandler()` |
| `feign.client.config.*` | `spring.http.client.service.group.*` |

## Testing HTTP Clients

### MockRestServiceServer

```java
@RestClientTest(UserServiceClient.class)
class UserServiceClientTest {
    @Autowired private UserServiceClient client;
    @Autowired private MockRestServiceServer server;

    @Test
    void testGetUser() {
        server.expect(requestTo("/users/1"))
            .andExpect(method(HttpMethod.GET))
            .andRespond(withSuccess(
                "{\"id\":1,\"name\":\"John\"}",
                MediaType.APPLICATION_JSON));

        User user = client.getUser(1L);
        assertThat(user.getName()).isEqualTo("John");
        server.verify();
    }
}
```

### RestTestClient

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureRestTestClient
class IntegrationTest {
    @Autowired private RestTestClient restTestClient;

    @Test
    void testEndpoint() {
        restTestClient.get()
            .uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> assertThat(user.getName()).isEqualTo("John"));
    }
}
```

### WireMock

```java
@SpringBootTest
@EnableWireMock({
    @ConfigureWireMock(
        name = "github-api",
        property = "spring.http.client.service.group.github.base-url"
    )
})
class RepositoryServiceTest {
    @InjectWireMock("github-api") private WireMockServer githubMock;
    @Autowired private RepositoryService repositoryService;

    @Test
    void shouldGetRepository() {
        githubMock.stubFor(get(urlPathEqualTo("/repos/spring/framework"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("{\"name\":\"framework\"}")));

        Repository repo = repositoryService.getRepository("spring", "framework");
        assertThat(repo.getName()).isEqualTo("framework");
    }
}
```

### TestRestTemplate Replacement

| TestRestTemplate | RestTestClient |
|-----------------|---------------|
| Template-method API | Fluent chainable API |
| `getForEntity()` | `get().uri().exchange().expectStatus()` |
| Manual assertions | Built-in expectations |
| No MockMvc support | Works with MockMvc |

## Migration Checklist

- [ ] Add `spring-boot-starter-restclient` or `spring-boot-starter-webclient`
- [ ] Replace RestTemplate usage with RestClient (use `RestClient.create(restTemplate)` as bridge)
- [ ] Migrate Feign clients to `@HttpExchange` interfaces with `@ImportHttpServices`
- [ ] Configure HTTP service groups via `spring.http.client.service.group.*`
- [ ] Replace `TestRestTemplate` with `RestTestClient` + `@AutoConfigureRestTestClient`
- [ ] Remove manual `HttpServiceProxyFactory` bean definitions
- [ ] Verify auto-instrumentation captures HTTP client traces
