# Go 1.22+ & Fiber 后端开发指南 (Agent 规则版)

本指南旨在为 AI Agent 提供最新的 Go 1.22+ 语法特性、Fiber 框架最佳实践及高性能后端架构规范。

---

## 1. Go 1.22+ 核心特性 (Core Language)

### 1.1 循环变量作用域修复
在 Go 1.22 之前，在循环中开启 goroutine 引用循环变量是经典陷阱。Go 1.22 起，**不再需要**手动 `v := v`。
```go
// Go 1.22 推荐做法
for _, v := range values {
    go func() {
        fmt.Println(v) // 此时 v 是安全的，每个迭代都有独立副本
    }()
}
```

### 1.2 标准库 HTTP 路由增强 (Context)
虽然我们使用 Fiber，但了解标准库 `net/http` 现在支持 `GET /path/{id}` 有助于理解现代 Go 路由风格。

---

## 2. Fiber 框架最佳实践 (Fiber v2/v3)

### 2.1 零拷贝与性能
Fiber 基于 `fasthttp`，极其追求性能。**注意**：`c.Params()`, `c.Query()` 等返回的字节切片在请求结束后会被回收。
- **规则**：如果需要在协程或异步任务中使用请求数据，**必须**使用 `c.Copy()` 或将其转换为字符串。

### 2.2 项目结构推荐 (Clean Architecture)
```
.
├── cmd/                # 入口
│   └── server/main.go
├── internal/
│   ├── api/            # 路由定义与 Handler
│   ├── service/        # 业务逻辑
│   ├── repository/     # 数据库操作
│   ├── model/          # 结构体定义
│   └── middleware/     # 中间件
├── pkg/                # 可导出的公共包
├── go.mod
└── .env
```

### 2.3 规范化 Handler 模板
```go
func GetUserHandler(c *fiber.Ctx) error {
    id, err := c.ParamsInt("id")
    if err != nil {
        return fiber.NewError(fiber.StatusBadRequest, "无效的 ID")
    }

    user, err := service.GetUserByID(id)
    if err != nil {
        return err // 由全局错误处理器处理
    }

    return c.Status(fiber.StatusOK).JSON(fiber.Map{
        "code": 200,
        "data": user,
    })
}
```

### 2.4 全局错误处理
**禁止**在每个 Handler 里写大量的 `if err != nil { log...; return c.Status... }`。
- **推荐**：在 Fiber 配置中定义一个 `ErrorHandler`，Handler 统一 `return err`。

---

## 3. 依赖注入与配置

1.  **配置加载**：使用 `viper` 或 `caarlos0/env` 加载环境变量。
2.  **依赖注入**：对于中大型项目，优先使用 `uber-go/fx` 或手动构造注入，避免全局变量。

---

## 4. 常见陷阱与避坑 (Common Pitfalls)

1.  **不可重用的 ctx**：严禁将 `*fiber.Ctx` 传递给子协程。
    - ✅ 做法：在进入协程前提取所有需要的数据。
2.  **忽略返回值**：Fiber 的 Handler 必须返回 `error`，即 `return c.SendString(...)`。
3.  **大对象传递**：优先传递指针以减少内存拷贝。

---

## 5. Agent 指令 (Agent Instructions)

当你在开发 Go 后端项目时，请遵循：
1. **类型安全**：严格使用结构体进行请求绑定 (`c.BodyParser`)。
2. **中间件优先**：鉴权、日志、限流应作为中间件，不要耦合在业务逻辑中。
3. **接口导向**：Repository 和 Service 层应定义接口，方便测试和解耦。
4. **Context 传播**：确保将 `c.Context()` 传递给下游的数据库或 API 调用。
