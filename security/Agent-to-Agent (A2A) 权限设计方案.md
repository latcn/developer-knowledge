
## 1. 概述

本文档描述一套面向多智能体系统（Multi-Agent）的 Agent-to-Agent（A2A）权限设计方案，基于 **Spring AI Alibaba**、**Spring Security OAuth2** 及 **Spring Authorization Server** 构建。方案重点解决以下核心需求：

- **服务间认证与授权**：Agent 之间安全互访，防止未授权调用。
- **临时资源细粒度许可**：可针对单个资源（如文档、会话）生成精准的操作权限（读、写、总结），权限范围受限且有效期极短。
- **随用随销（一次性令牌）**：敏感操作令牌只能使用一次，防止重放攻击。
- **用户身份透传**：Agent 可以代表最终用户执行操作，下游能追溯原始用户。
- **无 AI 网关**：每个 Agent 独立持有模型提供商的 API Key，内部直连阿里云 DashScope、OpenAI 等，不引入集中网关。
- **多模型提供商自由选择**：不同 Agent 可使用不同的大模型，AI 调用对调用方透明，权限不耦合到模型。

---
## 2. 总体架构

```
┌───────────────┐   Token Exchange + 细粒度 scope   ┌───────────────┐
│   Agent A     │ ─────────────────────────────────▶ │   Agent B     │
│ (OAuth2客户端)│                                   │ (资源服务器)   │
│               │◀───────────────────────────────── │               │
└───────┬───────┘  受保护业务 API（如 /docs/123/summarize） └───────┬───────┘
        │                                                         │
        │ client_credentials / token exchange                     │ 自行持有 API Key
        ▼                                                         ▼
┌─────────────────────────────┐               ┌──────────────────────────────┐
│ Spring Authorization Server │               │  模型提供商                   │
│ - 颁发 JWT 令牌              │               │  - 阿里云 DashScope          │
│ - Token Exchange 支持        │               │  - OpenAI                   │
│ - 自定义 claims              │               │  - Azure OpenAI 等          │
└─────────┬───────────────────┘               └──────────────────────────────┘
          │
          │ 验证用户 token
          ▼
   ┌──────────┐
   │   用户   │
   └──────────┘
```

核心组件说明：

- **授权服务器**：Spring Authorization Server，负责客户端注册、颁发 JWT、支持 Token Exchange（RFC 8693）。
- **每个 Agent**：
  - 作为 **OAuth2 客户端**，通过 `client_credentials` 或 Token Exchange 获取令牌。
  - 作为 **资源服务器**，暴露 A2A 业务 API（非 AI 接口），通过 JWT 鉴权并执行一次性检查。
- **Redis**：存储已使用过的 `jti`（JWT ID），实现一次性令牌。
- **模型提供商**：Agent 内部直接集成，通过 Spring AI 统一抽象，API Key 配置在各自的环境变量中。

---
## 3. 权限模型：三层 Scope 设计

我们将 OAuth2 Scope 分为三层，实现从通用能力到具体资源的细粒度控制：

| 层级         | Scope 示例                          | 说明                                         |
|--------------|-------------------------------------|----------------------------------------------|
| **通用能力** | `agent:rag`、`agent:summarize`      | 表示 Agent 提供的基础功能，用于粗粒度授权。    |
| **资源类型** | `doc:read`、`doc:summarize`         | 对某类资源的操作权限，可进一步限定实例。       |
| **临时细粒度** | `doc:summarize:report-123`         | 针对具体资源（如文档 ID）的一次性操作许可。    |

- **客户端注册时**授予较粗的 scope（如 `agent:summarize`、`doc:read`），作为基础权限。
- **执行具体操作时**，通过 Token Exchange 动态生成细粒度 scope（如 `doc:summarize:report-123`），并包含资源标识、一次性标记等。

---
## 4. 关键流程：Token Exchange 与一次性令牌

### 4.1 流程概览

