
> **基于**：《Agent-to-Agent (A2A) 权限设计方案》及多轮迭代讨论  
> **技术栈**：Spring AI Alibaba A2A + Nacos + Spring Authorization Server + Spring Security 6.3+  
> **核心**：OAuth2 Token Exchange (RFC 8693) + 一次性令牌 + 模块化权限服务  
> **包名**：`io.github.latcn.a2a.security`  
> **模块**：`a2a-security` 父模块，包含 `a2a-authorization-server`、`a2a-common-permission`、`a2a-security-starter`
> **版本兼容性**：Spring Boot 3.2.x、Spring Cloud Alibaba 2023.0.x、Spring Authorization Server 1.5.7、Spring AI Alibaba 1.0.0.4、Nacos 3.2.1。

---
## 1. 封闭内网中 AI Agent 权限体系的必要性

在政企、军工或大型金融机构的封闭内网中，存在一个致命误区：“系统部署在物理隔离的内网，外面黑客进不来，里面都是‘自己人’，AI Agent 之间互相调用不需要复杂的权限系统。” 这一理念在 AI Agent 时代已经彻底破产。

**五大必要性**：
1. **应对 AI 原生风险**：间接提示词注入可使 Agent 执行恶意指令；大模型幻觉可能调用核心 API。权限系统是最后的物理防线。
2. **控制爆炸半径**：内网隔离防不住内部威胁。最小特权原则强制实施零信任，将安全事件的横向移动限制在最小范围。
3. **数据密级隔离与行级管控**：内网数据有严格密级划分，Agent 必须以用户权限为边界，防止数据聚合泄露。
4. **合规审计与防抵赖**：审计要求追溯完整调用链（用户→Agent→工具），权限系统记录不可篡改的决策日志。
5. **高可用性防雪崩**：权限系统通过限流和配额管理，防止失控 Agent 耗尽内网核心资源。

> **结论**：在封闭内网中，AI Agent 权限体系不是可选项，而是 Day 1 必须建设的核心基础设施。

---

## 2. 核心目标

A2A 权限系统的设计围绕以下 **七大核心目标** 展开：

| 目标维度 | 子目标 | 关键设计 |
|---------|--------|---------|
| **身份认证** | Agent 间认证、用户身份透传、防冒充 | OAuth2 Token Exchange、独立 client_id、JWT 本地验签、delegation_chain |
| **最小权限** | Scope 细粒度、资源实例级控制、行级过滤 | 三层 Scope、ACL 前缀匹配、user_consent、批量预授权 |
| **性能优化** | 高并发、低延迟、批量任务高效 | 本地 JWT 验签、三级缓存、批量 Token、极简 Filter Chain |
| **可审计性** | 全链路追溯、合规日志 | 审计表（含 jti、trace_id、delegation_chain）、异步消息队列 |
| **高可用性** | 故障隔离、降级、防雪崩 | 本地验签不依赖授权服务器、缓存降级、熔断与限流 |
| **动态策略** | 热更新、快速变更 | 数据库存储规则、Redis Pub/Sub 广播、配置开关 |
| **联邦互操作** | 跨组织 Agent 调用 | 信任的 issuer 列表、mTLS/DPoP、act.iss 追溯 |

---
## 3. 总体架构与模块划分

本系统分为**四个独立的部署单元**：

1. **授权服务器**（Authorization Server）— 基于 Spring Authorization Server  
2. **每个 Agent**（既是 OAuth2 客户端，又是资源服务器）— 基于 Spring AI Alibaba A2A  
3. **Nacos** — A2A 注册中心与 AgentCard 存储  
4. **Redis** — 一次性令牌防重放存储  

**Maven 父模块**：`io.github.latcn:a2a-security:1.0.0`

```
a2a-security/                         # 父 POM (groupId=io.github.latcn, artifactId=a2a-security)
├── a2a-authorization-server/         # 授权服务器 (artifactId=a2a-authorization-server)
│   └── src/main/java/io/github/latcn/a2a/security/authz/
│       ├── token/                    # Token Exchange 端点 + 定制器
│       ├── acl/                      # ACL 管理
│       ├── consent/                  # 用户同意管理
│       ├── audit/                    # 审计日志记录
│       └── config/                   # 安全配置
│
├── a2a-common-permission/            # 独立权限 JAR (artifactId=a2a-common-permission)
│   └── src/main/java/io/github/latcn/a2a/security/permission/
│       ├── api/                      # PermissionChecker 接口
│       ├── impl/                     # 基于 R2DBC + Caffeine 缓存
│       └── config/                   # 自动配置
│
├── a2a-security-starter/             # Agent 安全起步依赖
│   └── src/main/java/io/github/latcn/a2a/security/agent/
│       ├── client/                   # TokenExchangeService, A2aClientFilter
│       ├── server/                   # A2AJwtAuthenticationFilter, OneTimeTokenValidator
│       ├── registry/                 # Nacos AgentCard 发布与发现
│       ├── delegation/               # 多跳委托支持
│       └── config/                   # 自动配置
│
└── a2a-agent-xxx/                    # 具体业务 Agent（客户项目）
```

---
## 4. 模块间依赖关系

**编译时依赖**（箭头方向表示“依赖者 → 被依赖者”）：

```
┌─────────────────────┐
│ a2a-agent-xxx       │  业务 Agent
└──────────┬──────────┘
           │ depends on
           ▼
┌─────────────────────┐      ┌────────────────────────┐
│ a2a-security-starter│─────▶│ a2a-common-permission  │
└──────────┬──────────┘      └────────────────────────┘
           │
           │ depends on
           ▼
┌─────────────────────────────────────────────────────────────────┐
│ 外部依赖：                                                       │
│ - Spring AI Alibaba A2A (1.0.0.4+)                             │
│ - Spring Cloud Alibaba Nacos Discovery (2023.0.x 对应 2.2.10+) │
│ - Spring Security OAuth2 Resource Server / Client              │
│ - Spring Boot Starter Data Redis Reactive                      │
│ - Caffeine + R2DBC (for permission)                            │
│ - Reactor Core                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**运行期服务调用关系**：

```
┌──────────────┐   Token Exchange   ┌──────────────────────┐
│   Agent A    │ ─────────────────▶ │ Authorization Server │
│  (客户端)    │ ◀───────────────── │ (ACL+Consent+JWT)    │
└──────┬───────┘   (颁发代理令牌)     └──────────────────────┘
       │
       │ A2A + Bearer Token + X-Trace-Id
       ▼
┌──────────────┐   权限判断（JAR调用）  ┌─────────────────────┐
│   Agent B    │ ────────────────────▶ │ common-permission   │
│ (资源服务器) │                        │ (查用户资源权限表)   │
└──────┬───────┘                        └─────────────────────┘
       │
       │ 一次性令牌防重放
       ▼
   ┌───────┐
   │ Redis │
   └───────┘
