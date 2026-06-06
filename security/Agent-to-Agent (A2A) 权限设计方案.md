
> **基于**：Spring AI Alibaba A2A + Nacos + Spring Authorization Server + Spring Security 6.3  
> **核心**：OAuth2 Token Exchange (RFC 8693) + 一次性令牌 + 模块化权限服务  
> **可选扩展**：mTLS、DPoP（配置开关）  

## 1. 概述

本文档定义一套面向多智能体（Multi-Agent）系统的 A2A 权限设计方案，满足以下核心需求：

- **Agent 间安全互操作**：实现 Agent 之间的安全认证与通信，防止未授权调用。
- **细粒度动态授权**：支持资源实例级（如 `doc:summarize:report-123`）、临时性、一次性的操作权限。
- **用户委派（User Delegation）**：Agent 可代表最终用户执行操作，下游能追溯到原始用户并校验数据权限。
- **无 AI 网关**：每个 Agent 独立持有大模型 API Key，通过 Spring AI Alibaba 直连模型服务商（如阿里云 DashScope、OpenAI 等）。
- **权限逻辑集中复用**：通过独立的权限模块（`common-permission`）供所有 Agent 调用，避免代码重复。
- **多模型提供商自由选择**：不同 Agent 可使用不同的大模型，权限体系与模型完全解耦。

本方案同时覆盖**封闭环境**（内网部署，必须实现核心机制）和**开放环境**（对外暴露接口或调用外部 Agent，按需开启扩展）两种场景。

> **背景**：A2A（Agent-to-Agent）协议是由 Google 在 2025 年 4 月发起、随后移交给 Linux 基金会（LF AI & Data 旗下）管理的开放标准。**该协议于 2026 年 3 月 10 日发布 v1.0.0 正式版本**，成为首个 Agent 间通信的开放标准。截至 2026 年 6 月，已有 150+ 家组织支持，Google Cloud、Microsoft Azure、AWS 均已集成，并进入企业级生产部署阶段。A2A 协议对认证方式保持中立，支持通过 HTTP 头承载 OAuth2 Bearer Token、API Key 或 mTLS 证书等主流方案。本方案基于 A2A 协议的互通性设计，在企业内部网络中构建标准化的多智能体安全通信底座。

### 1.1 与 A2A 协议标准的关系

| A2A 标准要求 | 本方案实现 |
|---|---|
| 认证通过 HTTP 头（如 Bearer Token）承载 | JWT Bearer Token（RFC 6750），由 Spring Authorization Server 颁发 |
| 支持 Token Exchange（OAuth 2.0）增强安全 | 完整实现 RFC 8693 Token Exchange，支持用户委派和跨 Agent 委托 |
| 支持 DPoP 或 mTLS 作为可选的持有证明方式 | DPoP 和 mTLS 均通过配置开关预留扩展点 |
| AgentCard 声明安全方案（基于 OpenAPI 3.0 Security Scheme Object） | Agent 启动时注册 AgentCard 到 Nacos，包含完整安全元数据 |
| 传输层必须使用 TLS | 内网 HTTPS 必须，开放环境 mTLS 可选 |

### 1.2 设计原则

- **默认安全**：封闭环境必须启用 OAuth2 + Token Exchange + 一次性令牌（随用随销），防重放是核心要求。
- **可插拔扩展**：mTLS、DPoP、动态策略（OPA）、速率限制等扩展功能通过配置开关控制，默认关闭，不增加核心复杂度。
- **最小权限原则**：基于 ACL 静态表 + Scope 前缀匹配 + 资源实例校验实现多层权限控制。
- **无网关依赖**：每个 Agent 作为独立的资源服务器，通过 JWT 认证 + 一次性令牌过滤器直接保护 A2A 端点，与 Spring AI Alibaba + Nacos 完全兼容。
- **权限逻辑模块化**：用户对资源的细粒度权限判断，通过 `common-permission` 模块集中封装，所有 Agent 复用。
- **兼容开放标准**：认证方案严格遵循 A2A 协议规范，确保与外部 Agent（如通过 AgentCard 获取元数据的第三方 Agent）的互操作性。
---
## 2. 总体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Nacos (A2A Registry)                     │
│               (Agent 服务注册与发现，AgentCard 元数据)            │
└─────────────────────────────────────────────────────────────────┘
         │ 注册/发现 Agent Card      │
         ▼                           ▼
┌───────────────┐   Token Exchange + 代理令牌   ┌───────────────┐
│   Agent A     │ ─────────────────────────────▶ │   Agent B     │
│ (调用方/客户端)│  (A2A 协议端点 /message)       │ (资源服务器)   │
└───────┬───────┘                                └───────┬───────┘
        │                                                │
        │ 1. client_id+secret + (可选 subject_token)    │ 持有模型 API Key
        │    → 代理令牌 (含 sub, act, scope, resource)  │ (DashScope/OpenAI等)
        ▼                                                ▼