1. **用户认证**（若涉及用户委派）：用户通过授权码模式获得 access token（`subject_token`）。
2. **Agent A 发起委派请求**：
   - 请求授权服务器的 Token Exchange 端点。
   - 参数：
     - `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
     - `subject_token` = 用户的 access_token
     - `subject_token_type=urn:ietf:params:oauth:token-type:access_token`
     - `client_id` / `client_secret` = Agent A 的凭证
     - `scope` = 所需的细粒度 scope（如 `doc:summarize:report-123`）
     - `resource` = 目标资源 ID（如 `report-123`）
     - `single_use` = `true`
3. **授权服务器处理**：
   - 验证 Agent A 客户端身份及 `subject_token` 有效性。
   - 检查 Agent A 是否被授权代理用户的这些 scope。
   - 生成新的 JWT，包含：
     - `sub`：Agent A 的 client_id
     - `act`：嵌套的原始用户信息（sub、scope 等）
     - `scope`：细粒度 scope 列表
     - `resource_id`：具体资源 ID
     - `single_use`：`true`
     - `jti`：唯一令牌 ID
     - `exp`：较短有效期（如 2 分钟）
4. **Agent A 调用 Agent B**：在 HTTP 头 `Authorization: Bearer <exchange_token>` 中携带该令牌。
5. **Agent B 验证与一次性销毁**：
   - 验证 JWT 签名、有效期、scope 等。
   - 检查 `single_use` 为 true 时，查询 Redis 中键 `jti:<jti>` 是否存在。
     - **不存在**：放行，并立即设置该键（带 TTL 与令牌过期一致）。
     - **已存在**：拒绝请求，返回 403。
   - 从 `resource_id` 校验与请求参数中的资源 ID 一致，防止越权。
   - 执行业务逻辑（内部可能调用任意大模型）。

### 4.2 “随用随销”的实现

Redis 原子操作保证并发安全：

```java
Boolean absent = redisTemplate.opsForValue()
    .setIfAbsent("jti:" + jti, "1", Duration.ofMinutes(2));
if (Boolean.FALSE.equals(absent)) {
    throw new OneTimeTokenUsedException("Token already used");
}
```

令牌一旦被校验通过，立即标记为已使用，任何后续重用都会被拒绝。

---
## 5. 组件实现细节

### 5.1 授权服务器（Spring Authorization Server）

**客户端注册**：每个 Agent 作为一个注册客户端，分配基础 scope 并允许 Token Exchange 授权类型。

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient agentA = RegisteredClient.withId("agent-a")
        .clientId("agent-alpha")
        .clientSecret("{noop}secret")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
        .authorizationGrantType(AuthorizationGrantType.TOKEN_EXCHANGE)
        .scope("agent:summarize")
        .scope("doc:read")
        .scope("doc:summarize")
        .build();
    return new InMemoryRegisteredClientRepository(agentA);
}
```

**定制 JWT 声明**：通过 `OAuth2TokenCustomizer<JwtEncodingContext>` 注入自定义字段。

```java
@Component
public class CustomJwtCustomizer implements OAuth2TokenCustomizer<JwtEncodingContext> {

    @Override
    public void customize(JwtEncodingContext context) {
        if (AuthorizationGrantType.TOKEN_EXCHANGE.equals(context.getAuthorizationGrantType())) {
            // 从授权请求参数中提取 resource 和 single_use
            Map<String, Object> params = context.getAuthorization().getAttributes();
            String resourceId = (String) params.get("resource");
            Boolean singleUse = Boolean.valueOf((String) params.get("single_use"));

            context.getClaims().claim("resource_id", resourceId);
            context.getClaims().claim("single_use", singleUse);
            // scope 已由框架写入，可追加细粒度
        }
    }
}
```

**Token Exchange 端点额外参数**：需扩展默认的 `OAuth2TokenExchangeAuthenticationConverter` 以支持 `resource`、`single_use` 等自定义参数，并将其放入 `Authentication` 的 attributes 中供后续使用。

### 5.2 Agent 作为资源服务器（Agent B）

**依赖**：`spring-boot-starter-oauth2-resource-server`

**安全配置**：

```java
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JtiOneTimeFilter jtiFilter) {
        http
            .addFilterAfter(jtiFilter, BearerTokenAuthenticationFilter.class)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/docs/**").hasAuthority("SCOPE_doc:read")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt());
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthorities = new JwtGrantedAuthoritiesConverter();
        grantedAuthorities.setAuthoritiesClaimName("scope");
        grantedAuthorities.setAuthorityPrefix("SCOPE_");
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthorities);
        return converter;
    }
}
```

**一次性令牌过滤器**：

```java
@Component
public class JtiOneTimeFilter extends OncePerRequestFilter {
    private final StringRedisTemplate redisTemplate;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            Jwt jwt = jwtAuth.getToken();
            Boolean singleUse = jwt.getClaim("single_use");
            String jti = jwt.getId();
            if (Boolean.TRUE.equals(singleUse) && jti != null) {
                Boolean success = redisTemplate.opsForValue()
                    .setIfAbsent("jti:" + jti, "1", Duration.ofMinutes(2));
                if (Boolean.FALSE.equals(success)) {
                    response.setStatus(HttpStatus.FORBIDDEN.value());
                    response.getWriter().write("One-time token already used");
                    return;
                }
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

**业务 API 示例**：

```java
@RestController
public class SummarizeController {

    private final ChatModel primaryModel; // 通过构造注入，决定使用哪个模型

    public SummarizeController(@Qualifier("dashscopeChatModel") ChatModel primaryModel) {
        this.primaryModel = primaryModel;
    }

