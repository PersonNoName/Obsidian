# JWT 实现

## 概述

JWT (JSON Web Token) 用于在客户端和服务端之间安全地传递信息，实现无状态认证。

## JWT 结构

```
┌────────────────────────────────────────────────────────────┐
│                      JWT Token                              │
├──────────────┬───────────────┬─────────────────────────────┤
│   Header     │    Payload    │        Signature            │
│  (Base64)    │   (Base64)    │      (Base64)               │
├──────────────┼───────────────┼─────────────────────────────┤
│ alg: HS256   │ sub: username │ HMAC-SHA256(                │
│ typ: JWT     │ exp: time     │   header + "." + payload,   │
│              │ iat: time     │   secret)                   │
└──────────────┴───────────────┴─────────────────────────────┘
                        │
                        ▼
              token = header.payload.signature
```

## JwtUtils - JWT 工具类

**文件位置**: `security/JwtUtils.java`

### 核心方法

#### 生成 Token

```java
public String generateToken(UserDetails userDetails) {
    Map<String, Object> claims = new HashMap<>();
    return createToken(claims, userDetails.getUsername());
}

public String generateToken(String username) {
    Map<String, Object> claims = new HashMap<>();
    return createToken(claims, username);
}

private String createToken(Map<String, Object> claims, String subject) {
    return Jwts.builder()
            .claims(claims)
            .subject(subject)
            .issuedAt(new Date(System.currentTimeMillis()))
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey())
            .compact();
}
```

#### 解析 Token

```java
public String extractUsername(String token) {
    return extractClaim(token, Claims::getSubject);
}

public Date extractExpiration(String token) {
    return extractClaim(token, Claims::getExpiration);
}

public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
    final Claims claims = extractAllClaims(token);
    return claimsResolver.apply(claims);
}

private Claims extractAllClaims(String token) {
    return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
}
```

#### 验证 Token

```java
public Boolean validateToken(String token, UserDetails userDetails) {
    final String username = extractUsername(token);
    return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
}

public Boolean validateToken(String token) {
    try {
        extractAllClaims(token);
        return !isTokenExpired(token);
    } catch (Exception e) {
        return false;
    }
}

private Boolean isTokenExpired(String token) {
    return extractExpiration(token).before(new Date());
}
```

## JwtAuthFilter - JWT 认证过滤器

**文件位置**: `security/JwtAuthFilter.java`

### 过滤器逻辑

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain
    ) throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String username;

        // 1. 检查 Authorization Header
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // 2. 提取 JWT Token
        jwt = authHeader.substring(7);

        try {
            // 3. 提取用户名
            username = jwtUtils.extractUsername(jwt);

            // 4. 设置认证
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

                if (jwtUtils.validateToken(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                    );
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (Exception e) {
            // Token 无效，静默处理
            logger.debug("JWT token validation failed: " + e.getMessage());
        }

        filterChain.doFilter(request, response);
    }
}
```

### 过滤器位置

在 `SecurityConfig` 中配置过滤器顺序：

```java
.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
```

确保 JWT 过滤器在 Spring Security 的用户名密码认证过滤器之前执行。

## JWT 配置

**文件位置**: `application.yml`

```yaml
jwt:
  secret: freetrader-secret-key-for-jwt-token-generation-must-be-at-least-256-bits-long
  expiration: 86400000  # 24 小时，单位：毫秒
```

### 配置说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `jwt.secret` | JWT 签名密钥，至少 256 位 | - |
| `jwt.expiration` | Token 有效期 | 86400000 (24小时) |

## 客户端使用

### 请求时携带 Token

```http
GET /api/favorites HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Token 存储

前端将 Token 存储在 localStorage 中：

```javascript
// 登录成功后
localStorage.setItem('token', response.data.token);

// 请求时携带
const token = localStorage.getItem('token');
fetch('/api/favorites', {
    headers: {
        'Authorization': `Bearer ${token}`
    }
});
```

## 相关笔记

- [[08-安全认证]]
- [[11-配置类]]
