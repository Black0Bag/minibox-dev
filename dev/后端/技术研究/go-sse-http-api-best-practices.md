# Go SSE 流式响应 & HTTP API 框架最佳实践研究

> 基于互联网搜索（pkg.go.dev 官方文档、go-chi/chi v5 文档、go-chi/cors 文档）及社区最佳实践整理。

---

## 1. Go 原生 SSE 实现（不依赖第三方库）

### 1.1 SSE 协议基础

SSE（Server-Sent Events）是基于 HTTP 的单向流式协议。服务端通过 `Content-Type: text/event-stream` 响应头声明 SSE 流，然后持续写入符合格式的文本事件。

**SSE 事件格式：**
```
event: message
data: {"content":"hello"}

event: done
data: [DONE]

```

关键规则：
- 每个字段以 `field: value\n` 格式写入
- 事件之间用 `\n\n`（空行）分隔
- `data:` 字段可以多行，每行以 `data:` 开头
- `event:` 字段指定事件类型（客户端可按类型监听）
- `id:` 字段设置事件 ID（用于断线重连）
- `retry:` 字段设置重连间隔（毫秒）
- 以 `:` 开头的行是注释（常用于心跳保活）

### 1.2 核心：http.Flusher 接口

来自 Go 标准库 `net/http`（pkg.go.dev 确认）：

```go
// Flush sends any buffered data to the client.
type Flusher interface {
    Flush()
}
```

SSE 的关键在于：**写入 ResponseWriter 后必须调用 Flush()**，否则数据会被缓冲在 Go 的 `bufio.Writer` 中，客户端收不到实时数据。

### 1.3 原生 SSE Handler 完整实现

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// SSE handler — 纯 net/http，不依赖任何第三方库
func sseHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 检查是否支持 Flusher
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming unsupported", http.StatusInternalServerError)
        return
    }

    // 2. 设置 SSE 必需的响应头
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    // 可选：禁用代理缓冲
    w.Header().Set("X-Accel-Buffering", "no")

    // 3. 写入初始响应头并 Flush
    //    （第一次 Flush 确保头部立即发送给客户端）
    flusher.Flush()

    // 4. 监听客户端断开
    ctx := r.Context()

    // 5. 事件循环
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for i := 0; ; i++ {
        select {
        case <-ctx.Done():
            // 客户端断开连接
            return
        case <-ticker.C:
            // 写入 SSE 格式数据
            fmt.Fprintf(w, "data: {\"count\": %d, \"time\": \"%s\"}\n\n", i, time.Now().Format(time.RFC3339))
            // 立即刷新缓冲区，推送给客户端
            flusher.Flush()
        }
    }
}

func main() {
    http.HandleFunc("/events", sseHandler)
    http.ListenAndServe(":8080", nil)
}
```

### 1.4 封装 SSE Writer 工具类型

```go
// SSEWriter 封装 SSE 写入逻辑，避免重复代码
type SSEWriter struct {
    w       http.ResponseWriter
    flusher http.Flusher
}

func NewSSEWriter(w http.ResponseWriter) (*SSEWriter, error) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        return nil, fmt.Errorf("streaming unsupported")
    }
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no")
    flusher.Flush()
    return &SSEWriter{w: w, flusher: flusher}, nil
}

// WriteEvent 写入一个完整 SSE 事件
func (s *SSEWriter) WriteEvent(eventType string, data string) error {
    if eventType != "" {
        fmt.Fprintf(s.w, "event: %s\n", eventType)
    }
    fmt.Fprintf(s.w, "data: %s\n\n", data)
    s.flusher.Flush()
    return nil
}

// WriteData 写入仅含 data 的事件
func (s *SSEWriter) WriteData(data string) error {
    return s.WriteEvent("", data)
}

// WriteComment 写入注释行（用于心跳保活）
func (s *SSEWriter) WriteComment(comment string) {
    fmt.Fprintf(s.w, ": %s\n\n", comment)
    s.flusher.Flush()
}
```

### 1.5 心跳保活（Keep-Alive）

SSE 连接可能被代理/负载均衡器因超时而断开。最佳实践是定期发送注释行：

```go
// 在事件循环中加入心跳
heartbeat := time.NewTicker(15 * time.Second)
defer heartbeat.Stop()