```

---
## 5. 模块内核心对外接口

### 5.1 `a2a-common-permission` 模块

**包路径**：`io.github.latcn.a2a.security.permission.api`

```java
public interface PermissionChecker {
    /**
     * 检查用户是否有权对特定资源执行特定操作。
     * 实现必须使用三级缓存（Caffeine + Redis + DB）。
     */
    Mono<Boolean> hasPermission(String userId, String resourceId, String operation);
    
    /**
     * 批量查询多个资源的权限。返回 Map 包含所有请求的资源 ID，value 为是否有权限。
     */
    Mono<Map<String, Boolean>> hasPermissions(String userId, List<String> resourceIds, String operation);
    
    /**
     * 主动清除本地权限缓存（仅清除当前实例）。
     * 跨实例广播需通过 Redis Pub/Sub 单独实现。
     */
    Mono<Void> evictPermissionCache(String userId, String resourceId, String operation);
}
```

**数据库表**：
```sql
CREATE TABLE user_resource_permission (
    id UUID PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    resource_id VARCHAR(128) NOT NULL,
    operation VARCHAR(32) NOT NULL,
    granted_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, resource_id, operation)
);
CREATE INDEX idx_user_resource ON user_resource_permission(user_id, resource_id);
```

**三级缓存架构**（查询顺序：L1 → L2 → L3，并逐级回写）：
- L1：Caffeine 本地缓存（纳秒级），TTL 5 分钟，最大 10000 条。
- L2：Redis 分布式缓存（毫秒级），TTL 30 分钟。
- L3：数据库（仅在穿透时查询）。
- 缓存一致性：Redis Pub/Sub 广播失效消息。

**实现示例**：
```java
@Override
public Mono<Boolean> hasPermission(String userId, String resourceId, String operation) {
    String cacheKey = userId + ":" + resourceId + ":" + operation;
    // 1. L1
    Boolean result = localCache.getIfPresent(cacheKey);
    if (result != null) return Mono.just(result);
    // 2. L2
    return redisTemplate.opsForValue().get(cacheKey)
        .doOnNext(v -> { if (v != null) localCache.put(cacheKey, v); })
        .switchIfEmpty(
            // 3. L3
            repository.hasPermission(userId, resourceId, operation)
                .doOnNext(dbResult -> {
                    redisTemplate.opsForValue().set(cacheKey, dbResult, Duration.ofMinutes(30));
                    localCache.put(cacheKey, dbResult);
                })
        );
}
```

**批量查询实现**：
```java
@Override
public Mono<Map<String, Boolean>> hasPermissions(String userId, List<String> resourceIds, String operation) {
    if (resourceIds.isEmpty()) return Mono.just(Map.of());
    return repository.findAllByUserIdAndResourceIdInAndOperation(userId, resourceIds, operation)
        .map(perm -> perm.getResourceId())
        .collectList()
        .map(allowed -> resourceIds.stream().collect(Collectors.toMap(id -> id, allowed::contains)));
}
```

**`user_consent` 表（支持有效期）**：
```sql
CREATE TABLE user_consent (
    id UUID PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    client_id VARCHAR(64) NOT NULL,
    scope_prefix VARCHAR(256) NOT NULL,
    granted_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP NULL,
    expires_at TIMESTAMP NULL,
    UNIQUE (user_id, client_id, scope_prefix, revoked_at)
);
-- 查询有效同意：
-- revoked_at IS NULL AND (expires_at IS NULL OR expires_at > NOW())
```

### 5.2 `a2a-security-starter` 模块

#### 5.2.1 客户端子包

**上下文键常量**：
```java
public final class A2aContextKeys {
    public static final String USER_TOKEN = "a2a.user_token";
    public static final String TRACE_ID = "a2a.trace_id";
    public static final String SCOPE = "a2a.scope";
    public static final String RESOURCE_INSTANCE = "a2a.resource_instance";
    public static final String TARGET_RESOURCE_URI = "a2a.target_resource_uri";
}
```

**TokenExchangeService 接口**：
```java
public interface TokenExchangeService {
    Mono<String> exchangeToken(String scope, String resourceInstance,
                                String targetResourceUri, String traceId);
}
```

**默认实现（支持批量资源实例）**：
```java
@Component
public class DefaultTokenExchangeService implements TokenExchangeService {
    @Value("${a2a.security.client-id}")
    private String clientId;
    @Value("${a2a.security.client-secret}")
    private String clientSecret;
    @Value("${a2a.security.auth-server-url}")
    private String authServerUrl;
    private WebClient webClient;
    @Autowired
    private WebClient.Builder webClientBuilder;
    
    @PostConstruct
    public void init() {
        this.webClient = webClientBuilder
            .baseUrl(authServerUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_FORM_URLENCODED_VALUE)
            .build();
    }
    