┌─────────────────────────────┐              ┌─────────────────────────┐
│ Spring Authorization Server │              │  大模型提供商            │
│ - Token Exchange 端点        │              │  (无需 AI 网关)          │
│ - ACL 静态授权表（前缀匹配）  │              └─────────────────────────┘
│ - 用户同意记录（可选）        │
│ - JWT 颁发与定制              │
└─────────────┬───────────────┘
              │
              │ 用户认证（若涉及用户委派）
              ▼
         ┌─────────┐
         │  用户   │
         └─────────┘
```

**核心组件**：
- **授权服务器**（Spring Authorization Server）：管理 Agent 客户端、ACL 表（支持 scope 前缀匹配），提供 Token Exchange 端点（RFC 8693），生成并签发代理令牌。
- **Nacos**：A2A 注册中心。Nacos 3.1.0 引入 A2A 注册中心功能，**生产环境推荐 Nacos 3.2.1 或更高版本**；Spring AI Alibaba 通过集成 Nacos，提供了开箱即用的 Agent 注册、发现与负载均衡能力。
- **每个 Agent**：
  - 作为 **OAuth2 客户端**：调用 Token Exchange 获取代理令牌。
  - 作为 **资源服务器**：验证 JWT，执行一次性令牌、audience、资源 ID 校验。
  - 内置 **权限模块（`common-permission`）** ：用于校验用户对资源的操作权限。
- **Redis**：存储一次性令牌的 `jti`（JWT ID），实现防重放。
- **权限模块**：公共 JAR 包，封装用户-资源权限判断逻辑（集中式权限服务，供所有 Agent 调用）。
---
## 3. 权限模型：三层 Scope + 前缀匹配

### 3.1 三层 Scope 定义

| 层级 | 格式 | 示例 | 说明 |
|---|---|---|---|
| **通用能力** | `agent:{capability}` | `agent:rag`、`agent:summarize` | 表示 Agent 提供的基础功能，用于粗粒度授权 |
| **资源类型** | `{resource}:{action}` | `doc:read`、`doc:summarize` | 对某类资源的操作权限，是 Scope 前缀的匹配基准 |
| **临时细粒度** | `{resource}:{action}:{instance-id}` | `doc:summarize:report-123` | 针对具体资源实例的操作许可，有效期短且可一次性使用 |

### 3.2 Agent 间调用授权表（ACL 静态表 + 前缀匹配）

**核心设计**：ACL 表存储允许的 **Scope 前缀**，授权服务器采用 **前缀匹配** 算法验证调用方的请求 Scope。

```sql
CREATE TABLE a2a_acl (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    caller_client_id VARCHAR(64) NOT NULL,        -- 调用方 Agent ID
    target_client_id VARCHAR(64) NOT NULL,        -- 目标 Agent ID
    allowed_scope_prefixes VARCHAR(512) NOT NULL, -- Scope 前缀，逗号分隔，如 "doc:summarize,doc:read"
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY (caller_client_id, target_client_id)
);
```

**前缀匹配规则**：
- 请求 Scope `doc:summarize:report-123` 匹配前缀 `doc:summarize`，通过。
- 请求 Scope `doc:summarize` 匹配前缀 `doc:summarize`，通过。
- 请求 Scope `doc:read:invoices-456` 不匹配 `doc:summarize`，拒绝。
- 前缀列表为空或为空字符串，表示不允许任何调用。

> **说明**：此匹配规则为方案的自定义扩展，非 OAuth2 标准要求。跨组织集成时需双方事先约定相同的匹配逻辑。

**验证算法**（授权服务器 Token Exchange 时执行）：

```java
public boolean isScopeAllowed(String requestedScope, String allowedPrefixesStr) {
    if (allowedPrefixesStr == null || allowedPrefixesStr.trim().isEmpty()) {
        return false;
    }
    String[] prefixes = allowedPrefixesStr.split(",");
    for (String prefix : prefixes) {
        prefix = prefix.trim();
        if (requestedScope.equals(prefix) || requestedScope.startsWith(prefix + ":")) {
            return true;
        }
    }
    return false;
}
```

### 3.3 用户委派与同意表（可选）

若需要 Agent 代表最终用户操作，添加用户同意表。设计需支持撤销和重新授权：

```sql
CREATE TABLE user_consent (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL,
    client_id VARCHAR(64) NOT NULL,
    scope_prefix VARCHAR(256) NOT NULL,   -- Scope 前缀，如 "doc:summarize"
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP NULL,            -- 撤销时间，NULL 表示有效
    UNIQUE KEY (user_id, client_id, scope_prefix, revoked_at) -- 允许多条历史，但最多一条有效
);
```

用户同意可在首次使用时通过 OAuth2 授权页面收集。Token Exchange 时，授权服务器查询该表，确认用户是否授予目标 Agent 代理指定 Scope 前缀（需 `revoked_at IS NULL`）。

### 3.4 通用能力与资源类型 Scope 的关系

- **本质区别**：`agent:summarize` 表示“Agent A 具有摘要能力”；`doc:summarize` 表示“对文档执行摘要操作”。两者是不同维度的授权。
- **ACL 中的配置建议**：若目标 Agent 要求调用方必须具有 `agent:summarize` 能力，则 ACL 表中应配置 `agent:summarize` 前缀；调用方请求 `doc:summarize:report-123` 时，授权服务器需同时匹配两个前缀（`agent:summarize` 和 `doc:summarize`），或简化设计为仅匹配 `doc:summarize`（语义上已隐含“Agent B 会被要求执行摘要操作”）。
- **实现建议**：在封闭环境中，ACL 表中通常只需要配置资源类型 Scope（如 `doc:summarize`）。通用能力 Scope 在客户端注册时作为 client scope 授予，不作为 ACL 匹配项。若需要强制检查能力，可将 `agent:summarize` 和 `doc:summarize` 同时配置在 `allowed_scope_prefixes` 中，调用方请求时必须同时满足。
---
## 4. 核心流程：Token Exchange 获取代理令牌

### 4.1 场景一：纯 Agent 直连（无最终用户）

Agent A 需要调用 Agent B 的某个功能（例如文档总结），无需代表任何用户。

**请求示例**：
```http
POST /oauth2/token HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&client_id=agent-a
&client_secret=...   # ⚠️ 需要进行 URL 编码
&scope=doc:summarize:report-123
&audience=agent-b
&single_use=true
&resource=report-123
```

> **注意**：`client_secret` 如果包含特殊字符（如 `+`、`/`、`=`），必须使用 `URLEncoder.encode(secret, StandardCharsets.UTF_8)` 进行编码。

**授权服务器处理**（Spring Security 6.3+ 原生支持 RFC 8693）：
1. 验证 `client_id` + `client_secret`。
2. 查询 ACL 表，对 `(caller=agent-a, target=agent-b)` 的 `allowed_scope_prefixes` 进行前缀匹配，判断 `doc:summarize:report-123` 是否被允许。
3. 生成 JWT（`sub=agent-a`，无 `act` 声明），包含 `scope`、`aud`、`resource_id`（扩展声明）、`single_use`（扩展声明）、`jti`、`exp`（≤2 分钟）。
4. 返回令牌。

### 4.2 场景二：用户委派（Agent 代表最终用户）

用户 Alice 向 Agent A 提交请求“总结 report-123”，并在请求中携带自己的 `user_token`（已通过 OAuth2 授权码流程获得）。

**请求示例**：
```http
POST /oauth2/token HTTP/1.1
Host: auth-server.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&client_id=agent-a
&client_secret=...
&subject_token=eyJhbGciOi...
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&scope=doc:summarize:report-123
&audience=agent-b
&single_use=true
&resource=report-123
```

> **`subject_token_type` 参数说明**：RFC 8693 定义了多种 token 类型，常用值包括：
> - `urn:ietf:params:oauth:token-type:access_token`（Access Token）
> - `urn:ietf:params:oauth:token-type:refresh_token`
> - `urn:ietf:params:oauth:token-type:jwt`
> - `urn:ietf:params:oauth:token-type:saml2`
> 本方案默认使用 `access_token` 类型。

**授权服务器处理**：
1. 验证 Agent A 的客户端凭证。
2. 验证 `subject_token` 有效性。
3. 查询 `user_consent` 表（`revoked_at IS NULL`），确认用户 Alice 已授权 Agent A 代理 `doc:summarize` 权限前缀（若未授权，可触发动态同意流程）。
4. 查询 ACL 表，对 `allowed_scope_prefixes` 进行前缀匹配，确认 Agent A 有权请求该 Scope 作用在 Agent B 上。
5. 生成 JWT，包含：

   ```json
   {
     "sub": "agent-a",
     "act": { "sub": "alice", "iss": "https://auth.example.com" },
     "scope": "doc:summarize:report-123",
     "aud": "agent-b",
     "resource_id": "report-123",
     "single_use": true,
     "jti": "unique-id-123",
     "exp": 1718000120,
     "delegation_remaining": 1,
     "trace_id": "trace-abc-123"
   }
   ```

   > 注：`resource_id`、`single_use`、`delegation_remaining`、`delegation_chain`、`trace_id` 均为**扩展声明**（非 RFC 8693 标准字段），调用双方需约定格式并在验签代码中正确处理。`delegation_chain` 用于多跳委托，实现方式可参考 RFC 8693 中嵌套 `act` 的做法。

6. 返回令牌。

**关键说明**：Agent A **不需要**预先为自己申请一个 `client_credentials` 令牌。Token Exchange 请求直接使用 `client_id` + `client_secret` 作为身份证明，这是 RFC 8693 的标准用法。最终代理令牌同时承载调用方身份（`sub`）和用户身份（`act`），调用 Agent B 时只需携带这一个令牌。

### 4.3 一次性令牌防重放

资源服务器（Agent B）在验证时，对 `single_use=true` 的令牌使用 Redis 原子操作：

```java
Boolean absent = redisTemplate.opsForValue()
    .setIfAbsent("jti:" + jti, "1", Duration.ofMinutes(2));
