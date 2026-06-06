# Snippetbox 项目结构归纳

> 本项目出自 Alex Edwards《Let's Go》一书。核心设计是**关注点分离**：
> `cmd/web`（处理 HTTP）、`internal/models`（处理数据）、`ui`（处理展示）三层彻底解耦。

## 一、目录树总览

```
snippetbox/
├── cmd/web/              # 应用程序入口（main 包）—— Web 服务器全部逻辑
│   ├── main.go           # 程序入口、配置、依赖注入、启动 HTTPS 服务器
│   ├── routes.go         # 路由表 + 中间件链组装
│   ├── handlers.go       # 各个 HTTP 处理器(handler) + 表单结构体
│   ├── helpers.go        # 辅助方法(渲染、错误、解码表单等)
│   ├── middleware.go     # 中间件(安全头、日志、panic 恢复、认证、CSRF)
│   ├── context.go        # 自定义 context key
│   ├── templates.go      # 模板缓存 + 传给模板的数据结构
│   └── *_test.go         # 单元/集成测试
│
├── internal/             # 私有包(只能被本模块导入)
│   ├── models/           # 数据层(数据库模型)
│   │   ├── snippets.go    # Snippet 模型 + 接口
│   │   ├── users.go       # User 模型 + 接口
│   │   ├── errors.go      # 自定义错误变量
│   │   ├── mock/          # 测试用的 mock 模型
│   │   └── testdata/      # 测试用 SQL(建表/清表)
│   ├── validator/        # 通用表单校验工具
│   │   └── validator.go
│   └── assert/           # 测试断言辅助
│       └── assert.go
│
├── ui/                   # 前端资源(通过 embed 嵌入二进制)
│   ├── efs.go            # //go:embed 声明，把 html/static 打包进程序
│   ├── html/
│   │   ├── base.tmpl      # 基础布局模板
│   │   ├── partials/      # 可复用片段(nav.tmpl)
│   │   └── pages/         # 各页面(home/view/create/login/signup...)
│   └── static/           # CSS / JS / 图片
│
├── tls/                  # 自签名 TLS 证书(cert.pem / key.pem)
├── go.mod / go.sum       # 依赖管理
└── .gitignore
```

---

## 二、关键结构体 & 它们持有的数据

### 1. 中枢：`application`（`main.go`）

所有依赖通过这个结构体注入，再把方法挂在它上面共享。

| 字段 | 类型 | 作用 |
|------|------|------|
| `debug` | `bool` | 是否调试模式（决定错误页是否显示堆栈） |
| `logger` | `*slog.Logger` | 结构化日志 |
| `snippets` | `models.SnippetModelInterface` | 代码片段数据访问（用**接口**，便于 mock 测试） |
| `users` | `models.UserModelInterface` | 用户数据访问 |
| `templateCache` | `map[string]*template.Template` | 预编译模板缓存 |
| `formDecoder` | `*form.Decoder` | 把 HTML 表单解码进结构体 |
| `sessionManager` | `*scs.SessionManager` | Session 管理（存 MySQL，12 小时） |

### 2. 数据层模型（`internal/models`）

**`Snippet`**（对应 MySQL `snippets` 表）

```
ID int | Title string | Content string | Created time.Time | Expires time.Time
```

- **`SnippetModel{ DB *sql.DB }`** → 方法：`Insert / Get / Latest`
- **`SnippetModelInterface`** → 上述三个方法的接口抽象

**`User`**（对应 MySQL `users` 表）

```
ID int | Name string | Email string | HashedPassword []byte | Created time.Time
```

- **`UserModel{ DB *sql.DB }`** → 方法：`Insert / Authenticate / Exists / Get / PasswordUpdate`（密码用 bcrypt 哈希）
- **`UserModelInterface`** → 接口抽象

> 注意：模型用「接口 + 具体实现」双套设计——生产用 `SnippetModel`，测试用 `mock` 包里的假实现。

### 3. 模板数据：`templateData`（`templates.go`）

每次渲染页面时传给模板的「数据袋」：

```
CurrentYear int | Snippet models.Snippet | Snippets []models.Snippet
Form any | Flash string | IsAuthenticated bool | CSRFToken string | User models.User
```

### 4. 表单结构体（`handlers.go`）

每个都**内嵌** `validator.Validator`，用 `form:"xxx"` 标签做字段映射：

| 结构体 | 字段 |
|--------|------|
| `snippetCreateForm` | Title, Content, Expires + Validator |
| `userSignupForm` | Name, Email, Password + Validator |
| `userLoginForm` | Email, Password + Validator |
| `accountPasswordUpdateForm` | CurrentPassword, NewPassword, NewPasswordConfirmation + Validator |

### 5. 校验器：`validator.Validator`（`internal/validator`）

```
NonFieldErrors []string           // 整体性错误(如"邮箱或密码错误")
FieldErrors    map[string]string  // 字段级错误(字段名→错误信息)
```

配套独立函数：`NotBlank / MaxChars / MinChars / MaxBytes / PermittedValue / Matches`。

---

## 三、一次请求的流转图

```
                          浏览器请求 (HTTPS)
                                │
                                ▼
┌───────────────────────────────────────────────────────────┐
│                    标准中间件链 (standard)                    │
│   recoverPanic → logRequest → commonHeaders(安全头)          │
└───────────────────────────────────────────────────────────┘
                                │
                                ▼
                        ServeMux 路由匹配 (routes.go)
                                │
            ┌───────────────────┴────────────────────┐
            ▼                                         ▼
   ┌──────────────────┐                   /static/*  →  embed.FS 静态文件
   │  dynamic 中间件链  │                              /ping → "OK"
   │ session→CSRF→     │
   │ authenticate      │
   └──────────────────┘
            │
            ├──────────────► 公开路由: home / about / view / signup / login
            │
            ▼
   ┌──────────────────────┐
   │ protected 中间件链     │  (dynamic + requireAuthentication)
   │ 未登录→重定向 /login   │
   └──────────────────────┘
            │
            ▼
            └─► 受保护路由: snippet/create · account/view · password/update · logout
                                │
                                ▼
┌───────────────────────────────────────────────────────────┐
│                   Handler (handlers.go)                     │
│  1. decodePostForm  →  填充 Form 结构体                       │
│  2. validator       →  校验，失败则回显表单(422)              │
│  3. app.snippets/users  →  调数据层(SQL/bcrypt)              │
│  4. sessionManager.Put  →  写 flash 消息                     │
│  5. app.render      →  从 templateCache 取模板 + templateData │
└───────────────────────────────────────────────────────────┘
            │                              │
            ▼                              ▼
   internal/models (数据库)         ui/html (embed 模板)
   MySQL: snippets / users          base + partials + pages
```

---

## 四、依赖关系一句话总结

```
cmd/web  ──依赖──►  internal/models   (数据)
   │                internal/validator (校验)
   │                ui                 (模板/静态资源)
   └──注入──►  application 结构体 ──持有──► 所有共享依赖
```

三个外部关键库：**scs**（session）、**alice**（中间件链）、**nosurf**（CSRF 防护），
加 **go-playground/form**（表单解码）、**bcrypt**（密码哈希）。