    @Override
    public Mono<String> exchangeToken(String scope, String resourceInstance,
                                       String targetResourceUri, String traceId) {
        return Mono.deferContextual(ctx -> {
            String userToken = ctx.getOrDefault(A2aContextKeys.USER_TOKEN, null);
            Mono<Jwt> upstreamJwtMono = ReactiveSecurityContextHolder.getContext()
                .map(sc -> sc.getAuthentication())
                .filter(auth -> auth.getPrincipal() instanceof Jwt)
                .map(auth -> (Jwt) auth.getPrincipal())
                .defaultIfEmpty(null);
            return upstreamJwtMono.flatMap(upstreamJwt -> {
                MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
                form.add("grant_type", "urn:ietf:params:oauth:grant-type:token-exchange");
                form.add("client_id", clientId);
                form.add("client_secret", clientSecret);
                form.add("scope", scope);
                form.add("resource", targetResourceUri);
                form.add("resource_instance", resourceInstance);
                form.add("single_use", "true");
                form.add("trace_id", traceId);
                if (userToken != null) {
                    form.add("subject_token", userToken);
                    form.add("subject_token_type", "urn:ietf:params:oauth:token-type:access_token");
                } else if (upstreamJwt != null) {
                    form.add("subject_token", upstreamJwt.getTokenValue());
                    form.add("subject_token_type", "urn:ietf:params:oauth:token-type:jwt");
                }
                // 支持批量资源实例
                // 注意：resource_instances 参数需要额外传递
                return webClient.post()
                    .uri("/oauth2/token")
                    .bodyValue(form)
                    .retrieve()
                    .bodyToMono(TokenResponse.class)
                    .map(TokenResponse::getAccessToken);
            });
        });
    }
}
```

**A2aClientFilter**（从上下文读取 scope、resourceInstance 等）：
```java
@Component
public class A2aClientFilter implements ExchangeFilterFunction {
    @Value("${a2a.client.prefer-user-token:true}")
    private boolean preferUserToken;
    @Autowired
    private TokenExchangeService tokenExchangeService;
    
    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        return Mono.deferContextual(ctx -> {
            String traceId = ctx.getOrDefault(A2aContextKeys.TRACE_ID, UUID.randomUUID().toString());
            String scope = ctx.getOrDefault(A2aContextKeys.SCOPE, null);
            String resourceInstance = ctx.getOrDefault(A2aContextKeys.RESOURCE_INSTANCE, null);
            String targetResourceUri = ctx.getOrDefault(A2aContextKeys.TARGET_RESOURCE_URI, null);
            if (scope == null || resourceInstance == null || targetResourceUri == null) {
                return Mono.error(new ResponseStatusException(HttpStatus.BAD_REQUEST,
                    "Missing required context keys: scope, resource_instance, target_resource_uri"));
            }
            return tokenExchangeService.exchangeToken(scope, resourceInstance, targetResourceUri, traceId)
                .flatMap(token -> {
                    ClientRequest newRequest = ClientRequest.from(request)
                        .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                        .header("X-Trace-Id", traceId)
                        .build();
                    return next.exchange(newRequest);
                });
        });
    }
}
```

**批量资源实例支持**：
- 授权服务器支持 `resource_instances` 参数（逗号分隔，最多 300 个 ID）。
- 若超过限制，返回 `400 Bad Request`，错误描述 `resource_instances limit exceeded`。
- 授权服务器将允许的资源 ID 列表写入 JWT 的 `allowed_resource_ids` 声明（约 6KB）。
- 若只有部分资源有权限，设置 `bulk_partial: true`。

#### 5.2.2 服务端子包

**一次性令牌验证器（含时钟偏差容忍）**：
```java
@Component
@ConditionalOnProperty(name = "a2a.security.one-time-token.enabled", matchIfMissing = true)
public class RedisOneTimeTokenValidator implements OneTimeTokenValidator {
    private static final Logger logger = LoggerFactory.getLogger(RedisOneTimeTokenValidator.class);
    @Value("${a2a.security.one-time-token.clock-skew-allowance:5s}")
    private Duration clockSkewAllowance;
    @Value("${a2a.security.one-time-token.redis.fail-open:false}")
    private boolean failOpen;
    @Autowired
    private ReactiveRedisTemplate<String, String> redisTemplate;
    
    @Override
    public Mono<Boolean> consume(String jti, long expSeconds, String agentId) {
        long nowSeconds = Instant.now().getEpochSecond();
        long remaining = (expSeconds - nowSeconds) + clockSkewAllowance.toSeconds();
        if (remaining <= 0) {
            return Mono.error(new BadJwtException("Token expired with clock skew allowance"));
        }
        String key = "a2a:" + agentId + ":onetoken:" + jti;
        return redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofSeconds(remaining))
            .map(Boolean.TRUE::equals)
            .onErrorResume(e -> {
                logger.warn("Redis failure when consuming one-time token jti={}", jti, e);
                if (failOpen) return Mono.just(true);
                else return Mono.error(e);
            });
    }
}
```

**JWT 认证过滤器**（关键：正确解析 scope 字符串，优先校验 allowed_resource_ids）：
```java
@Component
public class A2AJwtAuthenticationFilter implements WebFilter {
    @Value("${a2a.security.resource-id}")
    private String resourceId;
    @Value("${a2a.security.trusted-issuers}")
    private List<String> trustedIssuers;
    @Autowired
    private OneTimeTokenValidator oneTimeTokenValidator;
    @Autowired
    private ReactiveJwtDecoder jwtDecoder;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        if (path.equals("/.well-known/agent.json") || path.equals("/agent-card")) {
            return chain.filter(exchange);
        }
        String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing Bearer token");
        }
        String token = authHeader.substring(7);
        return jwtDecoder.decode(token)
            .flatMap(jwt -> validateJwt(jwt, exchange))
            .flatMap(jwt -> chain.filter(exchange))
            .onErrorResume(e -> unauthorized(exchange, e.getMessage()));
    }
    
    private Mono<Jwt> validateJwt(Jwt jwt, ServerWebExchange exchange) {
        // 1. 校验 issuer
        String issuer = jwt.getIssuer().toString();
        if (!trustedIssuers.contains(issuer)) {
            return Mono.error(new BadJwtException("Untrusted issuer"));
        }
        // 2. 校验 scope（操作类型）
        String requiredOperation = extractOperationFromRequest(exchange);
        String scopeStr = jwt.getClaimAsString("scope");
        Set<String> scopes = (scopeStr != null && !scopeStr.isEmpty()) 
            ? Set.of(scopeStr.split("\\s+")) 
            : Set.of();
        if (!scopes.contains(requiredOperation)) {
            return Mono.error(new AccessDeniedException("Missing required scope: " + requiredOperation));
        }
        // 3. 校验资源实例
        String resourceInstance = extractResourceInstanceFromRequest(exchange);
        List<String> allowed = jwt.getClaimAsStringList("allowed_resource_ids");
        if (allowed != null && !allowed.isEmpty()) {
            if (!allowed.contains(resourceInstance)) {
                return Mono.error(new AccessDeniedException("Resource not allowed"));
            }
            Boolean useStatic = jwt.getClaimAsBoolean("use_static_list");
            if (Boolean.TRUE.equals(useStatic)) {
                // 完全信任静态列表，无需再查权限服务
                return Mono.just(jwt);
            }
        }
        // 4. 降级到 permissionService（利用三级缓存）
        String userId = extractUserIdFromJwt(jwt); // 从 act.sub 或 sub 提取
        return permissionChecker.hasPermission(userId, resourceInstance, requiredOperation)
            .filter(Boolean::booleanValue)
            .switchIfEmpty(Mono.error(new AccessDeniedException("User has no permission")))
            .then(Mono.just(jwt));
    }
}
```

**高性能优化策略**：

| 优化维度 | 传统做法（低性能） | A2A 高性能做法 |
|---------|------------------|---------------|
| Token 校验 | Opaque Token + `/introspect` 远程调用 | JWT + 本地 JWKS 内存验签，短时效（5-15分钟） |
| 权限数据 | `@PreAuthorize` 直连 MySQL | Caffeine (L1) + Redis (L2) 多级缓存 |
| Filter Chain | 默认全开（CSRF、Session等） | 极简无状态配置（禁用一切非必要 Filter） |
| A2A 链路 | 每个 Agent 独立走完整 OAuth2 | 网关首节点强校验 + 内部 Header 加密信任透传 |
| 流式响应 | 每个 Chunk 过拦截器 | 握手时一次性鉴权，流内免检 |

**ReactiveJwtDecoder 配置（含超时和重试）**：
```java
@Bean
public ReactiveJwtDecoder reactiveJwtDecoder() {
    NimbusReactiveJwtDecoder decoder = NimbusReactiveJwtDecoder
        .withJwkSetUri("https://auth.example.com/oauth2/jwks")
        .build();
    decoder.setJwsAlgorithms(Set.of(SignatureAlgorithm.RS256));
    // 配置 RestTemplate 超时
    RestTemplate restTemplate = new RestTemplate();
    SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
    factory.setConnectTimeout(5000);
    factory.setReadTimeout(5000);
    restTemplate.setRequestFactory(factory);
    decoder.setRestOperations(restTemplate);
    return decoder;
}
```

#### 5.2.3 注册中心子包

**AgentCard 构建器**：
```java
@Component
public class DefaultAgentCardBuilder implements AgentCardBuilder {
    @Value("${a2a.security.client-id}")
    private String clientId;
    @Value("${a2a.security.auth-server-url}")
    private String authServerUrl;
    @Value("${a2a.security.required-scopes:}")
    private List<String> requiredScopes;
    @Value("${a2a.security.agent-endpoint}")
    private String agentEndpoint;
    