select {
case <-ctx.Done():
    return
case <-heartbeat.C:
    // SSE 注释行，客户端忽略，但保持连接活跃
    fmt.Fprintf(w, ": heartbeat\n\n")
    flusher.Flush()
case evt := <-eventChan:
    // 处理实际事件
    sseWriter.WriteEvent(evt.Type, evt.Data)
}
```

### 1.6 客户端断开检测

```go
// r.Context().Done() 在客户端断开时会被触发
ctx := r.Context()
select {
case <-ctx.Done():
    // 客户端关闭了连接
    log.Println("Client disconnected")
    return
default:
    // 继续处理
}
```

---

## 2. chi 路由器配合 SSE 的使用方式

### 2.1 chi 路由器基础（来自 pkg.go.dev 官方文档）

chi 是轻量级、惯用、可组合的 Go HTTP 路由器，100% 兼容 `net/http`。核心特性：
- ~1000 LOC，零外部依赖
- 基于 context 包的请求级值传递、取消和超时
- 支持中间件链、内联中间件、路由组和子路由挂载
- 支持 URL 参数 `{name}` 和正则匹配 `{name:\\d+}`

**基本使用：**
```go
package main

import (
    "net/http"
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

func main() {
    r := chi.NewRouter()
    r.Use(middleware.Logger)
    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("welcome"))
    })
    http.ListenAndServe(":3000", r)
}
```

### 2.2 chi 中间件链（生产级推荐配置）

来自 chi 官方文档的推荐中间件栈：

```go
r := chi.NewRouter()

// 生产级中间件栈
r.Use(middleware.RequestID)            // 注入请求 ID
r.Use(middleware.ClientIPFromRemoteAddr) // 或 ClientIPFromXFF（根据部署架构选择）
r.Use(middleware.Logger)               // 请求日志
r.Use(middleware.Recoverer)            // panic 恢复
r.Use(middleware.Timeout(60 * time.Second)) // 请求超时

// 注意：SSE 端点需要跳过 Timeout 中间件！
// 因为 SSE 是长连接，Timeout 会在超时后取消 context
```

### 2.3 chi + SSE 集成

**关键问题：SSE 端点不能使用 `middleware.Timeout`**，因为 SSE 是长连接，Timeout 中间件会通过 `ctx.Done()` 取消请求。

**解决方案 1：使用 `r.With()` 为 SSE 路由单独配置中间件**

```go
func main() {
    r := chi.NewRouter()

    // 全局中间件（不含 Timeout）
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)

    // 普通 API 路由（带超时）
    r.Group(func(r chi.Router) {
        r.Use(middleware.Timeout(30 * time.Second))
        r.Get("/api/tasks", listTasks)
        r.Post("/api/tasks", createTask)
    })

    // SSE 流式路由（不带超时，自行管理生命周期）
    r.Group(func(r chi.Router) {
        // 不使用 Timeout 中间件
        r.Get("/events", sseHandler)
        r.Get("/stream/chat", chatStreamHandler)
    })

    http.ListenAndServe(":3000", r)
}
```

**解决方案 2：在 SSE handler 中覆盖超时 context**

```go
func sseHandler(w http.ResponseWriter, r *http.Request) {
    // 创建不带超时的 context（但保留取消能力）
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 监听原始请求的取消
    go func() {
        <-r.Context().Done()
        cancel()
    }()

    // 使用新的 ctx 进行后续操作
    // ...
}
```

### 2.4 chi 路由组与子路由（来自官方文档）

```go
// RESTy 路由示例
r.Route("/articles", func(r chi.Router) {
    r.With(paginate).Get("/", listArticles)
    r.Post("/", createArticle)
    r.Get("/search", searchArticles)

    // 正则 URL 参数
    r.Get("/{articleSlug:[a-z-]+}", getArticleBySlug)

    // 子路由器
    r.Route("/{articleID}", func(r chi.Router) {
        r.Use(ArticleCtx) // 上下文中间件
        r.Get("/", getArticle)
        r.Put("/", updateArticle)
        r.Delete("/", deleteArticle)
    })
})