if (Boolean.FALSE.equals(absent)) {
    throw new OneTimeTokenUsedException();
}
```

### 4.4 Scope 前缀匹配的最终验证逻辑

在授权服务器的 JWT 生成过程中，必须执行 Scope 前缀匹配：

```java
public boolean isScopeAuthorized(String callerId, String targetId, String requestedScope) {
    AclEntry acl = aclRepository.findByCallerAndTarget(callerId, targetId);
    if (acl == null) return false;
    String[] allowedPrefixes = acl.getAllowedScopePrefixes().split(",");
    for (String prefix : allowedPrefixes) {
        if (requestedScope.equals(prefix.trim()) || 
            requestedScope.startsWith(prefix.trim() + ":")) {
            return true;
        }
    }
    return false;
}
```
---
## 5. 资源服务器（Agent B）安全实现

### 5.1 JWT 认证与一次性令牌过滤器

每个 Agent 作为资源服务器，保护 A2A 端点。A2A v1.0 协议标准端点路径为 `/message`（实际部署中可能挂载在上下文根下，如 `/a2a/message`）。此外，`/.well-known/agent.json` 端点应公开访问，供其他 Agent 发现元数据。

```java
@Component
public class A2AJwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtDecoder jwtDecoder;
    private final StringRedisTemplate redisTemplate;
    private final String myClientId;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String path = request.getRequestURI();
        
        // 公开端点：AgentCard 无需认证
        if (path.equals("/.well-known/agent.json") || path.equals("/agent-card")) {
            chain.doFilter(request, response);
            return;
        }
        
        // 仅拦截 A2A 协议端点（/message、/task 等）
        if (!path.matches("^/(message|task/.*)$")) {
            chain.doFilter(request, response);
            return;
        }
        
        // 1. 提取 JWT
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Missing or invalid Authorization header");
            return;
        }
        String token = authHeader.substring(7);
        
        // 2. 验证 JWT（签名、有效期、aud）
        Jwt jwt;
        try {
            jwt = jwtDecoder.decode(token);
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Invalid JWT: " + e.getMessage());
            return;
        }
        
        // 3. 校验 audience 是否为本 Agent
        String aud = jwt.getClaimAsString("aud");
        if (!myClientId.equals(aud)) {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("Invalid audience");
            return;
        }
        
        // 4. 一次性令牌检查（扩展声明）
        Boolean singleUse = jwt.getClaimAsBoolean("single_use");
        String jti = jwt.getId();
        if (Boolean.TRUE.equals(singleUse) && jti != null) {
            Boolean success = redisTemplate.opsForValue()
                .setIfAbsent("jti:" + jti, "1", Duration.ofMinutes(2));
            if (Boolean.FALSE.equals(success)) {
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                response.getWriter().write("One-time token already used");
                return;
            }
        }
        
        // 5. 存入 SecurityContext
        SecurityContextHolder.getContext().setAuthentication(
            new JwtAuthenticationToken(jwt, extractAuthorities(jwt))
        );
        
        chain.doFilter(request, response);
    }
}
```

### 5.2 业务 API 中的权限校验（用户资源权限）

Agent B 在处理具体请求时，需要校验用户（如果有）是否有权操作指定的资源。此逻辑通过**独立权限模块（`common-permission`）** 完成。

```java
@RestController
public class SummarizeController {
    