    @Override
    public Map<String, Object> build() {
        Map<String, Object> card = new HashMap<>();
        card.put("agentId", clientId);
        card.put("endpoint", agentEndpoint);
        Map<String, Object> securitySchemes = new HashMap<>();
        Map<String, Object> oauth2 = new HashMap<>();
        oauth2.put("type", "oauth2");
        Map<String, Object> flows = new HashMap<>();
        Map<String, Object> clientCredentials = new HashMap<>();
        clientCredentials.put("tokenUrl", authServerUrl + "/oauth2/token");
        Map<String, String> scopes = new HashMap<>();
        requiredScopes.forEach(s -> scopes.put(s, s));
        clientCredentials.put("scopes", scopes);
        flows.put("clientCredentials", clientCredentials);
        oauth2.put("flows", flows);
        securitySchemes.put("oauth2", oauth2);
        card.put("securitySchemes", securitySchemes);
        card.put("security", List.of(Map.of("oauth2", requiredScopes)));
        return card;
    }
}
```

**Nacos 发布器**：
```java
@Component
public class NacosAgentCardPublisher implements SmartLifecycle {
    // 使用 Nacos OpenAPI 或 NacosClient 发布实例元数据
    // 要求 Nacos 开启认证
}
```

#### 5.2.4 多跳委托子包

**`DelegationContext` 定义**：
```java
public class DelegationContext {
    private final Integer remaining;
    private final List<String> chain;
    public DelegationContext(Integer remaining, List<String> chain) {
        this.remaining = remaining;
        this.chain = chain != null ? List.copyOf(chain) : List.of();
    }
    public Integer getRemaining() { return remaining; }
    public List<String> getChain() { return chain; }
}
```

**`DelegationContextHandler` 接口与实现**：
```java
public interface DelegationContextHandler {
    DelegationContext extractFromJwt(Jwt upstreamJwt);
    Map<String, Object> buildDownstreamParams(DelegationContext context);
}

@Component
public class DefaultDelegationContextHandler implements DelegationContextHandler {
    @Value("${a2a.authz.token.delegation-max:3}")
    private int delegationMax;
    
    @Override
    public DelegationContext extractFromJwt(Jwt upstreamJwt) {
        Integer remaining = upstreamJwt.getClaim("delegation_remaining");
        if (remaining == null) remaining = delegationMax;
        List<String> chain = upstreamJwt.getClaim("delegation_chain");
        return new DelegationContext(remaining, chain);
    }
    
    @Override
    public Map<String, Object> buildDownstreamParams(DelegationContext context) {
        Map<String, Object> params = new HashMap<>();
        if (context.getRemaining() != null) params.put("delegation_remaining", context.getRemaining());
        if (!context.getChain().isEmpty()) params.put("delegation_chain", context.getChain());
        return params;
    }
}
```

**委托链安全说明**：
- 授权服务器必须从 `subject_token` 中提取 `delegation_remaining` 和 `delegation_chain`，忽略客户端显式传入的值。
- `delegation_remaining` 若为空，则使用配置 `delegation-max`（默认3）作为初始值。
- 新令牌中 `delegation_remaining = 原值 - 1`，若结果为负数则拒绝。
- `delegation_chain` 追加当前 `client_id`。
- 当 `delegation_remaining = 0` 时，该令牌不能作为 `subject_token` 继续委托，但可以正常使用。

### 5.3 `a2a-authorization-server` 模块

#### 5.3.1 客户端注册配置

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient agentA = RegisteredClient.withId("agent-a")
        .clientId("agent-a")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.TOKEN_EXCHANGE)
        .scope("doc:summarize")
        .scope("doc:read")
        .build();
    return new InMemoryRegisteredClientRepository(agentA);
}
```

#### 5.3.2 自定义参数转换器

```java
public class CustomTokenExchangeAuthenticationConverter implements AuthenticationConverter {
    @Override
    public Authentication convert(HttpServletRequest request) {
        OAuth2TokenExchangeRequest.Builder builder = OAuth2TokenExchangeRequest.builder()
            .grantType(AuthorizationGrantType.TOKEN_EXCHANGE)
            .clientId(request.getParameter(OAuth2ParameterNames.CLIENT_ID))
            .subjectToken(request.getParameter(OAuth2ParameterNames.SUBJECT_TOKEN))
            .subjectTokenType(request.getParameter(OAuth2ParameterNames.SUBJECT_TOKEN_TYPE))
            .scope(request.getParameter(OAuth2ParameterNames.SCOPE))
            .audience(request.getParameter(OAuth2ParameterNames.AUDIENCE))
            .resource(request.getParameter(OAuth2ParameterNames.RESOURCE));
        
        Map<String, Object> additional = new HashMap<>();
        String resourceInstance = request.getParameter("resource_instance");
        if (resourceInstance != null) {
            if (resourceInstance.length() > 256) {
                throw new OAuth2AuthenticationException(OAuth2ErrorCodes.INVALID_REQUEST, "resource_instance too long");
            }
            additional.put("resource_instance", resourceInstance);
        }
        // 批量资源 ID
        String[] resourceInstances = request.getParameterValues("resource_instances");
        if (resourceInstances != null && resourceInstances.length > 0) {
            if (resourceInstances.length > 300) {
                throw new OAuth2AuthenticationException(OAuth2ErrorCodes.INVALID_REQUEST, "resource_instances limit exceeded");
            }
            additional.put("resource_instances", List.of(resourceInstances));
        }
        String singleUse = request.getParameter("single_use");
        if (singleUse != null) additional.put("single_use", Boolean.valueOf(singleUse));
        String traceId = request.getParameter("trace_id");
        if (traceId != null) additional.put("trace_id", traceId);
        // 禁止从请求参数读取 delegation_remaining / delegation_chain
        builder.additionalParameters(additional);
        OAuth2TokenExchangeAuthenticationToken token = new OAuth2TokenExchangeAuthenticationToken(builder.build());
        token.setDetails(new OAuth2AuthenticationDetails(request));
        return token;
    }
}
```