// 挂载独立子路由器
r.Mount("/admin", adminRouter())
```

### 2.5 chi 中间件编写模式

```go
// 标准 net/http 中间件，与 chi 完全兼容
func MyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 前置处理
        ctx := context.WithValue(r.Context(), "user", "123")
        // 调用下一个 handler
        next.ServeHTTP(w, r.WithContext(ctx))
        // 后置处理
    })
}

// 使用 With() 添加内联中间件（仅对特定路由生效）
r.With(MyMiddleware).Get("/protected", handler)
```

### 2.6 chi URL 参数获取

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "userID") // 从 /users/{userID} 获取
    // 从 context 获取
    ctx := r.Context()
    key := ctx.Value("key").(string)
}
```

---

## 3. Go 中流式 LLM 响应的 SSE 传输最佳实践

### 3.1 架构概览

```
Client (浏览器/fetch)
    ↕ SSE (text/event-stream)
Go HTTP Server (chi)
    ↕ HTTP Streaming (SSE from LLM API)
LLM Provider (OpenAI / Anthropic / 本地模型)
```

核心思路：Go 服务器作为 **SSE 代理**，接收 LLM 的流式输出，转换为标准 SSE 格式转发给客户端。

### 3.2 调用 LLM 流式 API（以 OpenAI 为例）

```go
import (
    "bufio"
    "bytes"
    "encoding/json"
    "net/http"
)

// ChatRequest 是发送给 LLM 的请求体
type ChatRequest struct {
    Model    string    `json:"model"`
    Messages []Message `json:"messages"`
    Stream   bool      `json:"stream"`
}

// 调用 LLM 流式 API，返回 response 供后续逐行读取
func callLLMStream(ctx context.Context, req ChatRequest) (*http.Response, error) {
    body, _ := json.Marshal(req)
    httpReq, err := http.NewRequestWithContext(ctx, "POST", "https://api.openai.com/v1/chat/completions", bytes.NewReader(body))
    if err != nil {
        return nil, err
    }
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("Authorization", "Bearer "+apiKey)

    // 关键：禁止 Go http.Client 自动读取整个响应体
    // http.Client 默认会等待完整响应，但流式 API 不返回 Content-Length
    // Go 的 http.Client 天然支持流式读取——Response.Body 是 io.ReadCloser
    resp, err := http.DefaultClient.Do(httpReq)
    if err != nil {
        return nil, err
    }
    return resp, nil
}
```

### 3.3 完整 SSE 代理 Handler（LLM → 客户端）

```go
func chatStreamHandler(w http.ResponseWriter, r *http.Request) {
    // 1. 解析客户端请求
    var req ChatRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    req.Stream = true // 强制流式

    // 2. 初始化 SSE Writer
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming unsupported", http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no")
    flusher.Flush()

    // 3. 调用 LLM 流式 API
    ctx := r.Context()
    llmResp, err := callLLMStream(ctx, req)
    if err != nil {
        fmt.Fprintf(w, "event: error\ndata: %s\n\n", err.Error())
        flusher.Flush()
        return
    }
    defer llmResp.Body.Close()

    // 4. 逐行读取 LLM 的 SSE 流并转发
    scanner := bufio.NewScanner(llmResp.Body)
    // 增大 buffer 防止长行溢出
    scanner.Buffer(make([]byte, 0, 64*1024), 1024*1024)

    for scanner.Scan() {
        line := scanner.Text()

        // 检查客户端是否断开
        select {
        case <-ctx.Done():
            return
        default:
        }

        // OpenAI SSE 格式：每行以 "data: " 开头
        if strings.HasPrefix(line, "data: ") {
            data := strings.TrimPrefix(line, "data: ")

            // 检测流结束标记
            if data == "[DONE]" {
                fmt.Fprintf(w, "event: done\ndata: [DONE]\n\n")
                flusher.Flush()
                return
            }

            // 解析 LLM 返回的 JSON，提取内容增量
            var chunk struct {
                Choices []struct {
                    Delta struct {
                        Content string `json:"content"`
                    } `json:"delta"`
                    FinishReason *string `json:"finish_reason"`
                } `json:"choices"`
            }
            if err := json.Unmarshal([]byte(data), &chunk); err == nil && len(chunk.Choices) > 0 {
                content := chunk.Choices[0].Delta.Content
                if content != "" {
                    // 转发给客户端（保持 SSE 格式）
                    // 可以直接透传原始 data，也可以提取后重新封装
                    fmt.Fprintf(w, "data: %s\n\n", data)
                    flusher.Flush()
                }
                if chunk.Choices[0].FinishReason != nil {
                    fmt.Fprintf(w, "event: done\ndata: {\"finish_reason\":\"%s\"}\n\n", *chunk.Choices[0].FinishReason)
                    flusher.Flush()
                }
            }
        }
    }

    if err := scanner.Err(); err != nil {
        fmt.Fprintf(w, "event: error\ndata: %s\n\n", err.Error())
        flusher.Flush()
    }
}
```