    @Autowired
    private PermissionChecker permissionChecker;  // 来自 common-permission 模块
    
    @PostMapping("/docs/{docId}/summarize")
    public Mono<String> summarize(@PathVariable String docId,
                                  @RequestBody String content,
                                  JwtAuthenticationToken auth) {
        Jwt jwt = auth.getToken();
        
        // 校验 Scope
        if (!jwt.getClaimAsStringList("scope").contains("doc:summarize:" + docId)) {
            return Mono.error(new AccessDeniedException("Missing scope"));
        }
        
        // 校验资源 ID 匹配（防止水平越权）
        String resourceId = jwt.getClaimAsString("resource_id");
        if (!docId.equals(resourceId)) {
            return Mono.error(new AccessDeniedException("Resource ID mismatch"));
        }
        
        // 提取用户身份（若有委派）
        Map<String, Object> act = jwt.getClaim("act");
        String userId = (act != null) ? (String) act.get("sub") : null;
        
        // 调用权限模块判断用户是否有权执行 summarize 操作
        if (userId != null && !permissionChecker.hasPermission(userId, docId, "summarize")) {
            return Mono.error(new AccessDeniedException("User has no permission to summarize this document"));
        }
        
        // 调用大模型...
        return ChatClient.create(model)
            .prompt("请总结以下内容：" + content)
            .call()
            .content();
    }
}
```

> **错误响应格式建议**：推荐使用 **Problem Details for HTTP APIs (RFC 9457)** 返回标准化错误结构，例如：
> ```json
> {
>   "type": "https://example.com/errors/access-denied",
>   "title": "Access Denied",
>   "status": 403,
>   "detail": "User has no permission to summarize this document"
> }
> ```
---
## 6. 客户端（Agent A）实现：获取代理令牌

Agent A 需要实现 `TokenExchangeService`，用于向授权服务器换取代理令牌。

```java
@Service
public class TokenExchangeService {
    
