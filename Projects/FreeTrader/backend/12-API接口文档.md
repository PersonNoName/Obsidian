# API 接口文档

## 概述

项目提供 RESTful API 接口，所有接口返回统一格式的 `Result<T>` 响应。

## 基础信息

| 属性 | 值 |
|------|-----|
| 基础路径 | `/api` |
| 认证方式 | Bearer Token |
| 跨域来源 | `http://localhost:3000` |

---

## 认证模块 - /api/auth

### POST /api/auth/login - 用户登录

**权限**: 公开

**请求参数**:

```json
{
    "username": "string",
    "password": "string"
}
```

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
        "userId": 1,
        "username": "testuser",
        "email": "test@example.com"
    }
}
```

**响应错误**:

```json
{
    "code": 401,
    "message": "用户名或密码错误",
    "data": null
}
```

### POST /api/auth/register - 用户注册

**权限**: 公开

**请求参数**:

```json
{
    "username": "string (3-50字符)",
    "email": "string (邮箱格式)",
    "password": "string (6-100字符)"
}
```

**响应成功**: 同登录接口

**响应错误**:

```json
{
    "code": 400,
    "message": "用户名已存在",
    "data": null
}
```

---

## 板块模块 - /api/sectors

### GET /api/sectors - 获取所有板块列表

**权限**: 公开

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": [
        {
            "id": 1,
            "name": "科技",
            "description": "科技板块",
            "fundsCount": 50,
            "change": 1.25,
            "price": 1012.5,
            "marketCap": "2.5T",
            "isFavorite": true,
            "trend": [9.8, 10.1, 9.9, 10.3, 10.0, 10.2, 10.1]
        }
    ]
}
```

**SectorDTO 字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 板块 ID |
| name | String | 板块名称 |
| description | String | 板块描述 |
| fundsCount | Integer | ETF 数量 |
| change | Double | 平均涨跌幅 (%) |
| price | Double | 模拟价格 |
| marketCap | String | 市值模拟值 |
| isFavorite | Boolean | 是否已收藏 |
| trend | List\<Double\> | 趋势数据 |

### GET /api/sectors/{id} - 获取板块详情

**权限**: 公开

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| id | Integer | 板块 ID |

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "id": 1,
        "name": "科技",
        "description": "科技板块",
        "fundsCount": 50,
        "isFavorite": true,
        "funds": [
            {
                "name": "159981",
                "fullName": "创业板ETF",
                "price": 2.123,
                "returns": 1.25,
                "returnsPercent": 0.0125,
                "icon": "创",
                "iconBg": "bg-blue-600"
            }
        ]
    }
}
```

**EtfDTO 字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| name | String | ETF 代码 |
| fullName | String | ETF 全称 |
| price | Double | 最新净值 |
| returns | Double | 收益率 (%) |
| returnsPercent | Double | 收益率 (小数) |
| icon | String | 显示图标 |
| iconBg | String | 图标背景色 |

---

## 收藏模块 - /api/favorites

**认证**: 需要 Token

### GET /api/favorites - 获取收藏列表

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": [
        {
            "id": 1,
            "name": "科技",
            "isFavorite": true
        }
    ]
}
```

### POST /api/favorites/{cid} - 添加收藏

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| cid | Integer | 板块 ID |

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": null
}
```

**响应错误**:

```json
{
    "code": 500,
    "message": "已收藏该板块",
    "data": null
}
```

### DELETE /api/favorites/{cid} - 取消收藏

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| cid | Integer | 板块 ID |

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": null
}
```

### POST /api/favorites/{cid}/toggle - 切换收藏状态

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| cid | Integer | 板块 ID |

**响应成功**:

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "isFavorite": false
    }
}
```

---

## 错误码说明

| 错误码 | 说明 |
|--------|------|
| 200 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未授权 / Token 无效 |
| 500 | 服务器内部错误 |

## 请求示例

### 登录请求

```http
POST /api/auth/login HTTP/1.1
Content-Type: application/json

{
    "username": "testuser",
    "password": "password123"
}
```

### 带 Token 的请求

```http
GET /api/favorites HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## 相关笔记

- [[07-Controller层实现]]
- [[10-统一响应封装]]