### 3.4 使用 channel 的解耦模式

对于复杂的流处理（如多个 LLM 调用聚合、工具调用等），使用 channel 解耦：

```go
// Event 表示一个 SSE 事件
type Event struct {
    Type string // event 类型，空表示默认
    Data string // 事件数据
    Err  error  // 错误
}

// LLM 流式生成器，返回只读 channel
func streamLLM(ctx context.Context, req ChatRequest) <-chan Event {
    ch := make(chan Event, 100)
    go func() {
        defer close(ch)
        // ... 调用 LLM，解析响应，发送到 channel
        // 出错时发送 Event{Err: err}
        // 正常时发送 Event{Type: "message", Data: chunk}
        // 结束时发送 Event{Type: "done", Data: "[DONE]"}
    }()
    return ch
}

// SSE Handler 从 channel 读取并写入响应
func streamHandler(w http.ResponseWriter, r *http.Request) {
    flusher, _ := w.(http.Flusher)
    // ... 设置 SSE 头 ...

    ctx := r.Context()
    eventCh := streamLLM(ctx, req)

    for {
        select {
        case <-ctx.Done():
            return
        case evt, ok := <-eventCh:
            if !ok {
                return
            }
            if evt.Err != nil {
                fmt.Fprintf(w, "event: error\ndata: %s\n\n", evt.Err.Error())
                flusher.Flush()
                return
            }
            if evt.Type != "" {
                fmt.Fprintf(w, "event: %s\n", evt.Type)
            }
            fmt.Fprintf(w, "data: %s\n\n", evt.Data)
            flusher.Flush()
        }
    }
}
```

### 3.5 SSE 事件类型设计最佳实践

```
event: metadata     → 发送会话元信息（model, usage, session_id 等）
data: {"model":"gpt-4","session_id":"xxx"}

event: token        → 发送 LLM 生成的 token
data: {"content":"Hello"}

event: tool_call    → 发送工具调用信息
data: {"name":"search","args":{"q":"query"}}

event: done         → 流结束
data: [DONE]

event: error        → 错误
data: {"message":"rate limit exceeded","code":429}
```

### 3.6 错误处理与重试

```go
// LLM API 可能在流中途返回错误
// 需要检查 HTTP 状态码
if llmResp.StatusCode != http.StatusOK {
    body, _ := io.ReadAll(llmResp.Body)
    // 将错误以 SSE 事件发送给客户端
    errEvent := fmt.Sprintf(`{"error":{"message":"%s","status":%d}}`, string(body), llmResp.StatusCode)
    fmt.Fprintf(w, "event: error\ndata: %s\n\n", errEvent)
    flusher.Flush()
    return
}
```

---

## 4. Go HTTP API 生产级实践

### 4.1 统一错误处理