    @PreAuthorize("hasAuthority('SCOPE_doc:summarize')")
    @PostMapping("/docs/{docId}/summarize")
    public Mono<String> summarize(@PathVariable String docId,
                                  @RequestBody String content,
                                  JwtAuthenticationToken auth) {
        String resourceId = auth.getToken().getClaimAsString("resource_id");
        if (!docId.equals(resourceId)) {
            return Mono.error(new AccessDeniedException("Resource ID mismatch"));
        }
        // 内部调用大模型（阿里云、OpenAI 等）
        return ChatClient.create(primaryModel)
            .prompt(new Prompt("请总结以下内容：" + content))
            .call()
            .content();
    }
}
```

### 5.3 Agent 作为客户端（Agent A）

Agent A 使用 `WebClient` 配合 `ServletOAuth2AuthorizedClientExchangeFilterFunction` 自动获取令牌并携带。

**基础客户端凭证调用**（A2A 不涉及用户时）：

```java
@Bean
WebClient webClient(ReactiveClientRegistrationRepository clientRegs,
                    ReactiveOAuth2AuthorizedClientService authClientService) {
    var filter = new ServletOAuth2AuthorizedClientExchangeFilterFunction(
            new AuthorizedClientServiceOAuth2AuthorizedClientManager(clientRegs, authClientService));
    filter.setDefaultOAuth2AuthorizedClient(true);
    filter.setDefaultClientRegistrationId("agent-alpha");
    return WebClient.builder().apply(filter.oauth2Configuration()).build();
}
```

**Token Exchange 委派调用**（代表用户时）：需自定义 `OAuth2AuthorizedClientProvider`，使用 `WebClient` 直接调用授权服务器进行令牌交换，获得细粒度一次性令牌后再发起业务请求。

```java
public class TokenExchangeProvider implements OAuth2AuthorizedClientProvider {
    private final WebClient authServerWebClient;

    @Override
    public OAuth2AuthorizedClient authorize(OAuth2AuthorizationContext context) {
        String subjectToken = context.getAttribute("subject_token");
        String resource = context.getAttribute("resource_id");
        // 构造 POST /oauth2/token 表单
        // ...
        // 解析响应，构建 OAuth2AuthorizedClient 返回
    }
}
```

配置文件中注册多个客户端（如果 Agent 需以不同身份调用多个下游）。

### 5.4 多模型提供商集成

每个 Agent 独立选择大模型，通过 Spring AI 的 `ChatModel` 抽象注入。

**依赖**（按需引入）：

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

**配置示例**（`application.yml`）：

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
```

注入时通过 `@Qualifier` 选择所需模型，也可根据业务逻辑动态切换（如成本、用户层级等）。模型的选择完全是 Agent 内部实现细节，不影响 A2A 的权限体系。

---
## 6. 用户上下文传播

在 Token Exchange 生成的令牌中，`act` (actor) 声明携带了原始用户信息：

```json
{
  "sub": "agent-alpha",
  "act": {
    "sub": "user-alice",
    "scope": "user:basic"
  },
  "scope": "doc:summarize:report-123",
  "resource_id": "report-123",
  "single_use": true,
  "jti": "abc123...",
  "exp": ...
}
```

资源服务器可以从 `act.sub` 获取原始用户标识，用于审计日志或行级数据过滤，确保操作可追溯。

---
## 7. 安全加固

- **令牌有效期**：基础 `client_credentials` 令牌 5~15 分钟；一次性令牌 1~2 分钟。
- **最小权限**：每个 Agent 注册时仅分配必要 scope，Token Exchange 生成的 scope 严格局限于所需资源。
- **传输安全**：A2A 通信启用 HTTPS，并可在内部部署 mTLS 增强防护。
- **API Key 保护**：大模型 API Key 存放于环境变量或密钥管理服务（Vault），不硬编码，不暴露给客户端。
- **审计**：记录所有 A2A 请求的 `jti`、调用方 Agent、原始用户、操作资源、模型消耗等，便于安全审计和计费。
- **防重放**：由 `single_use` + Redis 保证一次性令牌不可重用。

---
## 8. 总结

本方案完全基于 Spring 生态，构建了一套满足企业级 A2A 交互的权限体系：

- **认证**：OAuth2 客户端凭证与 Token Exchange 相结合，支持纯服务调用和用户委派。
- **授权**：三层 Scope 实现从功能到临时资源的细粒度控制。
- **一次性令牌**：通过 JWT 的 `jti` 和 Redis 原子检查实现“随用随销”，防范重放。
- **模型解耦**：去除 AI 网关，每个 Agent 自行集成任意大模型，权限与模型无关，扩展灵活。
- **安全合规**：API Key 隔离、用户上下文传播、审计日志等机制保障整体安全性。

该设计可直接应用于多智能体协作平台，既保留了 Spring AI Alibaba 的便捷性，又确保了异构模型下的安全可控。