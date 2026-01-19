# Controller 层实现

## 概述

Controller 层负责处理 HTTP 请求，接收参数，调用 Service 层处理业务逻辑，并返回统一格式的响应。

## 统一响应格式

所有接口返回 `Result<T>` 格式：

```json
{
  "code": 200,
  "message": "success",
  "data": { }
}
```

## AuthController - 认证控制器

**文件位置**: `controller/AuthController.java`
**基础路径**: `/api/auth`

### 登录接口

```java
@PostMapping("/login")
public Result<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
    try {
        AuthResponse response = userService.login(request);
        return Result.success(response);
    } catch (Exception e) {
        return Result.error(401, e.getMessage());
    }
}
```

### 注册接口

```java
@PostMapping("/register")
public Result<AuthResponse> register(@Valid @RequestBody RegisterRequest request) {
    try {
        AuthResponse response = userService.register(request);
        return Result.success(response);
    } catch (Exception e) {
        return Result.error(400, e.getMessage());
    }
}
```

## SectorController - 板块控制器

**文件位置**: `controller/SectorController.java`
**基础路径**: `/api/sectors`

### 获取当前用户 ID

```java
private Integer getCurrentUserId() {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth != null && auth.isAuthenticated()
            && !"anonymousUser".equals(auth.getPrincipal())) {
        String username = auth.getName();
        User user = userService.findByUsername(username);
        return user != null ? user.getId() : null;
    }
    return null;
}
```

### 获取所有板块列表

```java
@GetMapping
public Result<List<SectorDTO>> getAllSectors() {
    try {
        Integer userId = getCurrentUserId();
        List<SectorDTO> sectors = sectorService.getAllSectors(userId);
        return Result.success(sectors);
    } catch (Exception e) {
        return Result.error(e.getMessage());
    }
}
```

### 获取板块详情

```java
@GetMapping("/{id}")
public Result<Map<String, Object>> getSectorDetail(@PathVariable Integer id) {
    try {
        Integer userId = getCurrentUserId();
        Map<String, Object> detail = sectorService.getSectorDetail(id, userId);
        return Result.success(detail);
    } catch (Exception e) {
        return Result.error(e.getMessage());
    }
}
```

## FavoriteController - 收藏控制器

**文件位置**: `controller/FavoriteController.java`
**基础路径**: `/api/favorites`

### 获取用户收藏列表

```java
@GetMapping
public Result<List<SectorDTO>> getFavorites() {
    try {
        Integer userId = getCurrentUserId();
        List<SectorDTO> favorites = favoriteService.getFavoriteSectors(userId);
        return Result.success(favorites);
    } catch (Exception e) {
        return Result.error(e.getMessage());
    }
}
```

### 添加收藏

```java
@PostMapping("/{cid}")
public Result<Void> addFavorite(@PathVariable Integer cid) {
    try {
        Integer userId = getCurrentUserId();
        favoriteService.addFavorite(userId, cid);
        return Result.success(null);
    } catch (Exception e) {
        return Result.error(e.getMessage());
    }
}
```

### 取消收藏

```java
@DeleteMapping("/{cid}")
public Result<Void> removeFavorite(@PathVariable Integer cid) {
    try {
        Integer userId = getCurrentUserId();
        favoriteService.removeFavorite(userId, cid);
        return Result.success(null);
    } catch (Exception e) {
        return Result.error(e.getMessage());
    }
}
```

### 切换收藏状态

```java
@PostMapping("/{cid}/toggle")
public Result<Map<String, Boolean>> toggleFavorite(@PathVariable Integer cid) {
    try {
        Integer userId = getCurrentUserId();
        boolean isFavorite = favoriteService.toggleFavorite(userId, cid);
        return Result.success(Map.of("isFavorite", isFavorite));
    } catch (Exception e) {
        return Result.error(e.getMessage());
    }
}
```

## 异常处理

Controller 层采用 try-catch 捕获异常，统一返回错误信息：

```java
try {
    // 业务逻辑
    return Result.success(data);
} catch (Exception e) {
    return Result.error(错误码, e.getMessage());
}
```

## 相关笔记

- [[06-Service层实现]]
- [[12-API接口文档]]