**定义统一错误响应格式：**
```go
// APIError 统一错误响应
type APIError struct {
    Code    int    `json:"code"`           // HTTP 状态码
    Error   string `json:"error"`          // 错误类型
    Message string `json:"message"`        // 人类可读的错误信息
    Details any    `json:"details,omitempty"` // 额外详情
    RequestID string `json:"request_id,omitempty"` // 请求 ID（来自 middleware.RequestID）
}

// ErrorResponse 统一错误写入
func ErrorResponse(w http.ResponseWriter, r *http.Request, status int, errType, message string, details ...any) {
    apiErr := APIError{
        Code:    status,
        Error:   errType,
        Message: message,
        RequestID: middleware.GetReqID(r.Context()),
    }
    if len(details) > 0 {
        apiErr.Details = details[0]
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(apiErr)
}

// 常用错误快捷函数
func BadRequest(w http.ResponseWriter, r *http.Request, msg string) {
    ErrorResponse(w, r, 400, "bad_request", msg)
}
func Unauthorized(w http.ResponseWriter, r *http.Request, msg string) {
    ErrorResponse(w, r, 401, "unauthorized", msg)
}
func NotFound(w http.ResponseWriter, r *http.Request, msg string) {
    ErrorResponse(w, r, 404, "not_found", msg)
}
func InternalError(w http.ResponseWriter, r *http.Request, msg string) {
    ErrorResponse(w, r, 500, "internal_error", msg)
}
```

**自定义 AppError 类型 + 统一 Recoverer：**
```go
// AppError 应用级错误
type AppError struct {
    Code    int
    Type    string
    Message string
    Cause   error
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error { return e.Cause }

func NewAppError(code int, errType, msg string) *AppError {
    return &AppError{Code: code, Type: errType, Message: msg}
}

// ErrorMiddleware 统一捕获 AppError 并写入响应
func ErrorMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 使用自定义 ResponseWriter 包装器捕获状态码
        ww := &responseWriter{ResponseWriter: w, status: 200}
        defer func() {
            if rec := recover(); rec != nil {
                if appErr, ok := rec.(*AppError); ok {
                    ErrorResponse(w, r, appErr.Code, appErr.Type, appErr.Message)
                } else {
                    ErrorResponse(w, r, 500, "panic", fmt.Sprintf("%v", rec))
                }
            }
        }()
        next.ServeHTTP(ww, r)
    })
}
```

### 4.2 中间件链完整配置

```go
func setupRouter() http.Handler {
    r := chi.NewRouter()

    // ── 全局中间件 ──
    r.Use(middleware.RequestID)               // 生成请求 ID
    r.Use(middleware.ClientIPFromXFF(         // 客户端 IP（根据部署选择）
        "10.0.0.0/8",
    ))
    r.Use(middleware.Logger)                  // 结构化请求日志
    r.Use(middleware.Recoverer)               // panic 恢复 + 堆栈打印
    r.Use(middleware.Compress(5))             // gzip 压缩
    r.Use(cors.Handler(cors.Options{          // CORS（见下文）
        AllowedOrigins:   []string{"https://app.example.com"},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
        ExposedHeaders:   []string{"Link"},
        AllowCredentials: false,
        MaxAge:           300,
    }))

    // ── 健康检查（不需要认证）──
    r.Get("/health", healthHandler)
    r.Get("/ready", readinessHandler)

    // ── API v1 ──
    r.Route("/api/v1", func(r chi.Router) {
        r.Use(middleware.Timeout(30 * time.Second)) // 非 SSE 路由设置超时
        r.Use(AuthMiddleware)                        // 认证
        r.Use(RateLimitMiddleware)                   // 限流

        r.Get("/tasks", listTasks)
        r.Post("/tasks", createTask)
        r.Route("/tasks/{taskID}", func(r chi.Router) {
            r.Use(TaskCtxMiddleware)
            r.Get("/", getTask)
            r.Put("/", updateTask)
            r.Delete("/", deleteTask)
        })
    })

    // ── SSE 流式端点（单独配置，不使用 Timeout）──
    r.Route("/stream", func(r chi.Router) {
        r.Use(AuthMiddleware)      // 仍然需要认证
        // 不使用 middleware.Timeout —— SSE 是长连接
        r.Get("/chat", chatStreamHandler)
        r.Get("/events", eventStreamHandler)
    })

    // ── 404 / 405 处理 ──
    r.NotFound(func(w http.ResponseWriter, r *http.Request) {
        NotFound(w, r, "resource not found")
    })
    r.MethodNotAllowed(func(w http.ResponseWriter, r *http.Request) {
        ErrorResponse(w, r, 405, "method_not_allowed", "method not allowed")
    })

    return r
}
```

