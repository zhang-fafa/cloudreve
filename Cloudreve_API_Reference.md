# Cloudreve v4 API 接口文档

> 基于 Cloudreve v4 源码自动整理，覆盖所有 Master 模式下的 REST API。

---

## 目录

- [基础信息](#基础信息)
- [鉴权方式](#鉴权方式)
- [通用响应格式](#通用响应格式)
- [错误码](#错误码)
- [接口列表](#接口列表)
  - [站点与健康检查](#1-站点与健康检查)
  - [用户会话与鉴权](#2-用户会话与鉴权)
  - [用户管理](#3-用户管理)
  - [文件操作](#4-文件操作)
  - [文件上传（分片）](#5-文件上传分片)
  - [分享](#6-分享)
  - [工作流任务](#7-工作流任务)
  - [WebDAV](#8-webdav)
  - [管理后台接口](#9-管理后台接口)
- [数据结构](#数据结构)

---

## 基础信息

| 项目 | 值 |
|------|-----|
| API 前缀 | `/api/v4` |
| 协议 | HTTP/HTTPS |
| 请求体格式 | `application/json`（除特殊标注） |
| 响应格式 | `application/json` |
| 所有 ID 编码 | HashID（字符串，如 `aB3xYk`），不是数据库原始 int |
| 源文件 | [routers/router.go](../routers/router.go) |

## 鉴权方式

### 1. JWT Bearer Token（推荐）

```
Authorization: Bearer <access_token>
```

- 通过 `/api/v4/session/token` 登录获取
- Access Token 有效期：默认 1 小时
- Refresh Token 有效期：默认 7 天
- Token 类型：HS256 签名 JWT

### 2. Session Cookie（Web 前端）

- 通过服务端 Set-Cookie 自动维护
- 适合浏览器环境

### 3. HMAC 签名

- 用于文件下载等 URL 鉴权接口
- Header: `Authorization: Cr <signature>`
- 参数由服务端自动签发，客户端直接使用返回的 URL 即可

### 4. WebDAV Basic Auth

```
Authorization: Basic base64(用户名:WebDAV密码)
```

- 适用于 `/dav/*` 路径
- WebDAV 密码在用户设备管理中创建

### 权限 Scope

部分 OAuth 保护接口检查 Token 中的 Scope：
- `Files.Read` / `Files.Write`
- `Workflow.Read` / `Workflow.Write`
- `Shares.Read` / `Shares.Write`
- `DavAccount.Read` / `DavAccount.Write`
- `UserInfo.Read` / `UserInfo.Write`
- `UserSecurityInfo.Write`
- `AdminRead` / `AdminWrite`

> Write Scope 自动隐含对应的 Read 权限。

## 通用响应格式

### 成功响应

```json
{
  "code": 0,
  "data": { ... },
  "msg": ""
}
```

### 错误响应

```json
{
  "code": 40021,
  "msg": "User not found",
  "error": "底层错误详情（仅非 Release 模式）",
  "correlation_id": "trace-uuid"
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 0=成功，非 0=失败；三位数复用 HTTP 语义，五位数为业务码 |
| `data` | any | 业务数据，成功时返回 |
| `msg` | string | 成功/错误信息 |
| `error` | string | 底层原始错误（仅开发模式） |
| `aggregated_error` | object | 批量操作部分失败时返回 |
| `correlation_id` | string | 请求追踪 ID |

## 错误码

### HTTP 语义错误码

| Code | 含义 |
|------|------|
| 203 | 未完全成功（如登录需要 2FA） |
| 401 | 未登录 / Token 无效 |
| 403 | 无权限 |
| 404 | 资源未找到 |
| 409 | 资源冲突 |

### 客户端错误（4xxxx）

| Code | 常量名 | 含义 |
|------|--------|------|
| 40001 | `CodeParamErr` | 参数错误 |
| 40002 | `CodeUploadFailed` | 上传失败 |
| 40004 | `CodeObjectExist` | 对象已存在 |
| 40005 | `CodeSignExpired` | 签名过期 |
| 40007 | `CodeGroupNotAllowed` | 用户组无此权限 |
| 40008 | `CodeAdminRequired` | 需管理员权限 |
| 40011 | `CodeUploadSessionExpired` | 上传会话过期 |
| 40012 | `CodeInvalidChunkIndex` | 无效分片序号 |
| 40013 | `CodeInvalidContentLength` | 正文长度无效 |
| 40014 | `CodeBatchSourceSize` | 批量外链超出限制 |
| 40016 | `CodeParentNotExist` | 父目录不存在 |
| 40017 | `CodeUserBaned` | 用户被封禁 |
| 40018 | `CodeUserNotActivated` | 用户未激活 |
| 40019 | `CodeFeatureNotEnabled` | 功能未开启 |
| 40020 | `CodeCredentialInvalid` | 凭证无效 |
| 40021 | `CodeUserNotFound` | 用户不存在 |
| 40022 | `Code2FACodeErr` | 2FA 代码错误 |
| 40026 | `CodeCaptchaError` | 验证码错误 |
| 40027 | `CodeCaptchaRefreshNeeded` | 验证码需刷新 |
| 40028 | `CodeFailedSendEmail` | 邮件发送失败 |
| 40029 | `CodeInvalidTempLink` | 临时链接无效 |
| 40030 | `CodeTempLinkExpired` | 临时链接过期 |
| 40032 | `CodeEmailExisted` | 邮箱已被使用 |
| 40044 | `CodeFileNotFound` | 文件不存在 |
| 40049 | `CodeFileTooLarge` | 文件太大 |
| 40050 | `CodeFileTypeNotAllowed` | 文件类型不允许 |
| 40051 | `CodeInsufficientCapacity` | 容量不足 |
| 40052 | `CodeIllegalObjectName` | 文件名非法 |
| 40053 | `CodeRootProtected` | 根目录受保护 |
| 40058 | `CodeShareLinkNotFound` | 分享链接无效 |
| 40073 | `CodeLockConflict` | 文件锁冲突 |
| 40075 | `CodeLockExpired` | 锁已过期 |
| 40077 | `CodeEntityNotExist` | Entity 不存在 |
| 40078 | `CodeFileDeleted` | 文件已删除 |
| 40089 | `CodeInsufficientScope` | OAuth Scope 不足 |

### 服务端错误（5xxxx）

| Code | 常量名 | 含义 |
|------|--------|------|
| 50001 | `CodeDBError` | 数据库操作失败 |
| 50002 | `CodeEncryptError` | 加密失败 |
| 50004 | `CodeIOFailed` | IO 操作失败 |
| 50005 | `CodeInternalSetting` | 内部配置错误 |
| 50009 | `CodeAddCORS` | 跨域策略添加失败 |
| 50010 | `CodeNodeOffline` | 节点离线 |

---

## 接口列表

### 0. OAuth2 开放授权

Cloudreve 内置完整的 OAuth2 服务端，支持第三方应用接入。

#### 0.1 获取应用注册信息

```
GET /api/v4/session/oauth/app/:app_id
```

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `app_id` | string | OAuth Client GUID |

**鉴权**：无

#### 0.2 授权同意

```
POST /api/v4/session/oauth/consent
```

**鉴权**：Token

**请求体**：
```json
{
  "client_id": "uuid-of-oauth-app",
  "scope": ["Files.Read", "UserInfo.Read"],
  "response_type": "code",
  "redirect_uri": "https://app.example.com/callback",
  "state": "csrf-token",
  "code_challenge": "...",
  "code_challenge_method": "S256"
}
```

**响应**：返回授权码或直接返回 Token（implicit flow）

#### 0.3 拒绝授权

```
POST /api/v4/session/oauth/consent/deny
```

**鉴权**：Token

**请求体**：同 consent 的 client_id 等字段

#### 0.4 用授权码换取 Token

```
POST /api/v4/session/oauth/token
Content-Type: application/x-www-form-urlencoded
```

**请求体**：
```
grant_type=authorization_code&
code=<authorization_code>&
redirect_uri=https://app.example.com/callback&
client_id=<client_id>&
client_secret=<client_secret>
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `grant_type` | string | ✅ | `authorization_code` 或 `refresh_token` |
| `code` | string | ✅ | 授权码 |
| `redirect_uri` | string | ✅ | 回调地址 |
| `client_id` | string | ✅ | 应用 ID |
| `client_secret` | string | ✅ | 应用密钥（confidential 客户端） |

**响应**：
```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

#### 0.5 获取当前用户信息（OAuth）

```
GET /api/v4/session/oauth/userinfo
```

**鉴权**：OAuth Bearer Token

#### 0.6 撤销授权

```
DELETE /api/v4/session/oauth/grant/:app_id
```

**鉴权**：Token + `UserSecurityInfo.Write`

---

### 1. 站点与健康检查

#### 1.1 Ping / 健康检查

```
GET /api/v4/site/ping
```

**鉴权**：无

**响应**：
```json
{ "code": 0, "msg": "pong" }
```

#### 1.2 获取验证码

```
GET /api/v4/site/captcha
```

**鉴权**：无

**响应**：图片二进制 + Cookie（存储 captcha session）

#### 1.3 获取站点配置

```
GET /api/v4/site/config/:section
```

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `section` | string | 配置段名（如 `basic`, `branding` 等） |

**鉴权**：无

---

### 2. 用户会话与鉴权

#### 2.1 用户登录（获取 Token）

```
POST /api/v4/session/token
```

**鉴权**：无

**请求体**：
```json
{
  "email": "user@example.com",
  "password": "your_password"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | ✅ | 用户邮箱 |
| `password` | string | ✅ | 密码（4-128 字符） |

**响应（无 2FA）**：
```json
{
  "code": 0,
  "data": {
    "user": { ...User },
    "token": {
      "access_token": "eyJhbGciOiJIUzI1NiIs...",
      "refresh_token": "dGhpcyBpcyBhIHJlZnJl...",
      "access_expires": "2026-10-27T11:00:00Z",
      "refresh_expires": "2026-11-03T11:00:00Z"
    }
  }
}
```

**响应（需要 2FA，code=203）**：
```json
{
  "code": 203,
  "data": "session-uuid-string",
  "msg": ""
}
```

#### 2.2 两步验证

```
POST /api/v4/session/token/2fa
```

**请求体**：
```json
{
  "otp": "123456",
  "session_id": "session-uuid-string"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `otp` | string | ✅ | TOTP 验证码 |
| `session_id` | string | ✅ | 上一步返回的 2FA 会话 ID |

**响应**：同登录接口（成功返回 Token 对）

#### 2.3 刷新 Token

```
POST /api/v4/session/token/refresh
```

**需要 Scope**：`offline_access`

**请求体**：
```json
{
  "refresh_token": "<refresh_token>"
}
```

**响应**：
```json
{
  "code": 0,
  "data": {
    "access_token": "...",
    "refresh_token": "...",
    "access_expires": "...",
    "refresh_expires": "..."
  }
}
```

#### 2.4 退出登录

```
DELETE /api/v4/session/token
```

**请求体**：
```json
{
  "refresh_token": "<refresh_token>"
}
```

#### 2.5 登录预检查

```
GET /api/v4/session/prepare?email=user@example.com
```

**鉴权**：无

**Query 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `email` | string | 用户邮箱 |

**响应**：
```json
{
  "code": 0,
  "data": {
    "webauthn_enabled": true,
    "password_enabled": true
  }
}
```

#### 2.6 WebAuthn 登录准备

```
PUT /api/v4/session/authn
```

**鉴权**：无

**响应**：WebAuthn PublicKeyCredentialRequestOptions

#### 2.7 WebAuthn 登录完成

```
POST /api/v4/session/authn
```

**请求体**：WebAuthn AuthenticatorAssertionResponse（JSON 字符串化）

**响应**：同登录接口（成功返回 Token 对）

---

### 3. 用户管理

#### 3.1 用户注册

```
POST /api/v4/user
```

**鉴权**：无（需管理员开启注册）

**请求体**：
```json
{
  "email": "new@example.com",
  "password": "secure_password",
  "language": "zh"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | ✅ | 邮箱 |
| `password` | string | ✅ | 密码 |
| `language` | string | ❌ | 语言代码 |

#### 3.2 发送密码重置邮件

```
POST /api/v4/user/reset
```

**请求体**：
```json
{
  "email": "user@example.com",
  "language": "zh"
}
```

#### 3.3 重设密码

```
PATCH /api/v4/user/reset/:id
```

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | HashID 编码的用户 ID |

**请求体**：
```json
{
  "password": "new_password_min6",
  "secret": "邮件中携带的 secret"
}
```

#### 3.4 邮件激活

```
GET /api/v4/user/activate/:id
```

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | HashID 编码的用户 ID |

#### 3.5 获取用户公开信息

```
GET /api/v4/user/info/:id
```

**Path 参数**：HashID 编码的用户 ID

#### 3.6 获取用户头像

```
GET /api/v4/user/avatar/:id
```

#### 3.7 当前登录用户信息

```
GET /api/v4/auth/user/me
```

**鉴权**：Token

**响应**：
```json
{
  "code": 0,
  "data": {
    "id": "abc123",
    "email": "user@example.com",
    "nick": "昵称",
    "group": { ... },
    "status": 1,
    ...
  }
}
```

#### 3.8 存储容量使用情况

```
GET /api/v4/auth/user/capacity
```

**鉴权**：Token

#### 3.9 用户个人设置

```
GET /api/v4/auth/user/setting
```

**鉴权**：Token

#### 3.10 修改用户设置

```
PATCH /api/v4/auth/user/setting
```

**鉴权**：Token + `UserInfo.Write`

**请求体**：
```json
{
  "nick": "新昵称",
  "language": "zh",
  "theme": "auto",
  "password": {
    "current": "旧密码",
    "new": "新密码"
  }
}
```

#### 3.11 初始化二步验证

```
GET /api/v4/auth/user/setting/2fa
```

**鉴权**：Token

**响应**：包含 TOTP secret 和 QR 码数据

#### 3.12 搜索用户

```
GET /api/v4/auth/user/search?keywords=xxx
```

**鉴权**：Token

---

### 4. 文件操作

> **关键概念**：Cloudreve v4 使用 URI 定位文件，格式如 `p/1/文件名` 或 `/根目录/子目录/文件名`。部分接口可通过 `id`（HashID）替代 URI。

#### 4.1 列出目录文件

```
GET /api/v4/file?uri=%2F&page=1&page_size=50&order_by=name&order_direction=asc
```

**鉴权**：Token + `Files.Read`

**Query 参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 目录 URI，根目录为 `/` |
| `page` | int | ❌ | 页码，默认 0 |
| `page_size` | int | ❌ | 每页数量 |
| `order_by` | string | ❌ | 排序字段 |
| `order_direction` | string | ❌ | `asc` / `desc` |
| `next_page_token` | string | ❌ | 分页 token |

**响应**：
```json
{
  "code": 0,
  "data": {
    "files": [ { ...FileResponse } ],
    "parent": { ...FileResponse },
    "pagination": { "total": 100, "page": 1, "page_size": 50 },
    "props": { ... },
    "context_hint": "uuid-for-next-request",
    "mixed_type": false,
    "storage_policy": { ...StoragePolicy }
  }
}
```

#### 4.2 获取文件/目录信息

```
GET /api/v4/file/info?uri=%2Fdocument.pdf
GET /api/v4/file/info?id=abc123&extended=true&folder_summary=true
```

**Query 参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ❌ | 文件 URI（与 id 二选一） |
| `id` | string | ❌ | HashID 编码的文件 ID |
| `extended` | bool | ❌ | 返回扩展信息（Entity、StoragePolicy 等） |
| `folder_summary` | bool | ❌ | 目录汇总信息 |

#### 4.3 创建空白文件/目录

```
POST /api/v4/file/create
```

**鉴权**：Token + `Files.Write`

**请求体**：
```json
{
  "uri": "/new_folder",
  "type": "folder",
  "metadata": {},
  "err_on_conflict": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 完整 URI |
| `type` | string | ✅ | `file` 或 `folder` |
| `metadata` | object | ❌ | 元数据键值对 |
| `err_on_conflict` | bool | ❌ | 同名存在时是否报错 |

#### 4.4 重命名

```
POST /api/v4/file/rename
```

**请求体**：
```json
{
  "uri": "/old_name.txt",
  "new_name": "new_name.txt"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 文件 URI |
| `new_name` | string | ✅ | 新名称（1-255 字符） |

#### 4.5 移动/复制

```
POST /api/v4/file/move
```

**请求体**：
```json
{
  "uris": ["/file1.txt", "/folder1"],
  "dst": "/target_folder",
  "copy": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uris` | string[] | ✅ | 源文件 URI 数组（至少 1 个） |
| `dst` | string | ✅ | 目标目录 URI |
| `copy` | bool | ❌ | `true`=复制，`false`=移动 |

#### 4.6 删除文件（到回收站）

```
DELETE /api/v4/file
```

**请求体**：
```json
{
  "uris": ["/file_to_delete.txt"],
  "unlink": false,
  "skip_soft_delete": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uris` | string[] | ✅ | 要删除的 URI |
| `unlink` | bool | ❌ | 物理删除（需高级权限） |
| `skip_soft_delete` | bool | ❌ | 跳过回收站直接物理删除 |

#### 4.7 清空回收站

```
DELETE /api/v4/file/trash
```

#### 4.8 从回收站恢复

```
POST /api/v4/file/restore
```

**请求体**：同删除接口的 `uris` 格式

#### 4.9 获取下载/预览 URL

```
POST /api/v4/file/url
```

**请求体**：
```json
{
  "uris": ["/document.pdf"],
  "download": true,
  "redirect": false,
  "skip_error": false,
  "archive": false,
  "no_cache": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uris` | string[] | ✅ | 文件 URI 数组 |
| `download` | bool | ❌ | 下载模式（影响 Content-Disposition） |
| `redirect` | bool | ❌ | 302 重定向到文件（仅 URI 数量=1 有效） |
| `entity` | string | ❌ | 指定 Entity ID（仅 URI 数量=1） |
| `use_primary_site_url` | bool | ❌ | 使用主站域名 |
| `skip_error` | bool | ❌ | 跳过错误继续返回部分结果 |
| `archive` | bool | ❌ | **特殊：打包下载，返回签名的归档会话 URL** |
| `no_cache` | bool | ❌ | 不使用缓存 |

**响应**：
```json
{
  "code": 0,
  "data": {
    "urls": [
      {
        "url": "https://cdn.example.com/path/to/file?sign=xxx",
        "entity_id": "abc123"
      }
    ],
    "expires": "2026-10-27T12:00:00Z"
  }
}
```

#### 4.9.1 分页文件列表（每项附带下载 URL）

```
GET /api/v4/file/list?uri=cloudreve://my/&page=1&page_size=50&order_by=name&order_direction=asc
```

**鉴权**：Token + `Files.Read`

分页列出目录内容，行为与 4.1 一致，但每个文件类型的 item 额外附带与 4.9 相同格式的临时下载/预览 URL，适合客户端一次请求即可展示列表并播放/下载，无需再逐个调用 `/file/url`。

**Query 参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 目录 URI，根目录为 `cloudreve://my/` |
| `page` | int | ❌ | 页码，默认 0 |
| `page_size` | int | ❌ | 每页数量 |
| `order_by` | string | ❌ | 排序字段 |
| `order_direction` | string | ❌ | `asc` / `desc` |
| `next_page_token` | string | ❌ | 分页 token |
| `download` | bool | ❌ | 下载模式（影响 Content-Disposition），默认 false 为预览模式 |
| `no_cache` | bool | ❌ | 不使用 URL 缓存 |

**响应**：
```json
{
  "code": 0,
  "data": {
    "files": [
      {
        "type": 0,
        "id": "abc123",
        "name": "document.pdf",
        "created_at": "2026-10-27T10:00:00Z",
        "updated_at": "2026-10-27T10:00:00Z",
        "size": 10240,
        "metadata": {},
        "path": "cloudreve://my/document.pdf",
        "owned": true,
        "urls": [
          {
            "url": "https://cdn.example.com/path/to/file?sign=xxx",
            "stream_saver_display_name": ""
          }
        ]
      },
      {
        "type": 1,
        "id": "def456",
        "name": "docs",
        "size": 0
      }
    ],
    "pagination": { "total": 100, "page": 1, "page_size": 50 },
    "expires": "2026-10-27T12:00:00Z"
  }
}
```

**响应字段说明**：

| 字段 | 说明 |
|------|------|
| `files[].urls` | 仅文件类型（`type=0`）附带；目录（`type=1`）无此字段。单个文件取 URL 失败时该项不带 `urls`，不影响整页 |
| `expires` | 本页所有 URL 中最早的过期时间 |

#### 4.9.2 随机推荐文件

```
GET /api/v4/file/recommend?uri=cloudreve://my/&count=10&download=false
```

**鉴权**：Token + `Files.Read`

从指定目录中随机抽取文件返回，item 结构与 4.9.1 完全一致（文件信息 + `urls`）。仅返回文件类型，目录不会被推荐。每次请求结果随机，适合作"猜你喜欢 / 随机播放"等场景。

**Query 参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 推荐来源目录 URI，根目录为 `cloudreve://my/` |
| `count` | int | ❌ | 推荐数量，默认 10 |
| `download` | bool | ❌ | 下载模式（影响 Content-Disposition），默认 false 为预览模式 |
| `no_cache` | bool | ❌ | 不使用 URL 缓存 |

**响应**：
```json
{
  "code": 0,
  "data": {
    "files": [
      {
        "type": 0,
        "id": "abc123",
        "name": "video.mp4",
        "created_at": "2026-10-27T10:00:00Z",
        "size": 10240,
        "path": "cloudreve://my/video.mp4",
        "urls": [
          {
            "url": "https://cdn.example.com/path/to/file?sign=xxx"
          }
        ]
      }
    ],
    "expires": "2026-10-27T12:00:00Z"
  }
}
```

**说明**：实现上先按默认排序取一个较大的候选集（`count × 5`，上限为导航器最大页大小），再打乱并截取 `count` 个。因此随机范围是该目录排序靠前的候选集；目录内文件不足时返回全部文件。

#### 4.10 下载/预览文件内容

```
GET /api/v4/file/content/:id/:speed/:name
```

**鉴权**：需要 URL 签名（通过上一步获取的 URL 自动携带）

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | HashID 编码的 Entity ID |
| `speed` | string | 下载速度限制（如 `0` 表示不限） |
| `name` | string | 文件名（用于 Content-Disposition） |

**响应**：文件二进制流

#### 4.11 获取缩略图 URL

```
GET /api/v4/file/thumb?uri=%2Fimage.png
```

**Query 参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 文件 URI |

**响应**：
```json
{
  "code": 0,
  "data": {
    "url": "https://cdn.example.com/thumb.png?sign=xxx",
    "expires": "2026-10-27T12:00:00Z"
  }
}
```

#### 4.12 获取永久直链

```
PUT /api/v4/file/source
```

**请求体**：
```json
{
  "uris": ["/document.pdf"]
}
```

**响应**：
```json
{
  "code": 0,
  "data": [
    {
      "link": "https://your-site.com/f/abc123/document.pdf",
      "file_url": "/document.pdf"
    }
  ]
}
```

#### 4.13 删除永久直链

```
DELETE /api/v4/file/source/:id
```

**Path 参数**：HashID 编码的 DirectLink ID

#### 4.14 解锁文件

```
DELETE /api/v4/file/lock
```

**请求体**：
```json
{
  "tokens": ["lock-token-1", "lock-token-2"]
}
```

#### 4.15 修改元数据

```
PATCH /api/v4/file/metadata
```

**请求体**：
```json
{
  "uris": ["/file.txt"],
  "metadata": { "key": "value" }
}
```

#### 4.16 在线编辑文件内容

```
PUT /api/v4/file/content?uri=%2Fnote.txt&previous=version_id
```

**请求体**：请求体直接是文件新内容（Content-Type: text/plain 等）

#### 4.17 列出压缩包内文件

```
GET /api/v4/file/archive?uri=%2Farchive.zip&entity=abc123
```

#### 4.18 全文搜索

```
GET /api/v4/file/search?query=关键词&offset=0
```

**需要开启**：管理后台开启全文搜索功能

#### 4.20 钉文件

```
PUT /api/v4/file/pin
```

**请求体**：
```json
{ "uri": "/important.txt" }
```

#### 4.21 取消钉文件

```
DELETE /api/v4/file/pin
```

**请求体**：
```json
{ "uri": "/important.txt" }
```

#### 4.22 版本管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/file/version/current` | 设置当前版本 |
| DELETE | `/api/v4/file/version` | 删除某个历史版本 |

**设置当前版本请求体**：
```json
{
  "uri": "/document.pdf",
  "entity_id": "abc123"
}
```

#### 4.23 创建 Viewer Session

```
PUT /api/v4/file/viewerSession
```

**鉴权**：Token + `Files.Write`

**请求体**：
```json
{
  "uri": "/document.docx"
}
```

**响应**：返回 WOPI 会话信息（如果配置了 WOPI）

#### 4.24 Server-Sent Events 事件推送

```
GET /api/v4/file/events
Accept: text/event-stream
```

**鉴权**：Token

**说明**：实时推送文件变更事件（需管理员开启功能）

---

### 5. 文件上传（分片）

上传流程：创建会话 → 分片上传 → 服务端自动完成

#### 5.1 创建上传会话

```
PUT /api/v4/file/upload
```

**鉴权**：Token + `Files.Write`

**请求体**：
```json
{
  "uri": "/target_path/large_file.zip",
  "size": 1073741824,
  "last_modified": 1730000000000,
  "mime_type": "application/zip",
  "policy_id": "abc123",
  "metadata": {},
  "entity_type": "",
  "encryption_supported": [],
  "previous": ""
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 目标文件 URI |
| `size` | int64 | ✅ | 文件大小（字节） |
| `last_modified` | int64 | ❌ | 最后修改时间（毫秒时间戳） |
| `mime_type` | string | ❌ | MIME 类型 |
| `policy_id` | string | ❌ | 指定存储策略 HashID |
| `metadata` | object | ❌ | 元数据（最多 256 键） |
| `entity_type` | string | ❌ | `live_photo` / `version` |
| `encryption_supported` | string[] | ❌ | 支持的加密算法 |
| `previous` | string | ❌ | 上一版本 ID（版本更新时用） |

**响应**：
```json
{
  "code": 0,
  "data": {
    "session_id": "upload-session-uuid",
    "upload_id": "cloudreve-upload-id",
    "chunk_size": 5242880,
    "expires": 1730003600,
    "upload_urls": ["https://master/upload/..."],
    "credential": "...",
    "ak": "...",
    "uri": "/target_path/large_file.zip",
    "callback_secret": "...",
    "storage_policy": { ...StoragePolicy }
  }
}
```

#### 5.2 上传分片

```
POST /api/v4/file/upload/:sessionId/:index
Content-Type: application/octet-stream
```

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionId` | string | 上传会话 ID |
| `index` | int | 分片序号（从 0 开始） |

**请求体**：分片的二进制数据

**⚠️ 注意事项**：
- `Content-Length` 必须精确等于该分片的字节数
- 最后一个分片大小 = 文件总大小 - index × chunk_size
- 并发上传时，上传会在所有分片都接收后自动完成

**成功响应**：
```json
{ "code": 0, "msg": "" }
```

#### 5.3 取消上传会话

```
DELETE /api/v4/file/upload
```

**请求体**：
```json
{
  "id": "session-id",
  "uri": "/target_path/large_file.zip"
}
```

---

### 6. 分享

#### 6.1 创建分享链接

```
PUT /api/v4/share
```

**鉴权**：Token + `Shares.Write`

**请求体**：
```json
{
  "uri": "/document.pdf",
  "is_private": false,
  "password": "abc123",
  "downloads": 0,
  "expire": 86400,
  "share_view": false,
  "show_readme": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uri` | string | ✅ | 文件/目录 URI |
| `is_private` | bool | ❌ | 是否私有分享 |
| `password` | string | ❌ | 访问密码（字母数字，最多 32） |
| `downloads` | int | ❌ | 剩余下载次数（0=不限） |
| `expire` | int | ❌ | 有效期（秒，0=永不过期） |
| `share_view` | bool | ❌ | 启用分享视图模式 |
| `show_readme` | bool | ❌ | 显示 README |

**响应**：
```json
{
  "code": 0,
  "data": "https://your-site.com/s/abc123"
}
```

#### 6.2 编辑分享链接

```
POST /api/v4/share/:id
```

**Path 参数**：HashID 编码的 Share ID

**请求体**：同创建接口

#### 6.3 列出我的分享

```
GET /api/v4/share
```

**鉴权**：Token

#### 6.4 获取分享信息

```
GET /api/v4/share/info/:id
```

**鉴权**：无（公开）

**响应**：
```json
{
  "code": 0,
  "data": {
    "id": "abc123",
    "name": "document.pdf",
    "url": "https://your-site.com/s/abc123",
    "visited": 100,
    "downloaded": 50,
    "expired": false,
    "unlocked": true,
    "password_protected": false,
    "source_type": 1,
    "owner": { ...User },
    "source_uri": "/document.pdf",
    "expires": "2026-11-27T00:00:00Z",
    "size": 1048576
  }
}
```

#### 6.5 删除单个分享

```
DELETE /api/v4/share/:id
```

#### 6.6 批量删除分享

```
DELETE /api/v4/share
```

**请求体**：
```json
{
  "ids": ["abc123", "def456"]
}
```

#### 6.7 短链接重定向

```
GET /s/:id
GET /s/:id/:password
```

**鉴权**：无（匿名）

**响应**：302 重定向到分享页面

---

### 7. 工作流任务

#### 7.1 列出任务

```
GET /api/v4/workflow
```

**鉴权**：Token + `Workflow.Read`

#### 7.2 获取任务进度

```
GET /api/v4/workflow/progress/:id
```

**Path 参数**：HashID 编码的 Task ID

#### 7.3 创建压缩任务（打包下载）

```
POST /api/v4/workflow/archive
```

**鉴权**：Token + `Workflow.Write`

**请求体**：
```json
{
  "uris": ["/file1.txt", "/folder1"]
}
```

#### 7.4 创建解压任务

```
POST /api/v4/workflow/extract
```

**请求体**：
```json
{
  "uris": ["/archive.zip"]
}
```

#### 7.5 创建离线下载任务

```
POST /api/v4/workflow/download
```

**鉴权**：Token + `Workflow.Write`

**请求体**：
```json
{
  "urls": ["https://example.com/large-file.zip"],
  "policy_id": "abc123",
  "dst": "/downloads"
}
```

#### 7.6 设置离线下载目标

```
PATCH /api/v4/workflow/download/:id
```

**请求体**：选中的文件列表

#### 7.7 取消离线下载

```
DELETE /api/v4/workflow/download/:id
```

---

### 8. WOPI 在线办公集成

Cloudreve 支持通过 WOPI 协议集成 Office Online / OnlyOffice / Collabora 等在线编辑器。

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v4/file/wopi/:id` | CheckFileInfo — 获取文件信息 |
| GET | `/api/v4/file/wopi/:id/contents` | GetFile — 获取文件内容 |
| POST | `/api/v4/file/wopi/:id/contents` | PutFile — 更新文件内容 |
| POST | `/api/v4/file/wopi/:id` | 通用文件操作（Rename/Lock 等） |

**Path 参数**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | HashID 编码的 Viewer Session ID |

**鉴权**：WOPI 签名验证（由 Viewer Session 创建时获得的 key 自动计算）

### 9. WebDAV

所有 WebDAV 请求使用 Basic Auth（用户名 + WebDAV 密码），路径前缀 `/dav/`。

| 方法 | 路径 | 说明 |
|------|------|------|
| PROPFIND | `/dav/` | 列出目录内容 |
| GET | `/dav/文件` | 下载文件 |
| PUT | `/dav/文件` | 上传文件 |
| MKCOL | `/dav/新目录` | 创建目录 |
| DELETE | `/dav/文件` | 删除 |
| MOVE | `/dav/源` | 移动/重定向（Destination Header 指定目标） |
| COPY | `/dav/源` | 复制 |
| LOCK / UNLOCK | `/dav/文件` | 文件锁 |
| PROPPATCH | `/dav/文件` | 修改属性 |

#### 创建 WebDAV 账号

```
PUT /api/v4/auth/devices/dav
```

**请求体**：
```json
{
  "name": "iOS设备",
  "password": "random_webdav_password",
  "path": "/"
}
```

#### 列出 WebDAV 账号

```
GET /api/v4/auth/devices/dav
```

#### 更新 WebDAV 账号

```
PATCH /api/v4/auth/devices/dav/:id
```

#### 删除 WebDAV 账号

```
DELETE /api/v4/auth/devices/dav/:id
```

---

### 9. 管理后台接口

> 所有管理接口都在 `/api/v4/auth/admin/` 下，需要登录管理员账号。

#### 9.1 站点概览

```
GET /api/v4/auth/admin/summary
```

**Scope**：`AdminRead`

#### 9.2 系统设置

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/settings` | 获取设置 |
| PATCH | `/api/v4/auth/admin/settings` | 修改设置 |

#### 9.3 用户管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/user` | 列出用户 |
| GET | `/api/v4/auth/admin/user/:id` | 获取用户 |
| PUT | `/api/v4/auth/admin/user` | 创建用户 |
| PUT | `/api/v4/auth/admin/user/:id` | 更新用户 |
| POST | `/api/v4/auth/admin/user/batch/delete` | 批量删除 |
| POST | `/api/v4/auth/admin/user/:id/calibrate` | 校准存储用量 |

#### 9.4 用户组管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/group` | 列出用户组 |
| GET | `/api/v4/auth/admin/group/:id` | 获取用户组 |
| PUT | `/api/v4/auth/admin/group` | 创建用户组 |
| PUT | `/api/v4/auth/admin/group/:id` | 更新用户组 |
| DELETE | `/api/v4/auth/admin/group/:id` | 删除用户组 |

#### 9.5 文件管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/file` | 列出文件 |
| GET | `/api/v4/auth/admin/file/:id` | 获取文件 |
| PUT | `/api/v4/auth/admin/file/:id` | 更新文件 |
| GET | `/api/v4/auth/admin/file/url/:id` | 获取文件 URL |
| POST | `/api/v4/auth/admin/file/batch/delete` | 批量删除 |

#### 9.6 Entity（物理存储）管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/entity` | 列出 Entity |
| GET | `/api/v4/auth/admin/entity/:id` | 获取 Entity |
| POST | `/api/v4/auth/admin/entity/batch/delete` | 批量删除 |
| GET | `/api/v4/auth/admin/entity/url/:id` | 获取 Entity URL |

#### 9.7 存储策略管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/policy` | 列出策略 |
| GET | `/api/v4/auth/admin/policy/:id` | 获取策略 |
| PUT | `/api/v4/auth/admin/policy` | 创建策略 |
| PUT | `/api/v4/auth/admin/policy/:id` | 更新策略 |
| DELETE | `/api/v4/auth/admin/policy/:id` | 删除策略 |
| POST | `/api/v4/auth/admin/policy/cors` | 创建跨域策略 |

#### 10.10 节点管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/node` | 列出节点 |
| GET | `/api/v4/auth/admin/node/:id` | 获取节点 |
| PUT | `/api/v4/auth/admin/node` | 创建节点 |
| PUT | `/api/v4/auth/admin/node/:id` | 更新节点 |
| DELETE | `/api/v4/auth/admin/node/:id` | 删除节点 |
| POST | `/api/v4/auth/admin/node/test` | 测试节点连接 |
| POST | `/api/v4/auth/admin/node/test/downloader` | 测试下载器 |

#### 10.11 OAuth Client 管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/oauthClient` | 列出 OAuth 应用 |
| GET | `/api/v4/auth/admin/oauthClient/:id` | 获取 OAuth 应用 |
| PUT | `/api/v4/auth/admin/oauthClient` | 创建 OAuth 应用 |
| PUT | `/api/v4/auth/admin/oauthClient/:id` | 更新 OAuth 应用 |
| DELETE | `/api/v4/auth/admin/oauthClient/:id` | 删除 OAuth 应用 |
| POST | `/api/v4/auth/admin/oauthClient/batch/delete` | 批量删除 |

#### 10.12 Entity（物理存储 Blob）管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v4/auth/admin/entity` | 列出 Entity Blob |
| GET | `/api/v4/auth/admin/entity/:id` | 获取 Entity |
| POST | `/api/v4/auth/admin/entity/batch/delete` | 批量删除 |
| GET | `/api/v4/auth/admin/entity/url/:id` | 获取 Entity 下载 URL |

#### 10.13 队列任务管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v4/auth/admin/queue/metrics` | 队列指标 |
| POST | `/api/v4/auth/admin/queue` | 列出任务 |
| GET | `/api/v4/auth/admin/queue/:id` | 获取任务 |
| POST | `/api/v4/auth/admin/queue/batch/delete` | 批量删除 |
| POST | `/api/v4/auth/admin/queue/cleanup` | 清理任务 |

#### 9.10 工具接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v4/auth/admin/tool/wopi` | 获取 WOPI Discovery |
| POST | `/api/v4/auth/admin/tool/thumbExecutable` | 测试缩略图生成器 |
| POST | `/api/v4/auth/admin/tool/mail` | 发送测试邮件 |
| DELETE | `/api/v4/auth/admin/tool/entityUrlCache` | 清除 Entity URL 缓存 |

---

## 数据结构

### User（用户）

```json
{
  "id": "abc123",
  "email": "user@example.com",
  "nick": "昵称",
  "avatar": "https://...",
  "status": 1,
  "group": { ...Group },
  "created_at": "2026-01-01T00:00:00Z",
  "two_factor_enabled": true
}
```

### Group（用户组）

```json
{
  "id": "abc123",
  "name": "默认",
  "is_default": true,
  "is_admin": false,
  "permissions": {
    "enabled": ["upload", "download", "archive"]
  }
}
```

### FileResponse（文件）

```json
{
  "type": 1,
  "id": "abc123",
  "name": "document.pdf",
  "created_at": "2026-10-01T00:00:00Z",
  "updated_at": "2026-10-27T00:00:00Z",
  "size": 1048576,
  "metadata": { "key": "value" },
  "path": "/folder/document.pdf",
  "shared": false,
  "owned": true,
  "primary_entity": "entity-hashid",
  "folder_summary": null,
  "extended_info": null
}
```

### StoragePolicy（存储策略）

```json
{
  "id": "abc123",
  "name": "本地存储",
  "type": "local",
  "max_size": 107374182400,
  "allowed_suffix": [],
  "denied_suffix": [],
  "relay": false,
  "chunk_concurrency": 3,
  "encryption": false
}
```

### Token（认证令牌）

```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "dGhpcyBpcyBh...",
  "access_expires": "2026-10-27T12:00:00Z",
  "refresh_expires": "2026-11-03T12:00:00Z"
}
```

### UploadSessionResponse（上传会话）

```json
{
  "session_id": "uuid-session-id",
  "upload_id": "cloudreve-upload-id",
  "chunk_size": 5242880,
  "expires": 1730003600,
  "upload_urls": ["..."],
  "credential": "...",
  "ak": "...",
  "uri": "/target/file.zip",
  "callback_secret": "...",
  "mime_type": "application/zip",
  "storage_policy": { ...StoragePolicy }
}
```

### Share（分享）

```json
{
  "id": "abc123",
  "name": "document.pdf",
  "url": "https://your-site.com/s/abc123",
  "visited": 10,
  "downloaded": 5,
  "expired": false,
  "unlocked": true,
  "password_protected": false,
  "source_type": 1,
  "owner": { ...User },
  "source_uri": "/document.pdf",
  "expires": "2026-11-27T00:00:00Z",
  "size": 1048576
}
```

### DirectLinkResponse（永久直链）

```json
{
  "link": "https://your-site.com/f/abc123/document.pdf",
  "file_url": "/document.pdf"
}
```

---

## URI 格式说明

Cloudreve v4 使用 URI 字符串定位文件，支持几种格式：

1. **路径格式**（推荐）：`/文件夹/文件名.ext`
2. **带策略前缀**：`p/{policy_id}/{文件路径}` （内部使用）
3. **Folder URI**：`f/{folder_path}` （内部使用）

客户端一般只需要用路径格式 `/folder/file.txt`。

---

*文档基于 Cloudreve v4 源码整理，生成日期：2026-10-27*
*信息来源：[routers/router.go](../routers/router.go)、各 service 参数定义、[pkg/serializer/error.go](../pkg/serializer/error.go)*