    private final WebClient authServerWebClient;
    private final String clientId;
    private final String clientSecret;
    
    public TokenExchangeService(
            @Value("${oauth2.auth-server-url}") String authServerUrl,
            @Value("${oauth2.client-id}") String clientId,
            @Value("${oauth2.client-secret}") String clientSecret) {
        this.authServerWebClient = WebClient.builder()
            .baseUrl(authServerUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_FORM_URLENCODED_VALUE)
            .build();
        this.clientId = clientId;
        this.clientSecret = clientSecret;
    }
    
    public String exchangeToken(String userToken, String scope, String resourceId, String targetAudience) {
        MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
        form.add("grant_type", "urn:ietf:params:oauth:grant-type:token-exchange");
        form.add("client_id", clientId);
        // 注意：secret 需要 URL 编码
        form.add("client_secret", URLEncoder.encode(clientSecret, StandardCharsets.UTF_8));
        if (userToken != null) {
            form.add("subject_token", userToken);
            form.add("subject_token_type", "urn:ietf:params:oauth:token-type:access_token");
        }
        form.add("scope", scope);
        form.add("audience", targetAudience);
        form.add("single_use", "true");
        form.add("resource", resourceId);
        
        TokenResponse response = authServerWebClient.post()
            .uri("/oauth2/token")
            .bodyValue(form)
            .retrieve()
            .onStatus(HttpStatusCode::isError, clientResponse ->
                clientResponse.bodyToMono(String.class)
                    .flatMap(errorBody -> Mono.error(new RuntimeException("Token exchange failed: " + errorBody))))
            .bodyToMono(TokenResponse.class)
            .block();
        
        return response.getAccessToken();
    }
}
```

在调用 `A2aRemoteAgent` 之前，先获取代理令牌，并注入到 HTTP 请求头中。

```java
@Configuration
public class A2aClientConfig {
    
    @Bean
    public A2aRemoteAgent agentBRemoteAgent(
            AgentCardProvider agentCardProvider,
            TokenExchangeService tokenExchangeService) {
        
        WebClient webClient = WebClient.builder()
            .filter((request, next) -> {
                // 从当前请求上下文获取用户令牌（若有）
                String userToken = extractUserTokenFromCurrentContext();
                String proxyToken = tokenExchangeService.exchangeToken(
                    userToken, "doc:summarize:report-123", "report-123", "agent-b");
                
                ClientRequest newRequest = ClientRequest.from(request)
                    .header(HttpHeaders.AUTHORIZATION, "Bearer " + proxyToken)
                    .build();
                return next.exchange(newRequest);
            })
            .build();
        
        return new A2aRemoteAgent("agent-b", agentCardProvider, webClient);
    }
}
```
---
## 7. 独立权限模块（`common-permission`）设计

### 7.1 模块结构

```
common-permission/
├── pom.xml
├── src/main/java/com/example/permission/
│   ├── PermissionChecker.java                   // 接口
│   ├── PermissionCheckerImpl.java               // 实现（直接查库，可选缓存）
│   ├── config/PermissionAutoConfiguration.java  // Spring Boot 自动配置
│   └── model/UserDocPermission.java             // 实体
└── src/main/resources/db/migration/             // 数据库迁移脚本
```

### 7.2 接口定义

```java
public interface PermissionChecker {
    boolean hasPermission(String userId, String resourceId, String operation);
}
```

### 7.3 实现（直接查库 + 缓存）

**依赖**：需在模块的 `pom.xml` 中添加 `spring-boot-starter-cache` 和 Caffeine 依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

**配置类**（启用缓存）：

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("userDocPerm");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .maximumSize(10000));
        return cacheManager;
    }
}
```