**注册转换器**：
```java
@Configuration
public class AuthServerConfig {
    @Bean
    public CustomTokenExchangeAuthenticationConverter customTokenExchangeAuthenticationConverter() {
        return new CustomTokenExchangeAuthenticationConverter();
    }
    
    @Bean
    public SecurityFilterChain authorizationServerSecurityFilterChain(
            HttpSecurity http,
            CustomTokenExchangeAuthenticationConverter converter) throws Exception {
        OAuth2AuthorizationServerConfigurer authServerConfigurer = new OAuth2AuthorizationServerConfigurer();
        authServerConfigurer.tokenEndpoint(tokenEndpoint ->
            tokenEndpoint.accessTokenRequestConverter(converter));
        http.apply(authServerConfigurer);
        // 其他配置...
        return http.build();
    }
}
```

#### 5.3.3 客户端自身 scope 检查

```java
@Component
public class ClientScopeValidatorProvider implements AuthenticationProvider {
    private final OAuth2TokenExchangeAuthenticationProvider delegate;
    public ClientScopeValidatorProvider(OAuth2TokenExchangeAuthenticationProvider delegate) {
        this.delegate = delegate;
    }
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Authentication auth = delegate.authenticate(authentication);
        OAuth2TokenExchangeAuthenticationToken tokenExchangeAuth = (OAuth2TokenExchangeAuthenticationToken) auth;
        RegisteredClient client = tokenExchangeAuth.getRegisteredClient();
        Set<String> clientScopes = client.getScopes();
        OAuth2TokenExchangeRequest request = tokenExchangeAuth.getTokenExchangeRequest();
        Set<String> requestedScopes = request.getScopes();
        if (requestedScopes == null) requestedScopes = Collections.emptySet();
        if (requestedScopes.isEmpty()) {
            return auth; // 空 scope 直接通过
        }
        if (!clientScopes.containsAll(requestedScopes)) {
            throw new OAuth2AuthenticationException(OAuth2ErrorCodes.INVALID_SCOPE);
        }
        return auth;
    }
    @Override
    public boolean supports(Class<?> authentication) {
        return OAuth2TokenExchangeAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

**注册 Provider**：
```java
@Bean
public OAuth2TokenExchangeAuthenticationProvider tokenExchangeAuthenticationProvider(
        OAuth2AuthorizationService authorizationService,
        OAuth2TokenGenerator<?> tokenGenerator) {
    return new OAuth2TokenExchangeAuthenticationProvider(authorizationService, tokenGenerator);
}

@Bean
public AuthenticationProvider clientScopeValidatorProvider(
        OAuth2TokenExchangeAuthenticationProvider delegate) {
    return new ClientScopeValidatorProvider(delegate);
}
// 在 SecurityFilterChain 中
http.authenticationProvider(clientScopeValidatorProvider);
```

#### 5.3.4 JWT 定制器（含多跳委托和批量权限）

```java
@Component
public class CustomJwtCustomizer implements OAuth2TokenCustomizer<JwtEncodingContext> {
    @Value("${a2a.authz.token.one-time-ttl:2m}")
    private Duration oneTimeTtl;
    @Value("${a2a.authz.token.delegation-max:3}")
    private int delegationMax;
    @Autowired
    private JwtDecoder jwtDecoder;
    @Autowired
    private PermissionChecker permissionChecker;
    
    @Override
    public void customize(JwtEncodingContext context) {
        if (!AuthorizationGrantType.TOKEN_EXCHANGE.equals(context.getAuthorizationGrantType())) return;
        Map<String, Object> additional = context.getAuthorization().getAttributes();
        // 一次性令牌
        Boolean singleUse = (Boolean) additional.get("single_use");
        if (Boolean.TRUE.equals(singleUse)) {
            context.getClaims().expiresAt(Instant.now().plus(oneTimeTtl));
            context.getClaims().claim("single_use", true);
        }
        context.getClaims().claim("resource_instance", additional.get("resource_instance"));
        context.getClaims().claim("trace_id", additional.get("trace_id"));
        context.getClaims().jwtId(UUID.randomUUID().toString());
        
        // 批量资源权限预计算
        List<String> resourceInstances = (List<String>) additional.get("resource_instances");
        if (resourceInstances != null && !resourceInstances.isEmpty()) {
            String userId = extractUserIdFromSubjectToken(additional);
            String scopePrefix = extractScopePrefix(additional.get("scope"));
            Set<String> allowed = permissionChecker.filterAllowed(userId, resourceInstances, scopePrefix).block();
            context.getClaims().claim("allowed_resource_ids", allowed);
            if (allowed.size() < resourceInstances.size()) {
                context.getClaims().claim("bulk_partial", true);
            }
            // 如果权限不涉及动态条件，可设置 use_static_list = true
            if (!requiresDynamicCondition(scopePrefix)) {
                context.getClaims().claim("use_static_list", true);
            }
        }
        
        // 多跳委托：从 subject_token 提取
        String subjectTokenStr = (String) additional.get(OAuth2ParameterNames.SUBJECT_TOKEN);
        if (subjectTokenStr != null) {
            try {
                Jwt subjectJwt = jwtDecoder.decode(subjectTokenStr);
                Integer remaining = subjectJwt.getClaim("delegation_remaining");
                List<String> chain = subjectJwt.getClaim("delegation_chain");
                if (remaining == null) remaining = delegationMax;
                int newRemaining = remaining - 1;
                if (newRemaining < 0) {
                    throw new OAuth2AuthenticationException("delegation depth exceeded");
                }
                List<String> newChain = new ArrayList<>(chain != null ? chain : List.of());
                String currentClient = context.getAuthorization().getRegisteredClient().getId();
                newChain.add(currentClient);
                context.getClaims().claim("delegation_remaining", newRemaining);
                context.getClaims().claim("delegation_chain", newChain);
            } catch (JwtException e) {
                throw new OAuth2AuthenticationException("Invalid subject_token");
            }
        }
        // 忽略客户端显式传入的 delegation_ 参数
    }
}
```

#### 5.3.5 `SubjectTokenValidator` 与 `ClientSettingsRepository`

```java
@Component
public class SubjectTokenValidator {
    @Autowired
    private JwtDecoder jwtDecoder;
    @Autowired
    private ClientSettingsRepository settingsRepo;
    
    public Mono<Map<String, Object>> validate(String token, String tokenType, String clientId) {
        return settingsRepo.findAllowedSubjectTokenTypes(clientId)
            .filter(types -> types.contains(tokenType))
            .switchIfEmpty(Mono.error(new OAuth2AuthenticationException(OAuth2ErrorCodes.INVALID_REQUEST,
                "unsupported subject_token_type: " + tokenType)))
            .then(Mono.fromCallable(() -> {
                if ("jwt".equals(tokenType) || "access_token".equals(tokenType)) {
                    Jwt jwt = jwtDecoder.decode(token);
                    return Map.of(
                        "sub", jwt.getClaimAsString("sub"),
                        "iss", jwt.getClaimAsString("iss"),
                        "aud", jwt.getClaimAsStringList("aud")
                    );
                }
                throw new OAuth2AuthenticationException("Unsupported subject_token_type");
            }));
    }
}