### 4.3 CORS 配置（来自 go-chi/cors 官方文档）

来自 pkg.go.dev 确认的 `go-chi/cors` v1.2.2 文档：

```go
// CORS 中间件设计为 chi 路由器的顶层中间件
// 注意：在 r.Group() 或 With() 中使用需要手动添加 OPTIONS 路由
r.Use(cors.Handler(cors.Options{
    // AllowedOrigins: []string{"https://foo.com"},  // 指定具体域名
    AllowedOrigins: []string{"https://*", "http://*"},  // 通配符
    // AllowOriginFunc: func(r *http.Request, origin string) bool { return true }, // 自定义函数
    AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
    AllowedHeaders:   []string{"Accept", "Authorization", "Content-Type", "X-CSRF-Token"},
    ExposedHeaders:   []string{"Link"},   // 暴露给前端的自定义响应头
    AllowCredentials: false,               // 是否允许携带凭证
    MaxAge:           300,                 // preflight 缓存时间（秒）
}))
```

**Options 结构体完整字段（来自官方文档）：**
- `AllowedOrigins []string` - 允许的源列表，`*` 表示全部
- `AllowOriginFunc func(r *http.Request, origin string) bool` - 自定义验证函数，设置后忽略 AllowedOrigins
- `AllowedMethods []string` - 允许的 HTTP 方法（默认 HEAD/GET/POST）
- `AllowedHeaders []string` - 允许的非简单请求头
- `ExposedHeaders []string` - 暴露给前端的响应头
- `AllowCredentials bool` - 是否允许凭证
- `MaxAge int` - preflight 缓存时间
- `OptionsPassthrough bool` - 是否让后续 handler 处理 OPTIONS
- `Debug bool` - 调试模式

### 4.4 请求验证

```go
// Validator 接口
type Validator interface {
    Validate() error
}

// decodeAndValidate 解析并验证请求体
func decodeAndValidate(r *http.Request, v Validator) error {
    if err := json.NewDecoder(r.Body).Decode(v); err != nil {
        return NewAppError(400, "invalid_json", "request body is not valid JSON")
    }
    if err := v.Validate(); err != nil {
        return NewAppError(400, "validation_error", err.Error())
    }
    return nil
}

// 使用示例
type CreateTaskRequest struct {
    Title       string `json:"title"`
    Description string `json:"description"`
    Priority    int    `json:"priority"`
}

func (r *CreateTaskRequest) Validate() error {
    if r.Title == "" {
        return errors.New("title is required")
    }
    if len(r.Title) > 200 {
        return errors.New("title must be less than 200 characters")
    }
    if r.Priority < 0 || r.Priority > 5 {
        return errors.New("priority must be between 0 and 5")
    }
    return nil
}

func createTask(w http.ResponseWriter, r *http.Request) {
    var req CreateTaskRequest
    if err := decodeAndValidate(r, &req); err != nil {
        if appErr, ok := err.(*AppError); ok {
            ErrorResponse(w, r, appErr.Code, appErr.Type, appErr.Message)
        } else {
            BadRequest(w, r, err.Error())
        }
        return
    }
    // ... 创建任务 ...
}
```

### 4.5 认证中间件

```go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            Unauthorized(w, r, "missing authorization header")
            return
        }
        // 去掉 "Bearer " 前缀
        token = strings.TrimPrefix(token, "Bearer ")

        // 验证 token（JWT / API Key / 其他）
        user, err := validateToken(token)
        if err != nil {
            Unauthorized(w, r, "invalid token")
            return
        }

        // 将用户信息存入 context
        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 4.6 限流中间件

```go
// 简单的令牌桶限流
func RateLimitMiddleware(next http.Handler) http.Handler {
    // 使用 chi/middleware/httprate 或自定义实现
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        clientIP := middleware.GetClientIP(r.Context())
        // ... 限流逻辑 ...
        next.ServeHTTP(w, r)
    })
}