**实现类**：

```java
@Service
@ConditionalOnMissingBean(PermissionChecker.class)
public class PermissionCheckerImpl implements PermissionChecker {
    
    @Autowired
    private UserDocPermissionRepository repository;
    
    @Cacheable(value = "userDocPerm", key = "#userId + ':' + #resourceId + ':' + #operation")
    @Override
    public boolean hasPermission(String userId, String resourceId, String operation) {
        return repository.existsByUserIdAndResourceIdAndOperation(userId, resourceId, operation);
    }
}
```

### 7.4 数据库表

```sql
CREATE TABLE user_doc_permission (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL,
    resource_id VARCHAR(128) NOT NULL,
    operation VARCHAR(32) NOT NULL,
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY (user_id, resource_id, operation)
);

CREATE INDEX idx_user_resource ON user_doc_permission(user_id, resource_id);
```

### 7.5 Agent 集成

每个 Agent 在 `pom.xml` 中引入该模块：

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>common-permission</artifactId>
    <version>1.0.0</version>
</dependency>
```

Spring Boot 自动配置会扫描到 `PermissionChecker` 的实现，直接注入使用。模块化权限服务的优势在于：
- **代码复用**：权限判断逻辑只需写一次，所有 Agent 共享。
- **集中维护**：权限规则变更时，只需修改模块并升级 JAR 版本。
- **无额外网络开销**：本地方法调用，不引入远程调用延迟。
- **可平滑演进**：未来可拆分为独立微服务，接口定义保持不变。
---
## 8. 与 Spring AI Alibaba A2A + Nacos 的集成

### 8.1 技术栈版本要求

| 组件 | 最低版本要求 | 说明 |
|---|---|---|
| **Nacos** | 3.1.0（最低），**3.2.1（生产推荐）** | 3.1.0 引入 A2A 注册中心功能；3.2.1 增强了 AI Registry 支持，包含 AgentCard v1 协议兼容性 |
| **Spring AI Alibaba** | 1.0.0.4+ | A2A 相关集成版本，建议使用 1.0.0.4 或更新版本（请参照官方公告） |
| **Spring Authorization Server** | 1.5.7（或更高） | **重要**：CVE-2026-22752 影响版本 1.3.x–1.5.6，1.5.7 已修复。若使用 1.4.x，补丁仅通过付费商业支持提供，建议直接升级到 1.5.7+ |
| **Spring Security** | 6.3+ | 支持 RFC 8693 Token Exchange 和 DPoP |
| **Redis** | 6.0+ | 用于一次性令牌存储 |

### 8.2 Agent 注册与发现：AgentCard 规范

Spring AI Alibaba 的 A2A 实现包含三个核心组件：
- **A2A Server**：将本地 ReactAgent 暴露为 A2A 服务（端点 `/message` 等）。
- **A2A Registry**：Agent 注册中心，支持 Nacos。
- **A2A Discovery**：Agent 发现机制，支持 Nacos（负责从注册中心获取 Agent 实例列表及负载均衡）。

生产环境中，A2A Registry 支持服务端健康检查：Nacos 定期探测 Agent 的健康状态，自动将不健康的实例从发现列表中剔除。同时，Nacos 客户端内置负载均衡策略（随机、轮询等），可根据配置分发请求。

**AgentCard 安全方案声明**（基于 OpenAPI 3.0 Security Scheme Object）：

Agent 启动时，`A2AServer` 自动发布以下格式的 AgentCard 到 Nacos：

```json
{
  "agentId": "agent-b",
  "endpoint": "https://agent-b.example.com",  // 实际请求时拼接 /message
  "securitySchemes": {
    "oauth2": {
      "type": "oauth2",
      "flows": {
        "clientCredentials": {
          "tokenUrl": "https://auth-server/oauth2/token",
          "scopes": {
            "doc:summarize": "Summarize documents",
            "doc:read": "Read documents"
          }
        }
      }
    }
  },
  "security": [
    { "oauth2": ["doc:summarize", "doc:read"] }
  ]
}
```

- `securitySchemes`：定义可用的安全方案（OAuth2、API Key、HTTP Bearer 等）。
- `security`：声明此 Agent 实际要求的安全方案名称及所需 scope 列表。

调用方通过 `AgentCardProvider` 从 Nacos 获取目标 Agent 的 AgentCard，解析 `security` 字段，选择对应的认证方式（例如使用 OAuth2 client credentials 流或 Token Exchange）。

**Nacos 安全配置提醒**：生产环境中 Nacos 必须启用认证（用户名/密码或 Token），防止未授权访问注册中心。

```yaml
nacos:
  auth:
    enabled: true
    username: nacos
    password: your-strong-password
