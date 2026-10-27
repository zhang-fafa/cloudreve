# Cloudreve v4 Flutter 客户端开发指南

> 本文档为 Cloudreve v4 网盘客户端 Flutter 开发的**完整可执行规范**，涵盖从环境搭建到上架发布的全流程。
> 配套 API 参考：[Cloudreve_API_Reference.md](./Cloudreve_API_Reference.md)

---

## 目录

- [1. 项目初始化与环境配置](#1-项目初始化与环境配置)
- [2. 项目目录结构设计](#2-项目目录结构设计)
- [3. 核心开发规范](#3-核心开发规范)
- [4. 开发调试流程](#4-开发调试流程)
- [5. 测试要求](#5-测试要求)
- [6. 构建打包流程](#6-构建打包流程)
- [7. 完整命令清单](#7-完整命令清单)

---

## 1. 项目初始化与环境配置

### 1.1 Flutter SDK 版本要求

| 组件 | 最低版本 | 推荐版本 | 安装命令 |
|------|----------|----------|----------|
| Flutter SDK | 3.24.0 | **3.27.x stable** | `flutter --version` 检查 |
| Dart SDK | 3.5.0 | **3.6.x** | 随 Flutter 自动安装 |
| Android Studio | Hedgehog (2023.1.1) | Ladybug (2024.2.x) | https://developer.android.com/studio |
| Xcode | 15.0 | **16.x** (macOS 15+) | App Store 安装 |
| CocoaPods | 1.13 | **1.15+** | `sudo gem install cocoapods` |

**初始化环境（一条命令）**：
```bash
# Windows PowerShell
winget install FlutterSDK
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$env:LOCALAPPDATA\Pub\Cache\bin", "User")
flutter doctor -v

# macOS
brew install --cask flutter
flutter doctor -v
```

### 1.2 创建项目

```bash
# 在 workspace 目录下创建项目
cd e:\go-project
flutter create cloudreve_client --org com.cloudreve --project-name cloudreve_client --platforms=android,ios,web
cd cloudreve_client

# 验证项目创建成功
flutter analyze
flutter run -d chrome   # 用浏览器跑起来验证
```

### 1.3 依赖管理规则

#### 1.3.1 pubspec.yaml 完整配置

> 以下依赖**已针对 Cloudreve v4 API 特性预选**（JWT、分片上传、WebDAV、缓存、下载等）。

```yaml
name: cloudreve_client
description: Cloudreve v4 官方 Flutter 客户端
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.6.0 <4.0.0'
  flutter: '>=3.27.0'

dependencies:
  flutter:
    sdk: flutter
  # ===== 网络层 =====
  dio: ^5.7.0                    # HTTP 客户端（拦截器、分片上传、取消请求）
  cookie_jar: ^4.0.8             # Cookie 支持（备选方案：Session Cookie）
  # ===== 状态管理 =====
  get: ^4.6.6                    # 轻量路由 + 依赖注入 + 响应式（生产首选）
  # ===== 本地存储 =====
  shared_preferences: ^2.3.4     # KV 持久化（Token、用户设置）
  hive: ^2.2.3                   # 结构化离线缓存（文件列表、缩略图）
  hive_flutter: ^1.1.0
  # ===== 文件系统 =====
  path_provider: ^2.1.5          # 获取文档目录、缓存目录
  open_filex: ^4.5.0             # 打开本地文件
  file_picker: ^8.1.7            # 选择要上传的文件
  permission_handler: ^11.3.1    # 存储权限请求
  # ===== 工具库 =====
  crypto: ^3.0.6                 # HashID/签名辅助（Cloudreve HashID 基于这个）
  intl: ^0.19.0                  # 日期格式化、国际化
  uuid: ^4.5.1                   # UUID 生成
  flutter_file_downloader: ^1.2.1 # 系统级下载（使用平台原生下载器）
  connectivity_plus: ^6.1.1      # 网络状态检测
  # ===== UI 组件 =====
  extended_image: ^9.1.0         # 图片查看器（支持缩略图）
  flutter_riverpod: ^2.6.1       # Riverpod（备选状态管理，与 Get 二选一）
  # ===== WebDAV（iOS 用户用） =====
  webdav: ^1.2.0                 # WebDAV 客户端（iOS 系统文件 App 备用方案）

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0          # 静态检查（官方推荐）
  hive_generator: ^2.0.1         # Hive 代码生成
  build_runner: ^2.4.14          # 代码生成运行器
  mockito: ^5.4.4                # 单元测试 Mock
  network_image_mock: ^2.1.1     # 网络图片 Mock（Widget 测试）
  flutter_gen_runner: ^5.8.0     # 资源代码生成（可选）

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/icons/
  fonts:
    - family: CloudreveIcon
      fonts:
        - asset: assets/fonts/CloudreveIcon.ttf

# ===== 代码生成配置 =====
# 运行: dart run build_runner build --delete-conflicting-outputs
```

#### 1.3.2 安装依赖

```bash
cd e:\go-project\cloudreve_client

# 拉取所有依赖
flutter pub get

# 生成 Hive 适配器代码（首次或修改 Hive Model 后运行）
dart run build_runner build --delete-conflicting-outputs

# 升级所有依赖到最新兼容版本
flutter pub upgrade --major-versions
```

#### 1.3.3 pubspec.lock 提交规则

```yaml
# ✅ 必须提交 pubspec.lock 到 git
# ✅ 禁止手动修改 pubspec.lock
# ✅ 禁止依赖使用 ^x.y.z 以外的版本约束（避免不可控升级）
# ❌ 禁止将 git/本地依赖用于生产
```

### 1.4 代码格式化与静态检查配置

#### 1.4.1 analysis_options.yaml（必须放在项目根目录）

```yaml
# Cloudreve 客户端专用静态检查配置
include: package:flutter_lints/flutter.yaml

analyzer:
  exclude:
    - build/**
    - "**/*.g.dart"
    - "**/*.freezed.dart"
  errors:
    # 严重错误（必须修复）
    invalid_assignment: error
    missing_return: error
    dead_code: error
    null_check_on_nullable_type_parameter: error
    # 警告（建议修复）
    unused_import: warning
    unused_local_variable: warning
    prefer_const_constructors: warning
    prefer_single_quotes: warning
    avoid_print: warning
    # 风格提示（可选）
    camel_case_types: info
    constant_identifier_names: info

linter:
  rules:
    # === 强制开启 ===
    - always_declare_return_types
    - prefer_single_quotes
    - prefer_const_constructors
    - prefer_const_literals_to_create_immutables
    - sort_child_properties_last
    - use_key_in_widget_constructors
    - must_call_super
    # === Cloudreve 项目特有 ===
    - prefer_relative_imports            # 禁止 package import
    - one_member_abstracts
    - avoid_classes_with_only_static_members
    - prefer_final_fields
    - prefer_final_locals
    - avoid_catches_without_on_clauses
    - prefer_expression_function_bodies
    - sized_box_for_whitespace
    # === 关闭（与项目风格冲突）===
    # - flutter_style_todos               # 不强制 TODO 格式
```

#### 1.4.2 .clang-format（iOS/Android 原生层可选）

Flutter 项目主用 Dart，通常不需要。如果 Platform Channel 写了 Swift/Kotlin，用对应官方配置。

#### 1.4.3 自动格式化命令

```bash
# 格式化所有 Dart 文件
dart format lib test

# 只格式化新增/修改文件（提交前）
git diff --name-only -- '*.dart' | xargs dart format

# 一键修复 lint 可自动修复的问题
dart fix --apply

# 按 lint 提示修复 + 格式化
dart fix --apply && dart format lib test

# 完整静态检查（CI 会跑这个）
flutter analyze
```

#### 1.4.4 VS Code 配置（.vscode/settings.json）

```json
{
  "dart.flutterSdkPath": "C:\\src\\flutter",
  "dart.sdkPath": "C:\\src\\flutter\\bin\\cache\\dart-sdk",
  "dart.enableSdkFormatter": true,
  "dart.lineLength": 120,
  "dart.analysisExcludedFolders": ["build"],
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "Dart-Code.dart-code",
  "editor.codeActionsOnSave": {
    "source.fixAll": "explicit",
    "source.organizeImports": "explicit"
  },
  "files.exclude": {
    "**/.git": true,
    "**/.svn": true,
    "**/.hg": true,
    "**/CVS": true,
    "**/.DS_Store": true,
    "**/Thumbs.db": true,
    "**/.pub-cache": true,
    "**/build": true,
    "**/.dart_tool": true
  },
  "search.exclude": {
    "**/build": true,
    "**/.dart_tool": true,
    "**/ios/Pods": true,
    "**/android/.gradle": true
  }
}
```

#### 1.4.5 Git Hooks（自动检查）

```bash
# 创建 pre-commit hook
mkdir -p .git/hooks
cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/bash
echo "=== Cloudreve Flutter Client pre-commit ==="
dart format --set-exit-if-changed lib test
if [ $? -ne 0 ]; then
  echo "❌ 检测到未格式化文件，请运行 dart format"
  exit 1
fi

flutter analyze
if [ $? -ne 0 ]; then
  echo "❌ flutter analyze 失败，请修复"
  exit 1
fi

echo "✅ 检查通过"
HOOK
chmod +x .git/hooks/pre-commit
```

---

## 2. 项目目录结构设计

### 2.1 整体架构：Feature-First + Clean Architecture

> 采用 **Feature-First** 组织业务功能，每个 Feature 内部使用 **Clean Architecture** 分层。
> 这样 Cloudreve 的每个 API 模块（Auth / Explorer / Share / User / Admin）对应一个 Feature，后续加 SSE/WebDAV 扩展也很清晰。

```
cloudreve_client/
├── lib/
│   ├── main.dart                          # ✅ 应用入口
│   ├── bootstrap.dart                    # 启动初始化（DI、数据库、日志）
│   ├── app.dart                           # MaterialApp 配置（路由、主题、国际化）
│   │
│   ├── core/                              # 🔧 工具层（跨 Feature 复用）
│   │   ├── network/                       # Dio 封装 + 拦截器 + Token 自动刷新
│   │   │   ├── api_client.dart            # 统一的 Dio Instance + BaseOptions
│   │   │   ├── auth_interceptor.dart      # JWT 注入 + 401 自动刷新（关键 Cloudreve 特性）
│   │   │   ├── sign_interceptor.dart      # Cloudreve HMAC 签名拦截器（文件下载用）
│   │   │   ├── cache_interceptor.dart     # 离线缓存拦截器
│   │   │   └── api_exception.dart        # 统一异常（映射 Cloudreve 错误码）
│   │   ├── storage/                       # 本地存储封装
│   │   │   ├── secure_storage.dart        # Token / 敏感数据
│   │   │   ├── cache_storage.dart         # Hive 离线缓存（文件列表、缩略图）
│   │   │   └── download_helper.dart       # 下载记录、系统调用
│   │   ├── hashid/                        # Cloudreve HashID 编解码（原生 Dart 实现）
│   │   │   └── hashid.dart
│   │   ├── theme/                         # 主题色、TextStyle、Dark Mode
│   │   │   ├── app_theme.dart
│   │   │   └── colors.dart
│   │   ├── routes/                        # 路由常量（不涉及 Feature 跳转逻辑）
│   │   │   └── app_routes.dart
│   │   ├── utils/                         # 通用工具函数
│   │   │   ├── file_utils.dart            # 文件大小格式化（bytes → MB/GB）
│   │   │   ├── uri_utils.dart             # Cloudreve URI 构造解析
│   │   │   ├── validators.dart            # 邮箱、密码、URL 校验
│   │   │   └── logger.dart                # 日志输出（Debug 模式 print，Release 埋点）
│   │   └── extensions/                    # Dart Extension
│   │       ├── context_extension.dart     # context.snackbar、context.pop
│   │       ├── string_extension.dart      # 安全的空值、文件后缀提取
│   │       └── int_extension.dart         # Byte 转人类可读
│   │
│   ├── models/                            # 📦 数据层 —— Cloudreve API 响应模型
│   │   ├── token.dart                     # Cloudreve /session/token 响应
│   │   ├── user.dart                      # User、Group、UserSettings
│   │   ├── file.dart                      # FileResponse、ListResponse、StoragePolicy
│   │   ├── share.dart                     # Share、DirectLink
│   │   ├── upload.dart                    # UploadSessionResponse
│   │   ├── task.dart                      # TaskResponse、TaskListResponse
│   │   ├── site.dart                      # Ping、SiteConfig
│   │   └── exception.dart                 # 业务异常类 + Cloudreve 错误码枚举
│   │
│   ├── services/                          # 🌐 数据层 —— 直接调用 API（Repository）
│   │   ├── auth_service.dart              # POST /session/token, /session/token/refresh
│   │   ├── user_service.dart              # GET /auth/user/me, PATCH /auth/user/setting
│   │   ├── explorer_service.dart          # GET /file, GET /file/info, POST /file/url...
│   │   ├── upload_service.dart            # PUT /file/upload, POST /file/upload/:id/:index
│   │   ├── share_service.dart             # PUT /share, GET /share, DELETE /share/:id
│   │   ├── workflow_service.dart          # POST /workflow/archive, POST /workflow/download
│   │   ├── site_service.dart              # GET /site/ping, GET /site/config
│   │   └── admin_service.dart             # 管理后台接口（可选）
│   │
│   ├── features/                          # 🎯 Feature 层（业务逻辑 + UI）
│   │   ├── auth/                          # 登录、注册、重置密码
│   │   │   ├── presentation/              # UI 层
│   │   │   │   ├── pages/
│   │   │   │   │   ├── login_page.dart
│   │   │   │   │   ├── register_page.dart
│   │   │   │   │   └── forgot_password_page.dart
│   │   │   │   ├── widgets/
│   │   │   │   │   ├── login_form.dart
│   │   │   │   │   └── password_field.dart
│   │   │   │   └── controllers/           # GetxController 或 Riverpod Notifier
│   │   │   │       └── auth_controller.dart
│   │   │   └── bindings/
│   │   │       └── auth_binding.dart
│   │   │
│   │   ├── explorer/                      # 文件浏览（核心）
│   │   │   ├── presentation/
│   │   │   │   ├── pages/
│   │   │   │   │   ├── explorer_page.dart            # 主文件浏览页
│   │   │   │   │   ├── file_preview_page.dart        # 图片/视频/文档预览
│   │   │   │   │   ├── file_detail_page.dart
│   │   │   │   │   └── search_page.dart
│   │   │   │   ├── widgets/
│   │   │   │   │   ├── file_list_tile.dart
│   │   │   │   │   ├── file_grid_view.dart
│   │   │   │   │   ├── upload_progress_overlay.dart
│   │   │   │   │   └── bottom_sheet_actions.dart
│   │   │   │   └── controllers/
│   │   │   │       └── explorer_controller.dart
│   │   │   └── bindings/
│   │   │       └── explorer_binding.dart
│   │   │
│   │   ├── upload/                        # 上传流程（分片）
│   │   │   ├── presentation/
│   │   │   │   ├── pages/upload_queue_page.dart
│   │   │   │   └── controllers/upload_controller.dart
│   │   │   └── bindings/
│   │   │       └── upload_binding.dart
│   │   │
│   │   ├── download/                      # 下载记录
│   │   │   └── ...
│   │   │
│   │   ├── share/                         # 分享管理
│   │   │   ├── presentation/
│   │   │   │   ├── pages/my_shares_page.dart
│   │   │   │   ├── pages/share_create_page.dart
│   │   │   │   └── controllers/share_controller.dart
│   │   │   └── bindings/
│   │   │
│   │   ├── user/                          # 用户中心
│   │   │   ├── presentation/
│   │   │   │   ├── pages/profile_page.dart
│   │   │   │   ├── pages/user_setting_page.dart
│   │   │   │   └── pages/storage_usage_page.dart
│   │   │   └── bindings/
│   │   │
│   │   ├── setting/                       # 应用设置
│   │   │   └── presentation/pages/setting_page.dart
│   │   │
│   │   └── home/                          # 底部 Tab 容器
│   │       └── presentation/pages/main_tab_page.dart
│   │
│   └── l10n/                              # 🌍 国际化
│       ├── app_en.arb
│       ├── app_zh.arb
│       └── app_localizations.dart         # 自动生成（flutter gen-l10n）
│
├── test/                                  # 🧪 测试
│   ├── unit/
│   │   ├── models/
│   │   │   └── file_test.dart
│   │   ├── services/
│   │   │   ├── auth_service_test.dart
│   │   │   ├── explorer_service_test.dart
│   │   │   └── upload_service_test.dart
│   │   ├── core/
│   │   │   ├── hashid_test.dart
│   │   │   └── uri_utils_test.dart
│   │   └── controllers/
│   │       └── explorer_controller_test.dart
│   ├── widget/
│   │   ├── auth/
│   │   │   └── login_page_test.dart
│   │   └── explorer/
│   │       └── file_list_tile_test.dart
│   └── integration/
│       ├── auth_flow_test.dart
│       └── explorer_flow_test.dart
│
├── assets/
│   ├── images/
│   ├── icons/
│   └── fonts/
│
├── android/                    # Android 原生层
├── ios/                         # iOS 原生层
├── web/                         # Web 输出
├── windows/                     # Windows 输出（可选）
├── linux/                       # Linux 输出（可选）
├── macos/                       # macOS 输出（可选）
│
├── pubspec.yaml
├── pubspec.lock
├── analysis_options.yaml
├── README.md
├── .gitignore
└── .vscode/
    └── settings.json
```

### 2.2 各层职责定义

```
┌─────────────────────────────────────────────────────────────┐
│  Presentation Layer（UI 层）                                │
│  - 只负责渲染 Widget + 响应用户交互                          │
│  - 所有业务逻辑通过 Controller 调用                          │
│  - Widget 状态变化只来自 Controller .obs                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Controller（GetX）或 Notifier（Riverpod）             │  │
│  │  - 持有 State（文件列表、上传进度、当前路径）           │  │
│  │  - 调用 Services 完成业务                               │  │
│  │  - 处理页面跳转、Toast、Dialog                          │  │
│  └───────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Services Layer（业务逻辑层）                               │
│  - 定义 Repository 接口                                    │
│  - 实现业务规则（重试、限流、合并多个接口）                  │
│  - 调用 API Client 或 本地 Cache                            │
│  - 不感知 UI，可独立单元测试                                │
├─────────────────────────────────────────────────────────────┤
│  Services（数据层）                                         │
│  - 直接调用 Cloudreve API via Dio                          │
│  - 解析 JSON → Model                                       │
│  - 处理网络错误                                            │
│  - 实现 Repository 接口                                    │
├─────────────────────────────────────────────────────────────┤
│  Core Layer（工具层）                                       │
│  - Dio 实例 + 所有拦截器                                   │
│  - Token 自动刷新（Auth Interceptor）                       │
│  - HashID 编解码                                           │
│  - 日志、缓存、主题                                        │
│  - 全项目共享                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 关键模块说明

| 模块 | 路径 | 对应 Cloudreve API | 说明 |
|------|------|---------------------|------|
| AuthService | `lib/services/auth_service.dart` | `POST /session/token`, `POST /session/token/refresh` | JWT 登录、刷新、退出 |
| ExplorerService | `lib/services/explorer_service.dart` | `GET /file`, `GET /file/info`, `POST /file/url` | 文件列表、详情、下载 URL |
| UploadService | `lib/services/upload_service.dart` | `PUT /file/upload`, `POST /file/upload/:id/:index` | **分片上传核心**（Cloudreve 特性） |
| ShareService | `lib/services/share_service.dart` | `PUT /share`, `DELETE /share/:id` | 创建/删除分享链接 |
| AuthInterceptor | `lib/core/network/auth_interceptor.dart` | 拦截所有请求 Header 注入 + 401 触发 `/session/token/refresh` | **所有 API 都用** |
| HashID | `lib/core/hashid/hashid.dart` | 内部算法 | Cloudreve 所有 `:id` 参数都需要编码 |

---

## 3. 核心开发规范

### 3.1 命名约定

| 类型 | 规则 | 示例 |
|------|------|------|
| **文件名** | snake_case | `auth_service.dart`, `explorer_controller.dart` |
| **类 / 枚举 / typedef** | PascalCase | `AuthService`, `FileResponse`, `OperationType` |
| **函数 / 方法 / 变量 / 参数** | lowerCamelCase | `fetchFileList()`, `currentPath`, `fileUri` |
| **私有成员** | 前导下划线 | `_tokenStorage`, `_fetchInternal()` |
| **常量** | lowerCamelCase 或 PascalCase | `maxChunkSize` / `ApiBaseURL` |
| **路由名** | kebab-case | `/login`, `/explorer/detail` |
| **资源文件** | snake_case | `app_icon.png`, `cloudreve_logo.svg` |
| **Test 文件** | `原文件名_test.dart` | `auth_service_test.dart` |

**禁止清单**：
- ❌ 匈牙利命名（`strName`, `iCount`）
- ❌ 缩写（`btn`→`button`, `lst`→`list`, `svc`→`service`）
- ❌ 下划线分隔的函数名（`fetch_file_list`）
- ❌ `l`, `O`, `I` 这类单字母变量（循环索引 `i`, `j`, `k` 允许）

### 3.2 组件编写规范

#### 3.2.1 StatelessWidget / StatefulWidget 选择

```
✅ StatelessWidget  —— 无内部可变状态（父 Widget 传入所有数据）
❌ 不用 StatefulWidget  —— 仅仅是为了"可以 setState"

✅ StatefulWidget  —— 必须有本地状态（动画、滚动位置、局部 UI 状态）
✅ GetxController  —— 页面级全局业务状态（文件列表、当前路径、上传进度）
```

#### 3.2.2 Widget 构造规范

```dart
// ✅ 正确
class FileListTile extends StatelessWidget {
  const FileListTile({
    super.key,
    required this.file,
    required this.onTap,
    required this.onLongPress,
    this.selected = false,
  });

  final FileResponse file;
  final VoidCallback onTap;
  final VoidCallback onLongPress;
  final bool selected;
  // ...
}

// ❌ 错误 —— positional 参数、非 const、没有 key
class FileListTile extends StatelessWidget {
  FileListTile(FileResponse file, VoidCallback onTap, {selected = false});
  // ...
}
```

#### 3.2.3 Controller 规范（GetX）

> **状态管理方案**：项目选用 **GetX** 作为主力状态管理 + 路由框架。
>
> **原因**：
> - Cloudreve 客户端需要大量全局状态（当前路径、文件列表、上传队列、下载记录）
> - GetX 的 `.obs` / `.value` 对列表嵌套状态追踪友好
> - 一套库同时解决路由、依赖注入、状态响应，减少依赖数量
> - Controller 自动生命周期管理（页面退出自动 dispose）
>
> **备选方案**：Riverpod（如果你不喜欢 GetX 黑魔法式 API）。两者选其一，不要混用。

```dart
// ✅ 正确 —— Controller 模板
// lib/features/explorer/presentation/controllers/explorer_controller.dart

class ExplorerController extends GetxController {
  ExplorerController({required ExplorerService service})
      : _service = service;

  final ExplorerService _service;

  // ===== 可观察状态（页面自动刷新）=====
  final RxString currentPath = '/'.obs;
  final RxList<FileResponse> files = <FileResponse>[].obs;
  final RxBool isLoading = false.obs;
  final Rx<AppException?> error = Rx<AppException?>(null);
  final Rx<StoragePolicy?> policy = Rx<StoragePolicy?>(null);
  final RxInt selectedCount = 0.obs;
  final RxBool selectionMode = false.obs;

  // ===== 公开方法 =====
  Future<void> loadDirectory({bool refresh = false}) async {
    isLoading.value = true;
    error.value = null;
    try {
      final response = await _service.listDirectory(uri: currentPath.value);
      files.assignAll(response.files);
      policy.value = response.storagePolicy;
    } on AppException catch (e) {
      error.value = e;
      Get.snackbar('错误', e.userMessage);
    } finally {
      isLoading.value = false;
    }
  }

  void selectFile(FileResponse file) { /* ... */ }
  void enterDirectory(FileResponse folder) { /* ... */ }
  void goBack() { /* ... */ }
}
```

**Controller 规则**：
- ✅ Controller 通过 Binding 注入 Service
- ✅ 一个 Controller 只管一个 Feature（ExplorerController 不管 Login 逻辑）
- ✅ 业务错误抛 `AppException`，Controller catch 后统一处理（Snackbar / Dialog）
- ❌ Controller 里写 UI 代码 / 直接操作 `Navigator`（用 `Get.to()` 算合法）
- ❌ Controller 持有 BuildContext（通过 `Get.context` 获取）
- ❌ 跨 Controller 直接通信（用 `Get.find<OtherController>()` 或事件总线）

### 3.3 路由导航规范

> 路由方案：**GetX Navigation**（和状态管理统一）

```dart
// lib/core/routes/app_routes.dart
class AppRoutes {
  static const String splash = '/';
  static const String login = '/login';
  static const String register = '/register';
  static const String forgotPassword = '/forgot-password';
  static const String mainTab = '/main';
  static const String explorer = '/explorer';       // 带参数: /explorer?uri=...
  static const String fileDetail = '/file/detail';
  static const String filePreview = '/file/preview';
  static const String uploadQueue = '/upload/queue';
  static const String myShares = '/shares';
  static const String createShare = '/share/create';
  static const String profile = '/profile';
  static const String userSetting = '/setting';
  static const String storageUsage = '/storage';
}

// lib/app.dart 里注册
GetMaterialApp(
  initialRoute: AppRoutes.splash,
  getPages: [
    GetPage(name: AppRoutes.splash, page: () => const SplashPage()),
    GetPage(
      name: AppRoutes.login,
      page: () => const LoginPage(),
      binding: AuthBinding(),
    ),
    GetPage(
      name: AppRoutes.explorer,
      page: () => const ExplorerPage(),
      binding: ExplorerBinding(),
    ),
    // ...
  ],
  unknownRoute: GetPage(name: '/notfound', page: () => const NotFoundPage()),
)
```

**导航操作**：
```dart
// ✅ 导航（新页面进栈）
Get.toNamed(AppRoutes.explorer, parameters: {'uri': '/documents'});

// ✅ 替换当前页面（登录后跳主页）
Get.offAllNamed(AppRoutes.mainTab);

// ✅ 带参数
final result = await Get.toNamed(AppRoutes.fileDetail, arguments: fileResponse);

// ✅ 返回
Get.back();

// ❌ 禁止 Navigator.push / Navigator.pop（统一用 Get.to / Get.back）
```

### 3.4 网络请求实现标准

#### 3.4.1 Dio 客户端（核心）

```dart
// lib/core/network/api_client.dart

import 'package:dio/dio.dart';
import 'package:dio_cookie_manager/dio_cookie_manager.dart';
import 'package:cookie_jar/cookie_jar.dart';

class ApiClient {
  static const String baseUrl = 'https://your-cloudreve-site.com';
  static const String apiPrefix = '/api/v4';

  late final Dio dio;
  late final PersistCookieJar cookieJar;

  static final ApiClient instance = ApiClient._singleton();
  ApiClient._singleton() {
    final options = BaseOptions(
      baseUrl: '$baseUrl$apiPrefix',
      connectTimeout: const Duration(seconds: 15),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 60),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'User-Agent': 'CloudreveFlutter/1.0',
      },
    );

    dio = Dio(options);
    cookieJar = PersistCookieJar(
      storage: FileStorage('${Directory.systemTemp.path}/cloudreve_cookies'),
    );

    // ===== 拦截器顺序很重要 =====
    dio.interceptors.addAll([
      AuthInterceptor(),          // 1. 注入 Token
      SignInterceptor(),          // 2. 为需要签名的 URL 加 HMAC
      CookieManager(cookieJar),   // 3. Session Cookie 支持
      CacheInterceptor(),         // 4. 离线缓存
      LogInterceptor(             // 5. Debug 日志（Release 自动关闭）
        request: true,
        requestHeader: true,
        requestBody: true,
        responseHeader: false,
        responseBody: kDebugMode,
        error: true,
        logPrint: (o) => debugPrint('[API] $o'),
      ),
      RetryInterceptor(),         // 6. 失败重试（仅幂等请求）
    ]);
  }
}
```

#### 3.4.2 Auth 拦截器（自动 JWT 注入 + 401 刷新）

```dart
// lib/core/network/auth_interceptor.dart

class AuthInterceptor extends Interceptor {
  AuthInterceptor();

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await TokenStorage.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      final refreshToken = await TokenStorage.getRefreshToken();
      if (refreshToken != null) {
        try {
          // Cloudreve POST /session/token/refresh
          final response = await Dio().post(
            '${ApiClient.baseUrl}/api/v4/session/token/refresh',
            data: {'refresh_token': refreshToken},
          );
          final newTokens = TokenResponse.fromJson(response.data['data']);
          await TokenStorage.saveTokens(newTokens);

          // 重试原请求（带新 Token）
          final retryOptions = err.requestOptions;
          retryOptions.headers['Authorization'] = 'Bearer ${newTokens.accessToken}';
          handler.resolve(await Dio().fetch(retryOptions));
          return;
        } catch (_) {
          // Refresh 也失败 → 跳登录
          await TokenStorage.clear();
          Get.offAllNamed(AppRoutes.login);
        }
      } else {
        Get.offAllNamed(AppRoutes.login);
      }
    }
    handler.next(err);
  }
}
```

#### 3.4.3 Service 实现示例（ExplorerService）

```dart
// lib/services/explorer_service.dart

class ExplorerService {
  final ApiClient _api = ApiClient.instance;

  /// 列出目录文件 → GET /api/v4/file?uri=...
  Future<ListResponse> listDirectory({
    required String uri,
    int page = 0,
    int pageSize = 50,
    String? orderBy,
    String? orderDirection,
  }) async {
    final response = await _api.dio.get(
      '/file',
      queryParameters: {
        'uri': uri,
        'page': page,
        'page_size': pageSize,
        if (orderBy != null) 'order_by': orderBy,
        if (orderDirection != null) 'order_direction': orderDirection,
      },
    );
    return _handleResponse(response, (json) => ListResponse.fromJson(json));
  }

  /// 获取文件详情 → GET /api/v4/file/info?uri=...
  Future<FileResponse> getFileInfo({String? uri, String? id}) async {
    final response = await _api.dio.get(
      '/file/info',
      queryParameters: {
        if (uri != null) 'uri': uri,
        if (id != null) 'id': id,
      },
    );
    return _handleResponse(response, (json) => FileResponse.fromJson(json));
  }

  /// 获取临时下载 URL → POST /api/v4/file/url
  Future<FileURLResponse> getDownloadUrl({
    required List<String> uris,
    bool download = true,
    bool archive = false,
    bool skipError = false,
  }) async {
    final response = await _api.dio.post(
      '/file/url',
      data: {
        'uris': uris,
        'download': download,
        'archive': archive,
        'skip_error': skipError,
      },
    );
    return _handleResponse(response, (json) => FileURLResponse.fromJson(json));
  }

  /// 重命名 → POST /api/v4/file/rename
  Future<FileResponse> rename({
    required String uri,
    required String newName,
  }) async {
    final response = await _api.dio.post(
      '/file/rename',
      data: {'uri': uri, 'new_name': newName},
    );
    return _handleResponse(response, (json) => FileResponse.fromJson(json));
  }

  /// 移动/复制 → POST /api/v4/file/move
  Future<void> move({
    required List<String> uris,
    required String dst,
    bool copy = false,
  }) async {
    await _api.dio.post(
      '/file/move',
      data: {'uris': uris, 'dst': dst, 'copy': copy},
    );
  }

  /// 删除 → DELETE /api/v4/file
  Future<void> delete({
    required List<String> uris,
    bool skipSoftDelete = false,
  }) async {
    await _api.dio.delete(
      '/file',
      data: {'uris': uris, 'skip_soft_delete': skipSoftDelete},
    );
  }

  /// 获取缩略图 → GET /api/v4/file/thumb?uri=...
  Future<FileThumbResponse> getThumb(String uri) async {
    final response = await _api.dio.get(
      '/file/thumb',
      queryParameters: {'uri': uri},
    );
    return _handleResponse(response, (json) => FileThumbResponse.fromJson(json));
  }

  /// 打包下载 → POST /api/v4/file/url {archive: true}
  Future<String> getArchiveUrl(List<String> uris) async {
    final response = await _api.dio.post(
      '/file/url',
      data: {'uris': uris, 'archive': true},
    );
    final data = response.data['data'] as Map<String, dynamic>;
    return (data['urls'] as List).first['url'] as String;
  }

  /// Cloudreve 统一响应处理
  T _handleResponse<T>(Response response, T Function(dynamic) parser) {
    final body = response.data as Map<String, dynamic>;
    final code = body['code'] as int?;

    if (code == 0) {
      return parser(body['data']);
    }
    // Cloudreve 错误码映射
    throw CloudreveException.fromCode(code ?? -1, body['msg'] ?? 'Unknown error');
  }
}
```

#### 3.4.4 分片上传实现（Cloudreve 核心特性）

> Cloudreve v4 要求客户端自己计算分片大小并逐片上传。流程见 API 文档第 5 节。

```dart
// lib/services/upload_service.dart

class UploadService {
  final ApiClient _api = ApiClient.instance;
  final int chunkSize = 5 * 1024 * 1024; // 5MB 分片

  /// 创建上传会话 → PUT /api/v4/file/upload
  Future<UploadSessionResponse> createSession({
    required String uri,
    required int size,
    String? mimeType,
    String? policyId,
    int? lastModified,
  }) async {
    final response = await _api.dio.put(
      '/file/upload',
      data: {
        'uri': uri,
        'size': size,
        if (mimeType != null) 'mime_type': mimeType,
        if (policyId != null) 'policy_id': policyId,
        if (lastModified != null) 'last_modified': lastModified,
      },
    );
    return UploadSessionResponse.fromJson(response.data['data']);
  }

  /// 上传单个分片 → POST /api/v4/file/upload/:sessionId/:index
  Future<void> uploadChunk({
    required String sessionId,
    required int index,
    required Stream<List<int>> stream,
    required int chunkLength,
    void Function(int sent, int total)? onProgress,
  }) async {
    await _api.dio.post(
      '/file/upload/$sessionId/$index',
      data: stream,
      options: Options(
        contentType: 'application/octet-stream',
        headers: {'Content-Length': chunkLength.toString()},
      ),
      onSendProgress: (sent, total) {
        onProgress?.call(sent, total);
      },
    );
  }

  /// 取消上传会话 → DELETE /api/v4/file/upload
  Future<void> cancelSession({
    required String sessionId,
    required String uri,
  }) async {
    await _api.dio.delete(
      '/file/upload',
      data: {'id': sessionId, 'uri': uri},
    );
  }

  /// 完整上传流程（大文件分片）
  Stream<UploadProgress> uploadFile({
    required String localPath,
    required String remoteUri,
    String? policyId,
  }) async* {
    final file = File(localPath);
    final fileSize = await file.length();
    final mimeType = lookupMimeType(localPath);

    // Step 1: 创建会话
    final session = await createSession(
      uri: remoteUri,
      size: fileSize,
      mimeType: mimeType,
      policyId: policyId,
      lastModified: await file.lastModified().millisecondsSinceEpoch,
    );

    final sessionId = session.sessionId;
    final totalChunks = (fileSize / chunkSize).ceil();
    int uploadedBytes = 0;

    // Step 2: 逐片上传
    for (int i = 0; i < totalChunks; i++) {
      final start = i * chunkSize;
      final end = min(start + chunkSize, fileSize);
      final chunkLength = end - start;

      final chunkStream = file.openRead(start, end);
      await uploadChunk(
        sessionId: sessionId,
        index: i,
        stream: chunkStream,
        chunkLength: chunkLength,
      );

      uploadedBytes += chunkLength;
      yield UploadProgress(
        sessionId: sessionId,
        uploaded: uploadedBytes,
        total: fileSize,
        currentChunk: i,
        totalChunks: totalChunks,
      );
    }
    // Cloudreve 收到所有分片后自动 CompleteUpload
  }
}
```

### 3.5 本地缓存实现标准

#### 3.5.1 三级缓存策略

```
L1: 内存缓存（Controller 中的 .obs 列表，跟随页面生命周期）
    ↓ Miss
L2: Hive 离线缓存（用户上次浏览的目录 + 缩略图 URL）
    ↓ Miss
L3: 网络请求（Dio → Cloudreve API）
    ↓ 成功 → 同时回写 L1 + L2
```

#### 3.5.2 Hive 定义

```dart
// lib/core/storage/cache_storage.dart

@HiveType(typeId: 0)
class CachedFileList {
  @HiveField(0) final String uri;
  @HiveField(1) final List<FileResponse> files;
  @HiveField(2) final DateTime cachedAt;

  CachedFileList({required this.uri, required this.files, required this.cachedAt});
}

// 使用
class CacheStorage {
  late final Box<CachedFileList> fileListBox;

  Future<void> init() async {
    await Hive.initFlutter();
    Hive.registerAdapter(CachedFileListAdapter());
    fileListBox = await Hive.openBox<CachedFileList>('file_lists');
  }

  CachedFileList? getFileList(String uri) => fileListBox.get(uri);

  Future<void> putFileList(CachedFileList list) async {
    await fileListBox.put(list.uri, list);
  }
}
```

### 3.6 Cloudreve 异常处理

```dart
// lib/models/exception.dart

class CloudreveException implements Exception {
  final int code;
  final String message;
  final String? correlationId;

  CloudreveException({required this.code, required this.message, this.correlationId});

  factory CloudreveException.fromCode(int code, String msg) {
    final mapping = {
      40021: '用户不存在',
      40022: '两步验证代码错误',
      40011: '上传会话已过期',
      40013: '分片大小不匹配',
      40051: '用户容量不足',
      40044: '文件不存在',
      40073: '文件正在被占用',
      40089: 'OAuth 权限不足',
      50001: '服务器数据库错误',
      50010: '节点离线',
    };
    return CloudreveException(
      code: code,
      message: mapping[code] ?? msg,
    );
  }

  String get userMessage => message;

  @override
  String toString() => 'CloudreveException[$code]: $message';
}
```

---

## 4. 开发调试流程

### 4.1 本地运行命令

```bash
# 查看可用设备列表
flutter devices

# 用指定设备运行
flutter run -d chrome           # Chrome
flutter run -d macos            # macOS
flutter run -d windows          # Windows
flutter run -d emulator-5554    # Android Emulator
flutter run -d 'iPhone 16'      # iOS Simulator

# 指定 Dart Entrypoint
flutter run -t lib/main.dart -d chrome

# 指定 flavor（开发/测试/生产）
flutter run --flavor development -t lib/main_development.dart
```

### 4.2 热重载规范

| 操作 | 快捷键 | 说明 |
|------|--------|------|
| Hot Reload（保留状态） | `r` / `F5` | 修改 Widget / Controller / 业务代码后 **首选** |
| Hot Restart（重置状态） | `R` / `Shift+F5` | 新增了全局变量、更改了 static 初始化时 |
| Quit | `q` | 退出运行 |
| Service Protocol | `v` | 打开 DevTools 链接 |

**哪些改动需要 Hot Restart**：
- ✅ 修改 `main()` 函数逻辑
- ✅ 修改 `static` / 全局变量初始值
- ✅ 修改 Theme / GetMaterialApp 顶层配置
- ❌ 普通 Widget 改动 → 用 Reload

### 4.3 常用调试工具

#### 4.3.1 DevTools

```bash
# 运行时自动打开（或手动）
flutter pub global activate devtools
dart run devtools

# 附加到正在运行的 App
flutter attach
```

**DevTools 关键面板**：

| 面板 | 用途 |
|------|------|
| **Inspector** | 查看 Widget Tree、定位 UI 问题 |
| **Network** | 查看 Dio 请求/响应（Cloudreve API 调试核心） |
| **Memory** | 内存泄漏检测 |
| **Performance** | UI 帧率、Jank 排查 |
| **Debugger** | 断点、单步、变量查看 |
| **Logging** | 应用内所有 `print`/`debugPrint` 输出 |

#### 4.3.2 Dio Network Inspector（推荐）

```yaml
# pubspec.yaml dev_dependencies 加一个
dio_http_formatter: ^2.3.1
```

```dart
// 调试时用
if (kDebugMode) {
  dio.interceptors.add(DioHttpFormatter());
}
```

#### 4.3.3 断点调试

```dart
// 关键位置打断点行：
// - AuthInterceptor.onError → 401 刷新流程
// - UploadService.uploadChunk → 分片上传进度
// - ExplorerController.loadDirectory → 列表解析
```

### 4.4 日志输出标准

```dart
// lib/core/utils/logger.dart

enum LogLevel { verbose, debug, info, warning, error }

class AppLogger {
  static const _enabled = kDebugMode;

  static void v(String tag, String msg) {
    if (_enabled) debugPrint('[V/$tag] $msg');
  }

  static void d(String tag, String msg) {
    if (_enabled) debugPrint('[D/$tag] $msg');
  }

  static void i(String tag, String msg) => debugPrint('[I/$tag] $msg');

  static void w(String tag, String msg, [StackTrace? stack]) {
    debugPrint('[W/$tag] $msg');
    if (stack != null) debugPrint(stack.toString());
  }

  static void e(String tag, String msg, [dynamic error, StackTrace? stack]) {
    debugPrint('[E/$tag] $msg');
    if (error != null) debugPrint(error.toString());
    if (stack != null) debugPrint(stack.toString());
  }
}

// 使用
AppLogger.i('AuthService', '登录成功');
AppLogger.e('ExplorerService', '文件列表加载失败', e, stackTrace);
```

**日志规范**：
- ❌ 禁止裸 `print('xxx')` —— 必须走 `AppLogger`
- ✅ Tag 用 Feature 名 + 类名（`AuthService`, `UploadController`）
- ✅ Release 模式下 debugPrint 会被框架自动剥离

---

## 5. 测试要求

### 5.1 测试类型与执行命令

```bash
# 运行所有测试
flutter test

# 只跑单元测试
flutter test test/unit/

# 只跑某个文件
flutter test test/unit/services/auth_service_test.dart

# 覆盖率报告（HTML）
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
# 然后用浏览器打开 coverage/html/index.html

# 运行 Widget 测试
flutter test test/widget/

# 运行集成测试（真机/模拟器）
flutter test integration_test/
# 或
flutter run integration_test/auth_flow_test.dart -d emulator-5554

# 一次跑完测试 + 覆盖率
flutter test --coverage && genhtml coverage/lcov.info -o coverage/html
```

### 5.2 单元测试示例（AuthService）

```dart
// test/unit/services/auth_service_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';

@GenerateMocks([Dio])
import 'auth_service_test.mocks.dart';

void main() {
  late AuthService authService;
  late MockDio mockDio;

  setUp(() {
    mockDio = MockDio();
    authService = AuthService(dio: mockDio);
  });

  group('登录流程', () {
    test('登录成功返回 Token 对', () async {
      when(mockDio.post(
        '/session/token',
        data: anyNamed('data'),
      )).thenAnswer((_) async => Response(
        requestOptions: RequestOptions(path: '/session/token'),
        data: {
          'code': 0,
          'data': {
            'user': {'id': 'abc123', 'email': 'test@test.com'},
            'token': {
              'access_token': 'access-jwt',
              'refresh_token': 'refresh-jwt',
              'access_expires': '2026-10-28T12:00:00Z',
              'refresh_expires': '2026-11-04T12:00:00Z',
            },
          },
        },
      ));

      final result = await authService.login('test@test.com', 'password');
      expect(result.accessToken, 'access-jwt');
      expect(result.refreshToken, 'refresh-jwt');
    });

    test('密码错误抛出 CloudreveException(40080)', () async {
      when(mockDio.post(
        '/session/token',
        data: anyNamed('data'),
      )).thenAnswer((_) async => Response(
        requestOptions: RequestOptions(path: '/session/token'),
        data: {'code': 40080, 'msg': 'Incorrect password or email'},
      ));

      expect(
        () => authService.login('test@test.com', 'wrong'),
        throwsA(isA<CloudreveException>()
            .having((e) => e.code, 'code', 40080)),
      );
    });
  });
}
```

### 5.3 Widget 测试示例

```dart
// test/widget/auth/login_page_test.dart

void main() {
  testWidgets('登录页面显示邮箱和密码输入框', (WidgetTester tester) async {
    await tester.pumpWidget(
      GetMaterialApp(home: LoginPage()),
    );

    expect(find.byType(TextField), findsNWidgets(2));
    expect(find.text('邮箱'), findsOneWidget);
    expect(find.text('密码'), findsOneWidget);
    expect(find.text('登录'), findsOneWidget);
  });
}
```

### 5.4 覆盖率要求

| 模块 | 最低覆盖率 | 目标 |
|------|-----------|------|
| `core/network/*` | 80% | 100% |
| `services/*` | 70% | 90% |
| `core/hashid/*` | 90% | 100% |
| `core/utils/*` | 70% | 90% |
| Controller | 60% | 80% |
| Widget | 40% | 70% |

### 5.5 CI 质量门禁（.github/workflows/test.yml）

```yaml
name: Test
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.x'
      - run: flutter pub get
      - run: dart format --check lib test
      - run: flutter analyze
      - run: flutter test --coverage
      - run: genhtml coverage/lcov.info -o coverage/html
      - run: |
          # 覆盖率检查（低于 60% 失败）
          total=$(lcov --summary coverage/lcov.info | tail -1 | awk '{print $NF}' | sed 's/%//')
          echo "Total coverage: $total%"
          [ "$total" -ge 60 ] || exit 1
```

---

## 6. 构建打包流程

### 6.1 多环境构建方案

采用 **Flavor + 多 Entrypoint** 方案区分 开发/测试/生产。

#### 6.1.1 多 main 入口

```
lib/
├── main.dart                    # 默认入口 → 生产
├── main_development.dart        # 开发环境
├── main_staging.dart            # 测试环境
└── main_production.dart         # 生产环境（可以不写，默认 main.dart）
```

```dart
// lib/main_development.dart
void main() {
  // 强制使用开发环境配置
  EnvConfig.set(
    baseUrl: 'https://dev-cloudreve.example.com',
    apiPrefix: '/api/v4',
    flavor: Flavor.development,
    enableLogging: true,
    enableAnalytics: false,
  );
  runApp(const CloudreveApp());
}

// lib/main_staging.dart
void main() {
  EnvConfig.set(
    baseUrl: 'https://staging-cloudreve.example.com',
    apiPrefix: '/api/v4',
    flavor: Flavor.staging,
    enableLogging: true,
    enableAnalytics: false,
  );
  runApp(const CloudreveApp());
}

// lib/main.dart（生产）
void main() {
  EnvConfig.set(
    baseUrl: 'https://cloudreve.example.com',
    apiPrefix: '/api/v4',
    flavor: Flavor.production,
    enableLogging: false,
    enableAnalytics: true,
  );
  runApp(const CloudreveApp());
}
```

#### 6.1.2 EnvConfig 实现

```dart
// lib/core/config/env_config.dart

enum Flavor { development, staging, production }

class EnvConfig {
  static Flavor flavor = Flavor.production;
  static String baseUrl = 'https://cloudreve.example.com';
  static String apiPrefix = '/api/v4';
  static bool enableLogging = false;
  static bool enableAnalytics = true;

  static void set({
    required String baseUrl,
    required String apiPrefix,
    required Flavor flavor,
    required bool enableLogging,
    required bool enableAnalytics,
  }) {
    EnvConfig.baseUrl = baseUrl;
    EnvConfig.apiPrefix = apiPrefix;
    EnvConfig.flavor = flavor;
    EnvConfig.enableLogging = enableLogging;
    EnvConfig.enableAnalytics = enableAnalytics;
  }

  static bool get isDevelopment => flavor == Flavor.development;
  static bool get isProduction => flavor == Flavor.production;
  static String get apiBase => '$baseUrl$apiPrefix';
}
```

### 6.2 Android 打包

#### 6.2.1 签名配置（一次性）

```bash
# 生成签名密钥（在项目根目录执行）
keytool -genkeypair -v \
  -keystore android/app/upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias upload

# 记下密钥密码、别名，妥善保管
```

创建 `android/key.properties`：
```properties
storePassword=your-store-password
keyPassword=your-key-password
keyAlias=upload
storeFile=upload-keystore.jks
```

**⚠️ 安全提醒**：
- `upload-keystore.jks` 和 `key.properties` 均加入 `.gitignore`
- 密钥文件备份到安全位置（密码管理器）
- Play Console 密钥丢失无法恢复

修改 `android/app/build.gradle`（关键部分）：
```gradle
android {
    defaultConfig {
        applicationId "com.cloudreve.client"
        minSdkVersion 23
        targetSdkVersion 34
        versionCode 1
        versionName "1.0.0"
    }

    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    flavorDimensions "env"
    productFlavors {
        development { dimension "env"; applicationIdSuffix ".dev"; versionNameSuffix "-dev" }
        staging     { dimension "env"; applicationIdSuffix ".staging"; versionNameSuffix "-staging" }
        production  { dimension "env" }
    }
}
```

#### 6.2.2 打包命令

```bash
cd cloudreve_client

# ===== 开发环境 =====
# APK（安装到真机测试）
flutter build apk --flavor development -t lib/main_development.dart --debug
# 产物：build/app/outputs/flutter-apk/app-development-debug.apk

# App Bundle（推荐，上传 Play Store）
flutter build appbundle --flavor development -t lib/main_development.dart --release

# ===== 测试环境 =====
flutter build appbundle --flavor staging -t lib/main_staging.dart --release
# 产物：build/app/outputs/bundle/staging-release/app-staging-release.aab

# ===== 生产环境 =====
flutter build appbundle --flavor production -t lib/main.dart --release
# 产物：build/app/outputs/bundle/production-release/app-production-release.aab

# 同时构建所有 flavor（可选）
flutter build apk --flavor development,staging,production
```

#### 6.2.3 ProGuard 配置

```proguard
# android/app/proguard-rules.pro

# Cloudreve Flutter Client ProGuard
-keep class com.cloudreve.client.** { *; }

# Dio
-keepattributes Signature
-keepattributes *Annotation*
-keep class dio.** { *; }

# GetX
-keep class get.** { *; }
-keepmembersclass ** { @GetInject *; }

# Hive
-keep class *.g.dart { *; }
-keepclassmembers class ** extends TypeAdapter { *; }

# Cloudreve 自定义 Model（保持字段）
-keep class model.** { *; }

# 保持 Map<String, dynamic> 序列化不混淆字段名
-keepclassmembers class ** { @com.google.gson.annotations.SerializedName <fields>; }
```

### 6.3 iOS 打包

#### 6.3.1 前置条件

```bash
# macOS + Xcode 16+
xcode-select -p            # 确认 Xcode 路径
pod --version              # CocoaPods 1.15+
flutter doctor -v          # 确认 iOS toolchain

# 配置 CocoaPods 国内镜像（可选）
pod repo update
```

#### 6.3.2 签名配置（Xcode 中操作）

```
1. 用 Xcode 打开 cloudreve_client/ios/Runner.xcworkspace
2. Target → Runner → Signing & Capabilities
3. Team: 选择你的 Apple Developer Team
4. Bundle Identifier: com.cloudreve.client（与 Android 一致或不同）
5. 勾选 Automatically manage signing
6. Info → Bundle versions: 与 pubspec.yaml 对齐
```

#### 6.3.3 打包命令

```bash
cd cloudreve_client

# 先更新 Pod 依赖（如果 ios/Podfile.lock 过期）
cd ios && pod install && cd ..

# ===== 开发环境（模拟器）=====
flutter build ios --flavor development -t lib/main_development.dart --debug --simulator

# ===== 测试/生产环境（真机 Archive）=====
flutter build ipa --flavor production -t lib/main.dart --release \
  --export-method=ad-hoc \
  --export-options-plist=ios/ExportOptions.plist

# ===== App Store 分发 =====
flutter build ipa --flavor production -t lib/main.dart --release \
  --export-method=app-store

# 产物位置
# IPA: build/ios/ipa/Cloudreve Client.ipa
# xcarchive: build/ios/archive/Runner.xcarchive
```

#### 6.3.4 ExportOptions.plist（可选）

```xml
<!-- ios/ExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>signingStyle</key>
    <string>automatic</string>
    <key>stripSwiftSymbols</key>
    <true/>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.cloudreve.client</key>
        <string>iOS App Store Provisioning Profile</string>
    </dict>
</dict>
</plist>
```

#### 6.3.5 iOS Info.plist（关键配置）

```xml
<!-- ios/Runner/Info.plist 关键项 -->
<key>CFBundleURLSchemes</key>
<array>
    <string>cloudreve</string>       <!-- 深链接 -->
</array>

<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>                         <!-- 生产不允许 HTTP -->
    <key>NSExceptionDomains</key>    <!-- 仅开发时允许 HTTP -->
    <dict>
        <key>dev-cloudreve.example.com</key>
        <dict>
            <key>NSIncludesSubdomains</key><true/>
            <key>NSAllowsArbitraryLoads</key><true/>
        </dict>
    </dict>
</dict>

<key>LSApplicationQueriesSchemes</key>
<array>
    <string>cloudreve</string>
    <string>cloudreve-dev</string>
</array>
```

### 6.4 版本号管理

```yaml
# pubspec.yaml —— 单一真实来源
# version: 显示版本+构建号，自动同步到 AndroidManifest.xml 和 Info.plist
version: 1.0.0+1     # [显示版本]+[构建号]
#   ↑ 每次发布改这里
#   ↑ 每次构建 +1
```

升级命令：
```bash
# 同时更新 Android 和 iOS 原生版本配置
flutter pub run flutter_launcher_icons:main   # 可选：生成新图标
flutter clean
flutter pub get
```

### 6.5 Web 构建（可选）

```bash
# 生产 Web 构建
flutter build web --release --web-renderer html

# 开发 Web 调试
flutter run -d chrome --web-hostname 0.0.0.0 --web-port 8080
```

### 6.6 发布渠道对比

| 渠道 | 命令 | 产物 | 分发方式 |
|------|------|------|----------|
| Android Internal Test | `flutter build appbundle --release` | `.aab` | Play Console → 内部测试轨道 |
| Android Alpha/Beta | 同上 | `.aab` | Play Console → Alpha/Beta 轨道 |
| Android Production | 同上 | `.aab` | Play Console → 生产轨道 |
| Android 侧载 | `flutter build apk --release` | `.apk` | 邮件/网盘/官网下载 |
| iOS TestFlight | `flutter build ipa --release --export-method=app-store` | `.ipa` | Xcode Organizer 上传 → TestFlight |
| iOS App Store | 同上 | `.ipa` | 审核通过后发布 |
| iOS Ad Hoc | `flutter build ipa --release --export-method=ad-hoc` | `.ipa` | 注册 UDID 的设备侧载 |
| iOS 企业签名 | `flutter build ipa --export-method=enterprise` | `.ipa` | MDM 或企业签名分发 |

---

## 7. 完整命令清单

### 7.1 从零开始（新开发人员入职 SOP）

```bash
# ========== Step 1: 准备 Flutter 环境 ==========

# Windows
winget install FlutterSDK
[Environment]::SetEnvironmentVariable(
  "Path",
  $env:Path + ";$env:LOCALAPPDATA\Pub\Cache\bin",
  "User"
)

# macOS
brew install --cask flutter

# 通用：验证
flutter --version           # 需 >= 3.27.0
flutter doctor -v            # 解决所有 ✗ 标记

# 首次运行会下载 Dart SDK 和平台工具
flutter doctor -v            # 再次检查直到所有项目 ✅

# ========== Step 2: 获取项目 ==========

# 获取仓库地址（Cloudreve 客户端仓库，需预先创建）
git clone https://github.com/your-org/cloudreve_flutter_client.git
cd cloudreve_flutter_client

# ========== Step 3: 安装依赖 ==========

flutter pub get                                 # 拉取所有依赖
dart run build_runner build --delete-conflicting-outputs  # 生成代码（Hive/Mock 等）

# iOS 必须先跑 pod install（第一次或 Podfile.lock 变更后）
cd ios && pod install && cd ..

# ========== Step 4: 配置签名 ==========

# Android 签名（已有则跳过）
if (-not (Test-Path android\app\upload-keystore.jks)) {
  keytool -genkeypair -v `
    -keystore android/app/upload-keystore.jks `
    -keyalg RSA -keysize 2048 -validity 10000 `
    -alias upload
}
# 手动创建 android/key.properties（按实际密码填写）

# iOS 签名（在 Xcode 里完成，或命令行配置）
open ios/Runner.xcworkspace     # Xcode 打开
# → Runner Target → Signing & Capabilities → 选 Team + Bundle ID

# ========== Step 5: 本地运行 ==========

# Chrome 先跑起来验证没大问题
flutter run -d chrome -t lib/main_development.dart

# Android Emulator（先在 Android Studio AVD Manager 创建一个）
flutter run -d emulator-5554 -t lib/main_development.dart

# iOS Simulator（需要 macOS + Xcode）
flutter run -d 'iPhone 16' -t lib/main_development.dart

# ========== Step 6: 质量检查 ==========

dart format --set-exit-if-changed lib test   # 格式化
flutter analyze                               # 静态检查
flutter test --coverage                       # 运行全部测试+覆盖率

# ========== Step 7: 生产构建 ==========

# Android
flutter build appbundle --flavor production -t lib/main.dart --release
# 产物: build/app/outputs/bundle/production-release/app-production-release.aab
# → Play Console 上传

# iOS（需要 macOS）
cd ios && pod install && cd ..
flutter build ipa --flavor production -t lib/main.dart --release \
  --export-method=app-store
# 产物: build/ios/ipa/*.ipa
# → Xcode Organizer → 上传 App Store Connect

# ========== Step 8: 提交代码 ==========

git add .
git commit -m "feat: 实现 Cloudreve 文件浏览核心功能"
git push origin feature/explorer
# 发 PR → CI 自动跑 lint/analyze/test
# Review → Merge main
```

### 7.2 日常开发命令速查表

```bash
# ===== 创建/更新 =====
flutter create .                              # 生成项目（已存在则 merge）
flutter pub get                               # 拉取依赖
flutter pub upgrade --major-versions          # 升级所有依赖
dart run build_runner build --delete-conflicting-outputs  # 生成代码
dart run build_runner watch                   # 监听自动生成

# ===== 开发运行 =====
flutter run -d chrome                         # Chrome 运行
flutter run -d macos                         # macOS 运行
flutter run -d emulator-5554                  # Android 模拟器
flutter run -d 'iPhone 16'                    # iOS 模拟器
flutter run --flavor development -t lib/main_development.dart  # 指定 flavor
flutter run -v                                # verbose 输出（调试 Dio 有用）
flutter attach                                # 附加到已有实例

# ===== 代码质量 =====
dart format lib test                          # 格式化
dart format --check lib test                  # 检查是否需格式化（CI 用）
dart fix --apply                              # 自动修复 lint 建议
flutter analyze                               # 静态检查
flutter analyze --fatal-infos                 # Info 也算错

# ===== 测试 =====
flutter test                                  # 全部测试
flutter test test/unit/services/             # 指定目录
flutter test --name "登录"                    # 按名字过滤
flutter test --coverage                       # 覆盖率
flutter test --coverage && genhtml coverage/lcov.info -o coverage/html
flutter run integration_test/explorer_test.dart -d emulator-5554

# ===== 构建 =====
flutter build apk --debug                     # Android Debug APK
flutter build apk --release                   # Android Release APK
flutter build appbundle --release             # Android AAB（推荐）
flutter build ios --release --simulator       # iOS Release（模拟器）
flutter build ipa --release --export-method=app-store  # iOS IPA
flutter build web --release                   # Web Release
flutter build web --release --web-renderer html
flutter build apk --target-platform android-arm,android-arm64,android-x64  # 多架构

# ===== 清理 =====
flutter clean                                 # 删除 build/ .dart_tool/ .flutter-plugins
flutter pub cache repair                      # 修复 pub 缓存
flutter doctor --android-licenses             # 同意 Android SDK 协议

# ===== SDK / 版本 =====
flutter --version                             # Flutter 版本
flutter --help                                # 所有命令
dart --version                                # Dart 版本
which flutter                                 # Flutter 路径

# ===== 平台工具 =====
open -a Simulator                             # 打开 iOS 模拟器
adb devices                                   # Android 设备列表
flutter devices                               # Flutter 设备列表（更准确）
flutter emulators                             # Flutter 模拟器列表
flutter emulators --launch <emulator_id>      # 启动模拟器
```

### 7.3 git workflow（推荐）

```bash
# ===== 分支模型 =====
# main     —— 生产代码，保护分支，PR Merge 才进
# develop  —— 开发集成分支（可选，小团队可省略）
# feature/* —— 功能分支，从 main 切
# hotfix/* —— 紧急修复，从 main 切，修完合 main + develop
# release/* —— 发布准备

# ===== 日常流程 =====
git checkout main
git pull origin main
git checkout -b feature/auth-login             # 功能分支
# ... 写代码 ...
flutter analyze && flutter test --coverage     # 本地验证
git add .
git commit -m "feat(auth): 实现 Cloudreve JWT 登录"
git push -u origin feature/auth-login
# 发 PR → Code Review → CI 通过 → Merge main

# ===== 发版流程 =====
git checkout main
git pull origin main
# 修改 pubspec.yaml 版本号
# 修改 CHANGELOG.md
flutter test --coverage && flutter analyze
# 构建 + 签名 + 上传
flutter build appbundle --flavor production -t lib/main.dart --release
flutter build ipa --flavor production -t lib/main.dart --release --export-method=app-store
git tag v1.1.0 && git push origin v1.1.0
# GitHub Release 附上构建产物
```

### 7.4 常见问题排查

| 问题 | 解决方案 |
|------|----------|
| `flutter run` 找不到设备 | `flutter devices` → 如果为空，Android Studio 启动模拟器 / iOS Simulator 启动 |
| `pod install` 卡住 | 配置 CocoaPods 国内镜像：`pod repo add-cdn https://cdn.cocoapods.org/` |
| iOS 编译 `ld: symbol not found` | `cd ios && pod deintegrate && pod install` 重装 Pod |
| Android `Execution failed for task ':app:signReleaseBundle'` | 检查 `android/key.properties` 密码是否正确 |
| Dio 401 死循环刷新 | AuthInterceptor 里检查 refresh 接口请求是否带了正确的 Content-Type，且 refresh 路径不走 AuthInterceptor（跳过 baseUrl 拼接） |
| 分片上传 `Content-Length` 错误 | 确保 chunk 最后一片的 Length = 文件总大小 - index × chunkSize |
| Cloudreve 返回 40013 | 同上，分片大小和服务端期望值不匹配 |
| 热重载后状态丢失 | 这是正常的 —— 状态持久化请走 Hive/shared_preferences |
| Release 包网络请求全部失败 | 检查 AndroidManifest 权限 + iOS ATS 配置；检查 EnvConfig.baseUrl 是否还指向 dev |
| Release 包体积太大 | 开启 `minifyEnabled true` + ProGuard + 检查 assets 是否都需要 |
| `Could not resolve symbol 'GetxController'` | `flutter pub get` 后重启 VS Code |

### 7.5 Flutter 升级策略

```bash
# 稳定版升级（推荐）
flutter upgrade                                # 升到最新 stable

# 指定版本
flutter version 3.27.0                         # 锁定版本
flutter version stable                         # 切回 stable channel

# channel 切换
flutter channel stable
flutter channel beta
flutter channel master                         # 最新但不稳定

# 升级后
flutter clean && flutter pub get && flutter test
```

**升级节奏建议**：Flutter stable 约每 3 个月一个 minor 版本。团队统一升级，避免有人还在 3.10 有人 3.27。

---

## 附录 A：Cloudreve API 快速索引

| Flutter Service | Cloudreve API | 鉴权 |
|-----------------|---------------|------|
| `AuthService.login()` | `POST /api/v4/session/token` | 无 |
| `AuthService.refresh()` | `POST /api/v4/session/token/refresh` | refresh_token |
| `ExplorerService.listDirectory()` | `GET /api/v4/file` | Bearer |
| `ExplorerService.getFileInfo()` | `GET /api/v4/file/info` | Bearer |
| `ExplorerService.getDownloadUrl()` | `POST /api/v4/file/url` | Bearer |
| `ExplorerService.delete()` | `DELETE /api/v4/file` | Bearer |
| `UploadService.createSession()` | `PUT /api/v4/file/upload` | Bearer |
| `UploadService.uploadChunk()` | `POST /api/v4/file/upload/:id/:index` | Bearer |
| `UploadService.cancelSession()` | `DELETE /api/v4/file/upload` | Bearer |
| `ShareService.create()` | `PUT /api/v4/share` | Bearer |
| `ShareService.list()` | `GET /api/v4/share` | Bearer |
| `ShareService.delete()` | `DELETE /api/v4/share/:id` | Bearer |
| `WorkflowService.archive()` | `POST /api/v4/workflow/archive` | Bearer |
| `WorkflowService.download()` | `POST /api/v4/workflow/download` | Bearer |

> 完整端点说明见 [Cloudreve_API_Reference.md](./Cloudreve_API_Reference.md)

## 附录 B：推荐开发工具清单

| 工具 | 用途 | 必选 |
|------|------|------|
| **VS Code** + Flutter 插件 | IDE | ✅ |
| **Android Studio** | Android 模拟器 + SDK Manager | ✅ |
| **Xcode** | iOS Simulator + Archive + 签名（仅 macOS） | ✅ (iOS 必需) |
| **DevTools** | Widget Inspector / Network / Performance | ✅ |
| **Postman / Apifox** | API 调试（Cloudreve 接口开发阶段） | ✅ |
| **Git** + GUI（GitKraken / Fork） | 版本控制 | ✅ |
| **Figma** / **MasterGo** | UI 设计稿（可跳过，Cloudreve 有官方 Web 版 UI 参考） | 推荐 |
| **Charles** / **Proxyman** | HTTPS 抓包（调试真实 API） | 推荐 |
| **Firebase Crashlytics** | Release 崩溃上报 | 推荐 |
| **Sentry** | Release 错误追踪（可选，Firebase 够用） | 可选 |

---

*文档版本：v1.0 · 生成日期：2026-10-27*
*基于 Cloudreve v4 API 和 Flutter 3.27.x 编写*
*配套 API 文档：[Cloudreve_API_Reference.md](./Cloudreve_API_Reference.md)*