// 或使用 go-chi/httprate 包
// import "github.com/go-chi/httprate"
// r.Use(httprate.LimitByIP(100, time.Minute)) // 每 IP 每分钟 100 次
```

### 4.7 SSE 特有的中间件注意事项

```go
// SSE 路由需要特殊处理以下中间件：
// 1. Compress 中间件 —— 可能干扰 SSE 流，需排除 SSE 路由
// 2. Timeout 中间件 —— 会中断长连接，必须排除
// 3. CORS —— SSE 的 EventSource API 需要 CORS（如果是跨域）
// 4. Recoverer —— 应该保留，panic 不应中断 SSE 流

// 方案：为 SSE 路由单独分组
r.Group(func(r chi.Router) {
    r.Use(middleware.Recoverer)
    r.Use(AuthMiddleware)
    // 不使用 Compress 和 Timeout
    r.Get("/stream/chat", chatStreamHandler)
})
```

---

## 5. 完整项目结构参考

```
myapp/
├── cmd/
│   └── server/
│       └── main.go          # 入口，启动 HTTP server
├── internal/
│   ├── handler/
│   │   ├── chat.go          # SSE 流式 chat handler
│   │   ├── tasks.go         # CRUD handler
│   │   └── health.go        # 健康检查
│   ├── middleware/
│   │   ├── auth.go          # 认证中间件
│   │   ├── ratelimit.go     # 限流
│   │   └── error.go         # 统一错误处理
│   ├── model/
│   │   ├── task.go          # 数据模型 + Validate()
│   │   └── api_error.go     # AppError 类型
│   ├── service/
│   │   ├── llm.go           # LLM 调用（流式）
│   │   └── task.go          # 业务逻辑
│   └── sse/
│       ├── writer.go        # SSEWriter 封装
│       └── event.go         # Event 类型定义
├── go.mod
└── go.sum
```

---

## 6. 关键要点总结

### SSE 实现核心
1. **`http.Flusher`** 是 SSE 的基础——每次写入后必须 `Flush()`
2. 设置正确的响应头：`Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
3. 通过 `r.Context().Done()` 检测客户端断开
4. 定期发送心跳（注释行 `: heartbeat\n\n`）防止连接超时
5. SSE 事件格式：`field: value\n`，事件间用 `\n\n` 分隔

### chi + SSE 注意事项
1. **SSE 路由不能使用 `middleware.Timeout`**——长连接会被超时取消
2. **SSE 路由慎用 `middleware.Compress`**——可能缓冲数据影响实时性
3. 使用 `r.Group()` 或 `r.Route()` 为 SSE 路由单独配置中间件栈
4. chi 100% 兼容 `net/http`，SSE handler 无需任何特殊适配

### LLM 流式代理最佳实践
1. 使用 `bufio.Scanner` 逐行读取 LLM 的 SSE 响应
2. 增大 Scanner buffer 防止长行溢出（`scanner.Buffer(buf, 1024*1024)`）
3. 检查 HTTP 状态码，LLM 可能在流中途返回错误
4. 使用 channel 解耦 LLM 调用和 SSE 写入
5. 设计清晰的事件类型（`token`, `metadata`, `done`, `error`, `tool_call`）

### 生产级 HTTP API
1. 统一错误响应格式（包含 code, error type, message, request_id）
2. 中间件链分层配置：全局 → 路由组 → 单个路由
3. CORS 使用 `go-chi/cors` 作为顶层中间件
4. 请求验证通过 `Validator` 接口统一处理
5. 使用 `middleware.RequestID` + `middleware.Logger` 实现请求追踪
6. `middleware.Recoverer` 防止 panic 导致进程崩溃

### 参考来源
- Go 标准库 `net/http` 文档：https://pkg.go.dev/net/http （Flusher 接口定义）
- chi v5 文档：https://pkg.go.dev/github.com/go-chi/chi/v5 （路由器、中间件、URL 参数）
- chi/cors 文档：https://pkg.go.dev/github.com/go-chi/cors （CORS 配置 Options 结构体）
- chi 生态包：cors, jwtauth, httprate, httplog, httptracer 等