```

### 8.3 调用时注入代理令牌

```java
@Bean
public A2aRemoteAgent agentBRemoteAgent(
        AgentCardProvider agentCardProvider,
        TokenExchangeService tokenExchangeService) {
    
    WebClient webClient = WebClient.builder()
        .filter((request, next) -> {
            String userToken = extractUserTokenFromCurrentContext();
            String proxyToken = tokenExchangeService.exchangeToken(
                userToken, "doc:summarize:report-123", "report-123", "agent-b");
            ClientRequest newRequest = ClientRequest.from(request)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + proxyToken)
                .build();
            return next.exchange(newRequest);
        })
        .build();
    
    return new A2aRemoteAgent("agent-b", agentCardProvider, webClient);
}
```
---
## 9. 可选扩展：mTLS 与 DPoP

> 所有扩展通过配置开关控制，默认关闭，不增加核心复杂度。开放环境按需启用。

### 9.1 mTLS（双向 TLS）

**作用**：在传输层强制客户端出示证书，增强身份认证，与 DPoP 互补。

**配置开关**：
```yaml
a2a:
  security:
    mtls:
      enabled: false          # 默认关闭
      server:
        key-store: classpath:server-keystore.p12
        key-store-password: ${KEYSTORE_PASSWORD}
        trust-store: classpath:truststore.jks
        trust-store-password: ${TRUSTSTORE_PASSWORD}
        client-auth: need     # 强制客户端证书
```

**服务端配置**（Spring Boot + Tomcat）：

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: ${a2a.security.mtls.server.key-store}
    key-store-password: ${a2a.security.mtls.server.key-store-password}
    trust-store: ${a2a.security.mtls.server.trust-store}
    trust-store-password: ${a2a.security.mtls.server.trust-store-password}
    client-auth: ${a2a.security.mtls.server.client-auth}
```

**服务端提取客户端证书**：

```java
@Component
public class MtlsCertificateExtractor {
    public X509Certificate extractClientCertificate(HttpServletRequest request) {
        X509Certificate[] certs = (X509Certificate[]) request.getAttribute("jakarta.servlet.request.X509Certificate");
        if (certs != null && certs.length > 0) {
            return certs[0];  // 客户端证书链的第一个
        }
        return null;
    }
}
```

然后可将证书的 CN 或 SAN 映射为 Agent 的 `client_id`，用于认证。

**客户端配置**：自定义 `WebClient` Bean，配置 SSLContext 加载客户端证书和信任库。若 mTLS 未启用，使用普通 HTTPS 或 HTTP。

### 9.2 DPoP（Demonstrating Proof of Possession）

**作用**：将 Bearer Token 与客户端私钥绑定（Sender-Constrained Token），即使令牌泄露，攻击者也无法使用。

**规范引用**：DPoP 由 **RFC 9449** 定义，其中使用 `cnf`（confirmation）声明（**RFC 7800**）将公钥指纹绑定到 JWT。

**配置开关**：
```yaml
a2a:
  security:
    dpop:
      enabled: false          # 默认关闭
```

**核心流程**：
1. 客户端生成临时密钥对，将公钥（JWK）放入 DPoP Proof JWT 的 Header 中，并对 Proof 签名。
2. 授权服务器验证 Proof 签名，并将公钥指纹（`cnf.jkt`）写入 Access Token（遵循 RFC 7800）。
3. 资源服务器验证每个请求的 DPoP Proof，使用令牌中的公钥指纹验签，并校验 `ath`（Access Token Hash）。

