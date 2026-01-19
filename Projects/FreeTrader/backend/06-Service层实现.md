# Service 层实现

## 概述

Service 层负责业务逻辑处理，封装具体的业务规则和事务。

## UserService - 用户服务

**文件位置**: `service/UserService.java`

### 登录流程

```java
public AuthResponse login(LoginRequest request) {
    // 1. 根据用户名查询用户
    User user = findByUsername(request.getUsername());

    // 2. 验证用户是否存在
    if (user == null) {
        throw new RuntimeException("用户不存在");
    }

    // 3. 验证密码
    if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
        throw new RuntimeException("密码错误");
    }

    // 4. 生成 JWT Token
    String token = jwtUtils.generateToken(user.getUsername());

    // 5. 返回认证响应
    return new AuthResponse(token, user.getId(), user.getUsername(), user.getEmail());
}
```

### 注册流程

```java
public AuthResponse register(RegisterRequest request) {
    // 1. 检查用户名是否已存在
    if (findByUsername(request.getUsername()) != null) {
        throw new RuntimeException("用户名已存在");
    }

    // 2. 检查邮箱是否已存在
    if (userMapper.selectCount(new QueryWrapper<User>()
            .eq("email", request.getEmail())) > 0) {
        throw new RuntimeException("邮箱已被注册");
    }

    // 3. 密码加密
    String encodedPassword = passwordEncoder.encode(request.getPassword());

    // 4. 创建新用户
    User user = new User();
    user.setUsername(request.getUsername());
    user.setEmail(request.getEmail());
    user.setPassword(encodedPassword);
    userMapper.insert(user);

    // 5. 生成 JWT Token
    String token = jwtUtils.generateToken(user.getUsername());

    return new AuthResponse(token, user.getId(), user.getUsername(), user.getEmail());
}
```

## SectorService - 板块服务

**文件位置**: `service/SectorService.java`

### 获取所有板块列表

```java
public List<SectorDTO> getAllSectors(Integer userId) {
    // 1. 获取交易日范围
    List<String> tradingDays = getTradingDayRange(defaultTradingDays);

    // 2. 查询板块及平均涨跌幅
    List<SectorDTO> sectors = categoryMapper.findAllSectorsWithAvgChange(
            latestDay, earliestDay);

    // 3. 获取用户收藏
    Set<Integer> favoriteCids = new HashSet<>();
    if (userId != null) {
        List<Integer> cids = userCollectionMapper.findFavoriteCidsByUserId(userId);
        favoriteCids.addAll(cids);
    }

    // 4. 补充板块数据
    for (SectorDTO sector : sectors) {
        sector.setIsFavorite(favoriteCids.contains(sector.getId()));
        sector.setMarketCap(calculateMarketCap(sector.getFundsCount()));
        sector.setPrice(calculatePrice(sector.getChange()));
        sector.setTrend(generateTrendData(sector.getChange()));
    }

    return sectors;
}
```

### 获取板块详情

```java
public Map<String, Object> getSectorDetail(Integer sectorId, Integer userId) {
    // 1. 查询板块信息
    Category category = categoryMapper.selectById(sectorId);
    if (category == null) {
        throw new RuntimeException("板块不存在");
    }

    // 2. 获取交易日范围
    List<String> tradingDays = getTradingDayRange(defaultTradingDays);

    // 3. 查询板块内收益率最高的 ETF
    List<EtfDTO> topFunds = etfInfoMapper.findTopEtfsBySector(
            category.getName(), latestDay, earliestDay, defaultTopFunds);

    // 4. 补充 ETF 显示属性
    for (int i = 0; i < topFunds.size(); i++) {
        EtfDTO fund = topFunds.get(i);
        fund.setIcon(generateIcon(fund.getFullName()));
        fund.setIconBg(bgColors[i % bgColors.length]);
    }

    // 5. 检查是否已收藏
    boolean isFavorite = false;
    if (userId != null) {
        List<Integer> cids = userCollectionMapper.findFavoriteCidsByUserId(userId);
        isFavorite = cids != null && cids.contains(sectorId);
    }

    // 6. 组装返回结果
    Map<String, Object> result = new HashMap<>();
    result.put("id", category.getCid());
    result.put("name", category.getName());
    result.put("fundsCount", category.getItemCount());
    result.put("isFavorite", isFavorite);
    result.put("funds", topFunds);

    return result;
}
```

### 收益率计算公式

```java
// 收益率 = (最新复权净值 - 期初复权净值) / 期初复权净值 * 100
returns = (latest.adjusted_nav - earliest.adjusted_nav) / earliest.adjusted_nav * 100
```

## FavoriteService - 收藏服务

**文件位置**: `service/FavoriteService.java`

### 获取用户收藏

```java
public List<SectorDTO> getFavoriteSectors(Integer userId) {
    List<Integer> favoriteCids = userCollectionMapper.findFavoriteCidsByUserId(userId);
    Set<Integer> favoriteSet = Set.copyOf(favoriteCids);

    return sectorService.getAllSectors(userId).stream()
            .filter(s -> favoriteSet.contains(s.getId()))
            .collect(Collectors.toList());
}
```

### 添加收藏

```java
@Transactional
public void addFavorite(Integer userId, Integer cid) {
    // 检查是否已收藏
    Long count = userCollectionMapper.selectCount(
            new QueryWrapper<UserCollection>()
                    .eq("user_id", userId)
                    .eq("cid", cid));

    if (count > 0) {
        throw new RuntimeException("已收藏该板块");
    }

    // 插入收藏记录
    UserCollection collection = new UserCollection();
    collection.setUserId(userId);
    collection.setCid(cid);
    userCollectionMapper.insert(collection);
}
```

### 切换收藏状态

```java
@Transactional
public boolean toggleFavorite(Integer userId, Integer cid) {
    Long count = userCollectionMapper.selectCount(
            new QueryWrapper<UserCollection>()
                    .eq("user_id", userId)
                    .eq("cid", cid));

    if (count > 0) {
        removeFavorite(userId, cid);
        return false;
    } else {
        addFavorite(userId, cid);
        return true;
    }
}
```

## 相关笔记

- [[05-Mapper层实现]]
- [[07-Controller层实现]]