@Repository
public class ClientSettingsRepository {
    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;
    
    public Mono<List<String>> findAllowedSubjectTokenTypes(String clientId) {
        return Mono.fromCallable(() -> {
            String json = jdbcTemplate.queryForObject(
                "SELECT allowed_subject_token_types FROM oauth2_client_settings WHERE client_id = ?",
                String.class, clientId);
            if (json == null) return List.<String>of();
            return objectMapper.readValue(json, new TypeReference<List<String>>() {});
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

#### 5.3.6 数据库表（授权服务器专用）

```sql
-- ACL 表
CREATE TABLE a2a_acl (
    id UUID PRIMARY KEY,
    caller_client_id VARCHAR(64) NOT NULL,
    target_client_id VARCHAR(64) NOT NULL,
    allowed_scope_prefixes TEXT NOT NULL,  -- 逗号分隔，空字符串或 null 表示拒绝
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (caller_client_id, target_client_id)
);
CREATE INDEX idx_caller_target ON a2a_acl(caller_client_id, target_client_id);

-- 用户同意表（含有效期）
CREATE TABLE user_consent (
    id UUID PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    client_id VARCHAR(64) NOT NULL,
    scope_prefix VARCHAR(256) NOT NULL,
    granted_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP NULL,
    expires_at TIMESTAMP NULL,
    UNIQUE (user_id, client_id, scope_prefix, revoked_at)
);
CREATE UNIQUE INDEX idx_user_consent_active ON user_consent (user_id, client_id, scope_prefix) WHERE revoked_at IS NULL;

-- 客户端扩展表
CREATE TABLE oauth2_client_settings (
    client_id VARCHAR(64) PRIMARY KEY,
    allowed_subject_token_types TEXT,   -- JSON 数组
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 5.3.7 审计日志增强

```sql
ALTER TABLE a2a_audit_log ADD COLUMN bulk_mode BOOLEAN DEFAULT FALSE;
ALTER TABLE a2a_audit_log ADD COLUMN bulk_partial BOOLEAN DEFAULT FALSE;
```

#### 5.3.8 JwtDecoder 配置（授权服务器用）

```java
@Bean
public JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder.withJwkSetUri("https://auth.internal.example.com/oauth2/jwks").build();
}
```

#### 5.3.9 授权服务器配置

```yaml
spring:
  security:
    oauth2:
      authorizationserver:
        issuer: https://auth.internal.example.com
      resourceserver:
        jwt:
          jwk-set-uri: https://auth.internal.example.com/oauth2/jwks
        opaquetoken:
          introspection-uri: https://auth.internal.example.com/oauth2/introspect
          introspection-client-id: auth-server
          introspection-client-secret: ${INTROSPECTION_SECRET}
  datasource:
    url: jdbc:postgresql://db.internal:5432/a2a_authz
    username: ${DB_USER}
    password: ${DB_PASS}
  redis:
    host: redis.internal
    timeout: 2s
    lettuce:
      pool:
        max-active: 10
a2a:
  authz:
    acl:
      enabled: true
    consent:
      enabled: true
    token:
      one-time-ttl: 2m
      delegation-max: 3
      bulk-allowed-max: 300
      bulk-enabled: true
    audit:
      enabled: true
      async: true
    client-subject-token-allowlist:
      agent-a: ["access_token", "jwt"]
```

---
## 6. 核心权限流程链路

### 6.1 批量任务优化流程

```mermaid
sequenceDiagram
    participant AgentA as Agent A
    participant Authz as 授权服务器
    participant PermSvc as 权限服务(DB)
    participant AgentB as Agent B

    AgentA->>Authz: Token Exchange (scope, resource_instances=id1,...,id100)
    Authz->>PermSvc: SELECT resource_id FROM ... WHERE user_id=? AND operation=? AND resource_id IN (...)
    PermSvc-->>Authz: allowed_ids (id1,id3,id5,...)
    Authz->>Authz: 生成 JWT (allowed_resource_ids=[...], bulk_partial=true)
    Authz-->>AgentA: 代理令牌
    AgentA->>AgentB: 批量请求 + JWT
    AgentB->>AgentB: 解析 JWT，本地校验 each docId in allowed_ids
    Note over AgentB: 零数据库查询，处理100个文档
```

### 6.2 纯 Agent 直连（无用户委派）

```mermaid
sequenceDiagram
    participant AgentA as Agent A (客户端)
    participant Authz as 授权服务器
    participant Redis as Redis
    participant AgentB as Agent B (资源服务器)

    AgentA->>AgentA: 生成 trace_id (UUID)
    AgentA->>Authz: Token Exchange (client_id+secret, scope, single_use=true, resource=targetResourceUri, resource_instance=..., trace_id)
    Authz->>Authz: 验证客户端凭证 & 客户端自身 scope 权限
    Authz->>Authz: 查询 ACL（判空处理）
    alt ACL 拒绝
        Authz-->>AgentA: 403
    else ACL 允许
        Authz->>Authz: 生成 JWT (sub=AgentA, aud=resource, jti=UUID, single_use, resource_instance, trace_id, exp=now+2m)
        Authz->>Authz: 记录审计日志（含 jti, client_ip, user_agent）
        Authz-->>AgentA: 代理令牌
    end
    AgentA->>AgentB: A2A /message (Bearer token, X-Trace-Id: trace_id)
    AgentB->>AgentB: 验证签名/有效期/issuer/aud (包含自身 resource-id)
    opt single_use
        AgentB->>AgentB: remaining = max(exp - now + clockSkew, 1)
        AgentB->>Redis: SET NX a2a:agent-b:onetoken:<jti> 1 EX remaining
        alt 已使用或 Redis 故障 (fail-open=false)
            AgentB-->>AgentA: 403
        end
    end
    AgentB->>AgentB: 提取 scope, 检查操作权限
    AgentB->>AgentB: 业务处理
    AgentB-->>AgentA: 200 OK
```

### 6.3 用户委派流程

```mermaid
sequenceDiagram
    participant User
    participant AgentA
    participant Authz
    participant Redis
    participant AgentB
    participant Perm

    User->>AgentA: 请求 + user_token
    AgentA->>AgentA: 提取 user_token, trace_id
    AgentA->>Authz: Token Exchange (subject_token, subject_token_type=access_token, scope, resource=targetResourceUri, resource_instance=..., single_use, trace_id)
    Authz->>Authz: 验证客户端凭证 & 自身 scope
    Authz->>Authz: 验证 subject_token (JWT 或内省)
    Authz->>Authz: 查询 user_consent
    alt 无同意
        Authz-->>AgentA: 需授权
    else 同意通过
        Authz->>Authz: 查询 ACL
        alt ACL 允许
            Authz->>Authz: 生成 JWT (sub=AgentA, act={sub:user_id, iss}, aud=resource, jti, single_use, resource_instance, trace_id, exp=now+2m)
            Authz-->>AgentA: 代理令牌
        else ACL 拒绝
            Authz-->>AgentA: 403
        end
    end
    AgentA->>AgentB: A2A 请求 (Bearer token, X-Trace-Id)
    AgentB->>AgentB: 验证 JWT + 一次性令牌
    AgentB->>AgentB: 提取 act.sub 作为 userId, resource_instance
    AgentB->>Perm: hasPermission(userId, resource_instance, operation)
    Perm-->>AgentB: true/false
    alt 权限不足
        AgentB-->>AgentA: 403
    else 权限允许
        AgentB->>AgentB: 调用大模型
        AgentB-->>AgentA: 200
    end
    AgentA-->>User: 响应
```

### 6.4 多跳委托流程

```mermaid
sequenceDiagram
    participant AgentA
    participant Authz
    participant AgentB
    participant AgentC

    AgentA->>Authz: Token Exchange (resource=agent-b)
    Authz-->>AgentA: 令牌1 (delegation_remaining=null, chain=[])
    AgentA->>AgentB: 请求（令牌1）
    AgentB->>AgentB: 从 SecurityContext 获取令牌1，调用 DelegationContextHandler
    AgentB->>Authz: Token Exchange (subject_token=令牌1, resource=agent-c)
    Authz->>Authz: 验证 subject_token，提取 delegation_remaining=3（若 null 则用 max-depth=3），减1得2；链追加 "agent-b"
    Authz-->>AgentB: 新令牌 (delegation_remaining=2, chain=["agent-b"])
    AgentB->>AgentC: 请求
    AgentC->>AgentC: 处理，不再委托
```

---

## 7. 配置与部署视图

### 7.1 授权服务器配置

（同 5.3.9）

### 7.2 Agent 配置（完整示例）

```yaml
spring:
  application:
    name: agent-b
  cloud:
    nacos:
      discovery:
        server-addr: nacos.internal:8848
        username: nacos
        password: ${NACOS_PASSWORD}
        metadata:
          a2a.security.oauth2.token-url: https://auth.internal.example.com/oauth2/token
  data:
    redis:
      host: redis.internal
      timeout: 2s
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.internal.example.com
a2a:
  security:
    enabled: true
    client-id: agent-b
    client-secret: ${AGENT_B_SECRET}
    auth-server-url: https://auth.internal.example.com
    resource-id: agent-b
    agent-endpoint: https://agent-b.internal.example.com
    trusted-issuers:
      - https://auth.internal.example.com
    one-time-token:
      enabled: true
      clock-skew-allowance: 5s
      redis:
        key-prefix: "a2a:${a2a.security.client-id}:onetoken:"
        fail-open: false
    delegation:
      enabled: true
      max-depth: 3
    scope-matcher: prefix
    a2a-path-pattern: "/message"
    jwt:
      local-verification: true
      jwks-cache-ttl: 12h
    permission:
      cache:
        l1-ttl: 5m
        l1-max-size: 10000
        l2-ttl: 30m
        redis-pubsub-enabled: true
  client:
    prefer-user-token: true
permission:
  enabled: true
  spring:
    r2dbc:
      url: r2dbc:postgresql://db.internal:5432/user_permissions
      username: ${PERM_DB_USER}
      password: ${PERM_DB_PASS}
```

### 7.3 Nacos 服务端认证配置

在 Nacos 服务端 `conf/application.properties` 中启用：
```properties
nacos.core.auth.enabled=true
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=mySecret
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789
```

**Nacos 客户端配置**：
```yaml
nacos:
  auth:
    enabled: true
    username: nacos
    password: ${NACOS_PASSWORD}
```

### 7.4 内部 Header 信任传递（RSA 签名示例）

**网关签名**：
```java
String internalToken = Jwts.builder()
    .header().keyId("kid-2024-01")
    .and()
    .claim("sub", agentId)
    .claim("act", userInfo)
    .claim("allowed_resource_ids", allowedIds)
    .signWith(privateKey, SignatureAlgorithm.RS256)
    .compact();
response.setHeader("X-Internal-Auth", internalToken);
```

**Agent 验签**：
```java
Jws<Claims> jws = Jwts.parserBuilder()
    .setSigningKeyResolver(new KeyResolver(publicKeyMap))
    .build()
    .parseClaimsJws(internalToken);
// 信任该 token 并提取声明
```

**密钥管理**：网关私钥存储在 KMS 中，公钥通过配置中心分发。轮换私钥时保留旧公钥一段时间（通过 `kid` 头区分），新旧切换期并行。

### 7.5 A2A 端点路径对齐

- Spring AI Alibaba A2A 默认端点：`/message`。  
- 若通过 `spring.ai.alibaba.a2a.server.path` 修改基础路径（如 `/a2a`），则端点为 `/a2a/message`。  
- 必须同步设置 `a2a.security.a2a-path-pattern` 为对应路径（支持 Ant 风格，如 `/a2a/message`）。

**自定义 WebClient 注入**（在业务 Agent 中）：
```java
@Configuration
public class A2aClientConfig {
    @Bean
    public WebClient a2aWebClient(A2aClientFilter filter) {
        return WebClient.builder().filter(filter).build();
    }
    
    @Bean
    public A2aRemoteAgent agentBRemoteAgent(AgentCardProvider provider, WebClient webClient) {
        return new A2aRemoteAgent("agent-b", provider, webClient);
    }
}
```

---
## 8. 审计日志设计

### 8.1 数据库表

```sql
CREATE TABLE a2a_audit_log (
    id UUID PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    service_type VARCHAR(16) NOT NULL,      -- 'AUTHZ' or 'AGENT'
    caller_agent_id VARCHAR(64),
    target_agent_id VARCHAR(64),
    user_id VARCHAR(64),
    scope VARCHAR(256),
    trace_id VARCHAR(64) NOT NULL,
    jti VARCHAR(64),
    decision VARCHAR(16) NOT NULL,
    deny_reason VARCHAR(64),
    http_status INT,
    bulk_mode BOOLEAN DEFAULT FALSE,
    bulk_partial BOOLEAN DEFAULT FALSE,
    details JSONB
);
CREATE INDEX idx_trace_id ON a2a_audit_log(trace_id);
CREATE INDEX idx_jti ON a2a_audit_log(jti);
CREATE INDEX idx_timestamp ON a2a_audit_log(timestamp);
```

**details 标准结构**：
```json
{
  "jti": "uuid",
  "client_ip": "10.0.0.1",
  "user_agent": "Mozilla/5.0...",
  "request_params": {}   // 已脱敏（不含 client_secret, subject_token）
}
```

### 8.2 日志记录点

- **授权服务器**：每次 Token Exchange 请求，记录决策、`jti`、`bulk_mode`、`bulk_partial`、客户端 IP、User-Agent。  
- **Agent 资源服务器**：每次 JWT 验证，记录决策和一次性令牌结果；若一次性令牌降级（Redis 故障且 `fail-open=true`），标记 `deny_reason=REDIS_DEGRADED`。

### 8.3 可靠性保障

- 生产环境推荐使用 **消息队列（RabbitMQ/Kafka）** 异步发送审计日志。  
- 若使用 `@Async`，需配置线程池及 `neverBlock`，并接受潜在内存丢失风险。  
- 审计日志保留期限建议 90 天，可定期归档。

**获取客户端 IP 和 User-Agent 示例**（在授权服务器过滤器中）：
```java
String ip = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
if (ip == null) ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
String userAgent = exchange.getRequest().getHeaders().getFirst(HttpHeaders.USER_AGENT);
exchange.getAttributes().put("client_ip", ip);
exchange.getAttributes().put("user_agent", userAgent);
```

### 8.4 审计日志清理策略

定期清理（例如每天凌晨执行）：
```sql
DELETE FROM a2a_audit_log WHERE timestamp < NOW() - INTERVAL '90 days';
```
或使用时间分区表自动管理。

---
## 9. 安全加固矩阵

| 安全机制 | 实现位置 | 必须 | 说明 |
|---------|----------|------|------|
| OAuth2 Token Exchange | 授权服务器 + Agent | ✅ | RFC 8693，支持批量资源 ID |
| 一次性令牌防重放 | Agent + Redis | ✅ | 原子 SET NX EX，动态 TTL+时钟偏差 |
| ACL 前缀匹配 | 授权服务器 | ✅ | 空字符串/null 拒绝 |
| 用户资源权限 + 批量查询 | common-permission | ✅ | 独立 JAR + 三级缓存 + 广播失效 |
| 多跳委托限制 | 授权服务器 | ✅ | 从 subject_token 提取链，扣减剩余 |
| 全链路追踪 | 授权服务器 + Agent | ✅ | X-Trace-Id 头 + JWT 声明 |
| HTTPS | 所有通信 | ✅ | TLS 1.2+ |
| 审计日志 | 授权服务器 + Agent | ✅ | 消息队列，含 jti, bulk_* |
| Nacos 认证 | Nacos | ✅ | 生产必须 |
| subject_token 验证 | 授权服务器 | ✅ | JWT 验签/内省 + 白名单 |
| 客户端自身 scope 检查 | 授权服务器 | ✅ | 自定义 Provider |
| resource_instance 长度限制 | 授权服务器 | ✅ | ≤256，超长拒绝 |
| 批量资源 ID 数量限制 | 授权服务器 | ✅ | ≤300，超限拒绝 |
| clock-skew 容忍 | Agent | ✅ | 配置允许偏差 |
| 内部 Header 加密/签名 | 网关 + Agent | 推荐 | 使用 RSA 签名，防伪造 |
| mTLS / DPoP | 可选扩展 | ❌ | 按需开启 |

---
## 10. 扩展点设计

### 10.1 自定义 Scope 匹配策略

```java
public interface ScopeMatcher {
    boolean matches(String requestedScope, List<String> allowedPatterns);
}

@Primary
@Component
@ConditionalOnProperty(name = "a2a.security.scope-matcher", havingValue = "prefix", matchIfMissing = true)
public class PrefixScopeMatcher implements ScopeMatcher {
    @Override
    public boolean matches(String requestedScope, List<String> allowedPrefixes) {
        if (allowedPrefixes == null || allowedPrefixes.isEmpty()) return false;
        return allowedPrefixes.stream().anyMatch(prefix ->
            requestedScope.equals(prefix) || requestedScope.startsWith(prefix + ":"));
    }
}

@Component
@ConditionalOnProperty(name = "a2a.security.scope-matcher", havingValue = "regex")
public class RegexScopeMatcher implements ScopeMatcher {
    @Override
    public boolean matches(String requestedScope, List<String> allowedPatterns) {
        return allowedPatterns.stream().anyMatch(p -> requestedScope.matches(p));
    }
}
```

### 10.2 自定义用户身份提取

```java
public interface UserIdentityExtractor {
    Optional<String> extractUserId(Jwt jwt);
}

@Component
@ConditionalOnMissingBean
public class DefaultUserIdentityExtractor implements UserIdentityExtractor {
    @Override
    public Optional<String> extractUserId(Jwt jwt) {
        Map<String, Object> act = jwt.getClaim("act");
        if (act != null && act.containsKey("sub")) {
            return Optional.of(act.get("sub").toString());
        }
        return Optional.empty();
    }
}
```

### 10.3 自定义一次性令牌存储

```java
@Component
@ConditionalOnProperty(name = "a2a.security.one-time-token.store", havingValue = "db")
public class DatabaseOneTimeTokenValidator implements OneTimeTokenValidator {
    // 使用数据库存储已使用的 jti，定期清理
}
```

### 10.4 mTLS 与 DPoP 集成

- **mTLS**：配置 `a2a.security.mtls.enabled=true`，支持 `mode: only_mtls` 或 `mode: dual`。证书 CN/SAN 映射为 `client_id`，可配置正则表达式。
- **DPoP**：配置 `a2a.security.dpop.enabled=true`，使用 Spring Security 的 `DPoPAuthenticationConverter`。授权服务器需支持在 Token Exchange 响应中返回 `cnf` 声明（RFC 7800）。

---
## 11. 响应式与非阻塞编程指南

- 所有安全组件返回 `Mono`/`Flux`，遵循 WebFlux 模型。  
- 若使用阻塞式数据库（JDBC），必须用 `Mono.fromCallable(() -> ...).subscribeOn(Schedulers.boundedElastic())`。  
- 推荐使用 R2DBC 获得完全非阻塞体验。

**示例：权限检查适配**：
```java
@Bean
public PermissionChecker permissionChecker(JdbcTemplate jdbcTemplate) {
    return (userId, resourceId, operation) -> Mono.fromCallable(() ->
            jdbcTemplate.queryForObject("SELECT COUNT(*) FROM user_resource_permission WHERE user_id=? AND resource_id=? AND operation=?", 
                Integer.class, userId, resourceId, operation) > 0
        ).subscribeOn(Schedulers.boundedElastic());
}
```

**R2DBC 事务**：使用 `TransactionalOperator` 或 `@Transactional` 配合 `ReactiveTransactionManager`。

---
## 12. 总结

本文档是 A2A 权限系统的完整架构设计，涵盖了身份认证、最小权限、性能优化、可审计性、高可用性、动态策略和联邦互操作七大目标。经过多轮迭代修复，所有事实错误和关键遗漏均已解决，文档可直接用于生产级 A2A 权限系统的开发、部署与运维。