**资源服务器端 DPoP 验证示例**（Spring Security 6.3+）：

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.oauth2ResourceServer(oauth2 -> oauth2
        .jwt()
        .jwtAuthenticationConverter(new DPoPAuthenticationConverter())
    );
    return http.build();
}
```

Spring Security 提供 `DPoPAuthenticationConverter` 自动处理 DPoP 头验证。若手动实现，需：
- 从请求头 `DPoP` 获取 Proof JWT。
- 从 Access Token 的 `cnf.jkt` 获取公钥指纹，还原公钥。
- 验证 Proof 签名、`ath`、`htm`、`htu`、`jti` 唯一性。

---

## 10. Spring Authorization Server 定制细节

### 10.1 启用 Token Exchange 并支持自定义参数

在授权服务器配置中，需注册自定义 `OAuth2TokenExchangeAuthenticationConverter` 以支持 `resource`、`single_use` 等扩展参数。

```java
@Bean
public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
    OAuth2AuthorizationServerConfigurer authorizationServerConfigurer =
        new OAuth2AuthorizationServerConfigurer();
    
    // 自定义 Token Exchange 参数转换器
    authorizationServerConfigurer.tokenEndpoint(tokenEndpoint ->
        tokenEndpoint.accessTokenRequestConverter(new CustomTokenExchangeAuthenticationConverter())
    );
    
    http.apply(authorizationServerConfigurer);
    return http.build();
}
```

`CustomTokenExchangeAuthenticationConverter` 需要从请求中读取自定义参数并放入 `Authentication` 的 `attributes` 中，供后续 `OAuth2TokenCustomizer` 使用。

```java
@Component
public class CustomJwtCustomizer implements OAuth2TokenCustomizer<JwtEncodingContext> {
    @Override
    public void customize(JwtEncodingContext context) {
        if (AuthorizationGrantType.TOKEN_EXCHANGE.equals(context.getAuthorizationGrantType())) {
            Map<String, Object> params = context.getAuthorization().getAttributes();
            String resourceId = (String) params.get("resource");
            Boolean singleUse = Boolean.valueOf((String) params.get("single_use"));
            
            context.getClaims().claim("resource_id", resourceId);
            context.getClaims().claim("single_use", singleUse);
            // 其他自定义声明...
        }
    }
}
```

### 10.2 扩展声明使用规范

在文档所有出现自定义声明的地方，必须标注为“扩展声明（非 RFC 8693 标准字段）”，并强调调用双方需事先约定格式。资源服务器在验签时，应忽略未知声明，但对已知扩展声明需按约定处理。

---
## 11. 安全加固总结（核心必须）

| 安全机制 | 实现方式 | 必须性 |
|---|---|---|
| 身份认证 | OAuth2 客户端凭证 + Token Exchange（RFC 8693） | 必须 |
| 授权 | ACL 静态表（Scope 前缀匹配）+ 三层 Scope + 资源实例校验 | 必须 |
| 防重放 | 一次性令牌（`single_use=true`）+ Redis 原子操作 | 必须 |
| 传输安全 | HTTPS（内网生产环境强制） | 必须 |
| 最小权限 | 客户端注册时仅分配必要 Scope；ACL 表按需配置 | 必须 |
| API Key 隔离 | 模型密钥存储于环境变量或 Vault，不暴露给客户端 | 必须 |
| 审计日志 | 记录 `trace_id`、调用方 Agent、目标 Agent、用户 ID（若有）、Scope、决策结果 | 必须 |
| 令牌有效期 | 基础 `client_credentials` 令牌 5~15 分钟；一次性令牌 1~2 分钟 | 必须 |

---
## 12. 扩展能力对比

| 扩展 | 配置开关 | 适用场景 | 实现复杂度 | 开放环境必要性 |
|---|---|---|---|---|
| mTLS | `a2a.security.mtls.enabled=true` | 高安全内网、跨数据中心调用 | 中（需 PKI） | 推荐 |
| DPoP | `a2a.security.dpop.enabled=true` | 开放 API 防令牌窃取 | 中（客户端需生成密钥对并签名） | 强烈推荐 |
| OPA 动态策略 | `a2a.security.opa.enabled=true` | 复杂动态授权规则 | 高（需部署 OPA 服务） | 可选 |
| 速率限制 | `a2a.security.rate-limiting.enabled=true` | 对外开放 API 防滥用 | 低（基于 Redis） | 推荐 |

---
## 13. 方案优势总结

- **完整覆盖封闭/开放环境**：核心机制可部署于内网，扩展功能按需启用，配置开关设计确保平滑演进。
- **用户委派标准化**：严格遵循 RFC 8693 Token Exchange，代理令牌同时携带 Agent 身份（`sub`）和用户身份（`act`），且 `act` 包含 `iss` 以支持跨信任域验证。
- **权限逻辑集中复用**：通过独立权限模块避免代码重复，模块化设计支持未来拆分为独立微服务。
- **无网关依赖**：每个 Agent 直接保护 A2A 端点，与 Spring AI Alibaba + Nacos 完美集成。
- **ACL 前缀匹配**：支持动态资源实例 Scope，无需预注册，兼顾安全性与灵活性。
- **行业标准对齐**：
  - A2A 协议与 Linux Foundation 管理标准一致，确保与外部 Agent 的互操作性。
  - OAuth2 Token Exchange 符合 RFC 8693，Spring Security 6.3+ 原生支持。
  - DPoP 符合 RFC 9449 和 RFC 7800，Spring Authorization Server 官方支持。
  - Nacos 3.1.0+ 提供 A2A 注册中心功能，与 Spring AI Alibaba 深度集成。
  - 错误响应推荐使用 Problem Details（RFC 9457）。

**适用场景**：企业内部多 Agent 协作平台、基于 A2A 协议的跨部门 Agent 调用、需要代表最终用户操作的安全敏感业务系统。本方案可直接落地生产环境，符合开放标准与安全最佳实践。
