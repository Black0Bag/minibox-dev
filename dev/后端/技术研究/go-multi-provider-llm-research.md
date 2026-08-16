# Go 语言多供应商 LLM 接入方案研究报告

## 一、go-openai 库 (github.com/sashabaranov/go-openai)

### 1.1 基本信息
- **GitHub**: https://github.com/sashabaranov/go-openai
- **Stars**: 10.7k | **Forks**: 1.7k | **Contributors**: 185
- **License**: Apache License 2.0
- **Go版本要求**: Go 1.18+
- **最新版本**: v1.41.2 (127 releases)
- **被依赖**: 8.4K 项目

### 1.2 核心接口设计

#### ClientConfig — 客户端配置结构
```go
type ClientConfig struct {
    authToken            string
    BaseURL              string    // 可覆盖为任意 API 端点
    OrgID                string    // OpenAI 组织 ID
    APIType              APIType   // 区分 OpenAI/Azure/Anthropic
    APIVersion           string    // Azure 或 Anthropic 需要
    AssistantVersion     string
    AzureModelMapperFunc func(model string) string  // Azure 部署名映射
    HTTPClient           HTTPDoer  // 可注入自定义 HTTP 客户端
    EmptyMessagesLimit   uint      // 流式空消息上限
}
```

#### APIType — 多供应商类型枚举
```go
type APIType string
const (
    APITypeOpenAI          APIType = "OPEN_AI"
    APITypeAzure           APIType = "AZURE"
    APITypeAzureAD         APIType = "AZURE_AD"
    APITypeCloudflareAzure APIType = "CLOUDFLARE_AZURE"
    APITypeAnthropic       APIType = "ANTHROPIC"
)
```

#### HTTPDoer — 可注入的 HTTP 接口
```go
type HTTPDoer interface {
    Do(req *http.Request) (*http.Response, error)
}
```
这是实现 Key 轮询/负载均衡的关键扩展点——通过注入自定义 HTTPDoer，可以在每次请求时动态替换 Auth Token。

#### Client 结构
```go
type Client struct {
    config         ClientConfig
    requestBuilder utils.RequestBuilder
    createFormBuilder func(io.Writer) utils.FormBuilder
}
```

### 1.3 多供应商兼容方式

go-openai 通过 **BaseURL 覆盖 + APIType 切换** 实现多供应商兼容：

1. **OpenAI 原生**：`DefaultConfig(authToken)` → BaseURL = `https://api.openai.com/v1`
2. **Azure OpenAI**：`DefaultAzureConfig(apiKey, baseURL)` → 自定义 URL 路径含 deployment name
3. **Anthropic**：`DefaultAnthropicConfig(apiKey, baseURL)` → BaseURL = `https://api.anthropic.com/v1`，使用 `anthropic-version` header
4. **任意 OpenAI 兼容端点**：只需修改 `config.BaseURL` 即可指向 DeepSeek、Moonshot、零一万物等

#### 认证 Header 差异处理 (setCommonHeaders)
```go
switch c.config.APIType {
case APITypeAzure, APITypeCloudflareAzure:
    req.Header.Set("api-key", c.config.authToken)
case APITypeAnthropic:
    req.Header.Set("anthropic-version", c.config.APIVersion)
default: // OpenAI / AzureAD
    req.Header.Set("Authorization", fmt.Sprintf("Bearer %s", c.config.authToken))
}
```

### 1.4 错误处理策略

```go
// API 错误（来自服务端 JSON 响应）
type APIError struct {
    Code           any         `json:"code,omitempty"`
    Message        string      `json:"message"`
    Param          *string     `json:"param,omitempty"`
    Type           string      `json:"type"`
    HTTPStatus     string      `json:"-"`
    HTTPStatusCode int         `json:"-"`
    InnerError     *InnerError `json:"innererror,omitempty"` // Azure 专用
}

// 请求错误（网络层/HTTP 层）
type RequestError struct {
    HTTPStatus     string
    HTTPStatusCode int
    Err            error
    Body           []byte
}
```

- `handleErrorResp()` 统一解析错误响应，优先解析为 `APIError`，失败则包装为 `RequestError`
- `RequestError` 实现 `Unwrap()` 支持 `errors.Is/As` 链式判断
- `APIError.UnmarshalJSON` 自定义反序列化，兼容 Azure 的 `innererror` 字段和 code 为 int/string 的情况

### 1.5 ChatCompletion 请求结构（核心数据模型）
```go
type ChatCompletionMessage struct {
    Role             string             `json:"role"`           // system/user/assistant/function/tool/developer
    Content          string             `json:"content,omitempty"`
    MultiContent     []ChatMessagePart                          // 多模态内容
    ReasoningContent string             `json:"reasoning_content,omitempty"` // deepseek-reasoner
    FunctionCall     *FunctionCall      `json:"function_call,omitempty"`
    ToolCalls        []ToolCall         `json:"tool_calls,omitempty"`
    ToolCallID       string             `json:"tool_call_id,omitempty"`
}
```

---

## 二、langchaingo LLM Provider 抽象层 (github.com/tmc/langchaingo)

### 2.1 基本信息
- **GitHub**: https://github.com/tmc/langchaingo
- **Stars**: 9.6k | **Forks**: 1.1k | **Contributors**: 187
- **License**: MIT
- **最新版本**: v0.1.14 (21 releases)
- **被依赖**: 1.9K 项目

### 2.2 核心接口定义

#### Model 接口 — 所有 LLM 的统一抽象
```go
// Model is an interface multi-modal models implement.
type Model interface {
    // 最通用的多模态接口
    GenerateContent(ctx context.Context, messages []MessageContent, options ...CallOption) (*ContentResponse, error)

    // 简化的纯文本接口（已废弃，保留向后兼容）
    Call(ctx context.Context, prompt string, options ...CallOption) (string, error)
}

// 支持推理/思考的模型扩展接口
type ReasoningModel interface {
    Model
    SupportsReasoning() bool
}
```

#### CallOptions — 函数式选项模式
```go
type CallOption func(*CallOptions)

type CallOptions struct {
    Model              string
    MaxTokens          int
    Temperature        float64
    TopK               int
    TopP               float64
    Seed               int
    StopWords          []string
    StreamingFunc      func(ctx context.Context, chunk []byte) error
    StreamingReasoningFunc func(ctx context.Context, reasoningChunk, chunk []byte) error
    Tools              []Tool
    ToolChoice         any
    JSONMode           bool
    ResponseMIMEType   string
    // ... 更多选项
}

// 选项构造器
func WithModel(model string) CallOption
func WithMaxTokens(maxTokens int) CallOption
func WithTemperature(temperature float64) CallOption
func WithStreamingFunc(fn func(ctx context.Context, chunk []byte) error) CallOption
```

### 2.3 支持的 Provider 子目录

langchaingo/llms/ 下每个 provider 有独立实现包：

| Provider | 目录 | 说明 |
|----------|------|------|
| OpenAI | `llms/openai/` | 支持 OpenAI 及兼容 API |
| Anthropic | `llms/anthropic/` | Claude 系列 |
| Google AI | `llms/googleai/` | Gemini |
| Cohere | `llms/cohere/` | Command R+ |
| Mistral | `llms/mistral/` | Mistral 系列 |
| Ollama | `llms/ollama/` | 本地模型 |
| HuggingFace | `llms/huggingface/` | TGI 端点 |
| Bedrock | `llms/bedrock/` | AWS Bedrock |
| Cloudflare | `llms/cloudflare/` | Workers AI |
| ERNIE | `llms/ernie/` | 百度文心 |
| Watsonx | `llms/watsonx/` | IBM |
| Maritaca | `llms/maritaca/` | 巴西 |
| Local | `llms/local/` | 本地可执行 |
| Llamafile | `llms/llamafile/` | llama.cpp |
| Fake | `llms/fake/` | 测试用 |

每个 provider 实现都满足 `Model` 接口，可通过 `llms.GenerateFromSinglePrompt()` 等统一函数调用。

### 2.4 统一错误处理系统

#### ErrorCode 标准化错误码
```go
type ErrorCode string

const (
    ErrCodeUnknown             ErrorCode = "unknown"
    ErrCodeAuthentication      ErrorCode = "authentication"
    ErrCodeRateLimit           ErrorCode = "rate_limit"
    ErrCodeInvalidRequest      ErrorCode = "invalid_request"
    ErrCodeResourceNotFound    ErrorCode = "resource_not_found"
    ErrCodeTimeout             ErrorCode = "timeout"
    ErrCodeCanceled            ErrorCode = "canceled"
    ErrCodeQuotaExceeded       ErrorCode = "quota_exceeded"
    ErrCodeContentFilter       ErrorCode = "content_filter"
    ErrCodeTokenLimit          ErrorCode = "token_limit"
    ErrCodeProviderUnavailable ErrorCode = "provider_unavailable"
    ErrCodeNotImplemented      ErrorCode = "not_implemented"
)
```

#### Error 结构 — 统一错误载体
```go
type Error struct {
    Code     ErrorCode
    Message  string
    Provider string                    // 哪个 provider 产生的错误
    Details  map[string]interface{}    // provider 特定细节
    Cause    error                     // 原始错误
}

// 实现 error, Unwrap, Is 接口
// 支持 errors.Is(err, llms.ErrRateLimit) 判断
```

#### ErrorMapper — Provider 特定错误映射器
```go
type ErrorMapper struct {
    provider string
    matchers []ErrorMatcher
}

type ErrorMatcher struct {
    Match     func(error) bool   // 判断是否匹配
    Code      ErrorCode          // 映射到的标准错误码
    Transform func(error) string // 可选的消息转换
}
```

**映射策略**：通过字符串模式匹配将 provider 特定错误文本映射到标准 ErrorCode：
- `authPatterns`: ["unauthorized", "authentication", "api key", "401"] → `ErrCodeAuthentication`
- `rateLimitPatterns`: ["rate limit", "too many requests", "429"] → `ErrCodeRateLimit`
- `quotaPatterns`: ["quota", "limit exceeded", "insufficient"] → `ErrCodeQuotaExceeded`
- `contentPatterns`: ["content filter", "safety", "blocked"] → `ErrCodeContentFilter`

**Provider 专属 Mapper**：
- `OpenAIErrorMapper()`: 匹配 `invalid_api_key` → 认证错误, `model_not_found` → 资源未找到
- `AnthropicErrorMapper()`: 匹配 `invalid_x_api_key` → 认证错误, `credit_balance` → 配额超限
- `GoogleAIErrorMapper()`: 匹配 `API key not valid` → 认证错误, `RECITATION`/`SAFETY` → 内容过滤

便捷判断函数：`IsRateLimitError(err)`, `IsAuthenticationError(err)`, `IsTimeoutError(err)` 等。

---

## 三、多 API Key 轮询 / 负载均衡 / 故障切换的 Go 实现模式

### 3.1 Round-Robin Key 轮询

**核心思路**：维护一组 API Key，每次请求原子递增索引取模选择 Key。

```go
type KeyRotator struct {
    keys   []string
    index  uint64  // atomic 操作
}

func (r *KeyRotator) Next() string {
    if len(r.keys) == 0 {
        return ""
    }
    idx := atomic.AddUint64(&r.index, 1)
    return r.keys[idx%uint64(len(r.keys))]
}
```

**进阶：带权重的轮询**（Weighted Round-Robin），不同 Key 分配不同权重。

### 3.2 断路器模式 (Circuit Breaker)

**推荐库**: `github.com/sony/gobreaker` (Sony 出品，Go 社区最流行)

断路器三态：
1. **Closed（关闭）**: 正常放行请求，记录失败次数
2. **Open（打开）**: 连续失败达阈值后熔断，直接返回错误不发请求
3. **Half-Open（半开）**: 超时后放行一个试探请求，成功则回 Closed，失败则回 Open

```go
// 与 go-openai 集成示例
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "OpenAI",
    MaxRequests: 5,                  // 半开态最大试探请求数
    Interval:    60 * time.Second,   // 清零计数周期
    Timeout:     30 * time.Second,   // Open → Half-Open 等待时间
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Printf("CB %s: %s → %s", name, from, to)
    },
})

// 包装调用
resp, err := cb.Execute(func() (interface{}, error) {
    return client.CreateChatCompletion(ctx, req)
})
```

### 3.3 故障切换 (Failover) 模式

**策略**：按优先级排列多个 Provider/Key，当前 Provider 失败（超时/5xx/429）时自动切换到下一个。

```go
type FailoverClient struct {
    providers []Provider  // 按优先级排序
    breaker   *gobreaker.CircuitBreaker
}

func (f *FailoverClient) Generate(ctx context.Context, req Request) (Response, error) {
    var lastErr error
    for _, p := range f.providers {
        resp, err := f.breaker.Execute(func() (interface{}, error) {
            return p.Generate(ctx, req)
        })
        if err == nil {
            return resp.(Response), nil
        }
        lastErr = err
        // 可重试错误（429/5xx/超时）→ 切换下一个 provider
        // 不可重试错误（400/401/403）→ 直接返回
        if !isRetryable(err) {
            return Response{}, err
        }
    }
    return Response{}, fmt.Errorf("all providers failed, last error: %w", lastErr)
}
```

### 3.4 与 go-openai 的 Key 轮询集成方案

利用 go-openai 的 `HTTPDoer` 接口注入自定义 Transport：

```go
type KeyRotationTransport struct {
    rotator *KeyRotator
    base    http.RoundTripper
}

func (t *KeyRotationTransport) Do(req *http.Request) (*http.Response, error) {
    key := t.rotator.Next()
    req.Header.Set("Authorization", "Bearer "+key)
    return t.base.RoundTrip(req)
}

// 使用
config := openai.DefaultConfig("") // 初始 token 为空
config.HTTPClient = &http.Client{
    Transport: &KeyRotationTransport{
        rotator: NewKeyRotator([]string{"key1", "key2", "key3"}),
        base:    http.DefaultTransport,
    },
}
client := openai.NewClientWithConfig(config)
```

### 3.5 退避重试 (Exponential Backoff)

**推荐库**: `github.com/cenkalti/backoff/v4`

```go
err = backoff.Retry(func() error {
    resp, err := client.CreateChatCompletion(ctx, req)
    if err != nil {
        if isOpenAIError(err, 429) { // rate limit
            return err // 可重试
        }
        return backoff.Permanent(err) // 不可重试
    }
    return nil
}, backoff.WithMaxRetries(backoff.NewExponentialBackOff(), 3))
```

### 3.6 已知多供应商 LLM 网关项目

| 项目 | GitHub | 语言 | 特点 |
|------|--------|------|------|
| **one-api** | github.com/songquanpeng/one-api | Go | 最流行的 LLM API 网关，支持 OpenAI/Anthropic/Gemini/国产模型统一转发，内置 Key 轮询、渠道权重、负载均衡 |
| **new-api** | github.com/Calcium-Ion/new-api | Go | one-api 的增强 fork，增加更多模型支持 |
| **gobreaker** | github.com/sony/gobreaker | Go | Sony 断路器库，纯 Go 实现 |
| **go-resiliency** | github.com/eapache/go-resiliency | Go | 包含 breaker、retrier、deadline 等弹性模式 |
| **failsafe-go** | github.com/failsafe-go/failsafe | Go | 统一的弹性编程框架，集成重试/断路器/超时/降级 |

---

## 四、OpenAI / Anthropic / Gemini API 差异与统一抽象方案

### 4.1 API 差异对比

| 维度 | OpenAI | Anthropic | Gemini (Google AI) |
|------|--------|-----------|---------------------|
| **Base URL** | `https://api.openai.com/v1` | `https://api.anthropic.com/v1` | `https://generativelanguage.googleapis.com/v1beta` |
| **认证方式** | `Authorization: Bearer sk-xxx` | `x-api-key: sk-ant-xxx` + `anthropic-version: 2023-06-01` | URL 参数 `?key=xxx` 或 `x-goog-api-key` header |
| **Chat 端点** | `/chat/completions` | `/messages` | `/models/{model}:generateContent` |
| **请求格式** | `messages[]` with `role` + `content` | `messages[]` with `role` + `content`, `system` 独立字段 | `contents[]` with `role` + `parts[]` |
| **System Message** | 作为 messages 数组中 role=system 的消息 | 独立的 `system` 顶层字段 | `system_instruction` 字段 |
| **流式响应** | SSE `text/event-stream`，`data: {json}` | SSE `event: content_block_delta` + `data: {json}` | SSE `data: {json}` 或 streamGenerateContent |
| **Function Calling** | `tools[]` + `tool_choice` | `tools[]` + `tool_choice` (格式略有不同) | `functionDeclarations[]` in `tools[]` |
| **Token 计数** | 无独立端点（响应中返回 usage） | `/messages/count_tokens` 端点 | `countTokens` 端点 |
| **速率限制 Header** | `x-ratelimit-*` | `anthropic-ratelimit-*` | `x-ratelimit-*` (有差异) |
| **错误码** | `invalid_api_key`, `model_not_found` | `invalid_x_api_key`, `credit_balance` | `API key not valid`, `RECITATION`, `SAFETY` |

### 4.2 统一抽象方案（基于源码分析的最佳实践）

#### 方案 A：langchaingo 模式 — 接口抽象 + Provider 实现

```
统一 Model 接口
  ├── GenerateContent(ctx, []MessageContent, ...CallOption) → *ContentResponse
  └── Call(ctx, string, ...CallOption) → string

每个 Provider 独立实现：
  openai.LLM    → 实现 Model 接口，内部调用 OpenAI API
  anthropic.LLM → 实现 Model 接口，内部调用 Claude API
  googleai.LLM  → 实现 Model 接口，内部调用 Gemini API

统一 CallOptions: Model, MaxTokens, Temperature, Tools, StreamingFunc...
统一 Error: ErrorCode + Provider + Message + Cause
统一 ErrorMapper: 字符串模式匹配 → 标准错误码
```

**优势**：类型安全、编译期检查、Provider 隔离  
**劣势**：每个 Provider 需单独实现，维护成本高

#### 方案 B：go-openai 模式 — BaseURL 覆盖 + APIType 切换

```
单一 Client + ClientConfig:
  config.BaseURL = "https://api.deepseek.com/v1"  →  接入 DeepSeek
  config.BaseURL = "https://api.moonshot.cn/v1"   →  接入 Moonshot
  config.APIType = APITypeAnthropic               →  接入 Anthropic
```

**优势**：代码改动最小、OpenAI 兼容生态直接复用  
**劣势**：仅兼容 OpenAI API 协议的供应商，非兼容供应商（如原生 Gemini）需额外适配

#### 方案 C：网关代理模式 — one-api/new-api

```
应用 → [统一 OpenAI 格式请求] → LLM 网关 → [协议转换] → 各供应商 API
                                    ↓
                          Key 池轮询 + 断路器 + 负载均衡
```

**优势**：应用层零改动、集中管理 Key 和配额  
**劣势**：增加网络跳数、单点风险

#### 推荐：混合方案

对于需要在 Go 应用内直接集成的场景：

1. **使用 langchaingo 的 Model 接口** 作为统一抽象层
2. **使用 go-openai 作为 OpenAI 兼容 Provider 的底层客户端**（通过 BaseURL 覆盖支持 DeepSeek/Moonshot 等）
3. **注入自定义 HTTPDoer** 实现 Key 轮询和断路器
4. **利用 langchaingo 的 ErrorMapper** 统一错误处理，支持基于错误码的智能重试和故障切换
5. **使用 gobreaker + backoff** 实现断路器和指数退避

### 4.3 智能重试决策矩阵

| 标准错误码 | 是否重试 | 切换 Key | 切换 Provider | 说明 |
|-----------|---------|---------|--------------|------|
| `authentication` | 否 | 是 | 否 | Key 失效，换 Key 但同 Provider |
| `rate_limit` | 是（退避） | 是 | 可选 | 先退避重试，持续则换 Key |
| `quota_exceeded` | 否 | 是 | 是 | 配额用尽，换 Key 或 Provider |
| `timeout` | 是 | 否 | 可选 | 网络问题，重试即可 |
| `provider_unavailable` | 是（退避） | 否 | 是 | 服务端问题，切换 Provider |
| `content_filter` | 否 | 否 | 否 | 请求内容问题，不重试 |
| `token_limit` | 否 | 否 | 否 | 请求太长，不重试 |
| `invalid_request` | 否 | 否 | 否 | 参数错误，不重试 |
| `resource_not_found` | 否 | 否 | 可选 | 模型不存在，可能需切换 Provider |

---

## 五、关键文件索引

所有源码已下载到 workspace：

| 文件 | 来源 | 路径 |
|------|------|------|
| go-openai config.go | ClientConfig/APIType/多供应商配置 | `/workspace/.omnibot/browser/subagent-781baa42/config_1785499745745_599f613d.go` |
| go-openai client.go | Client/HTTPDoer/请求发送/错误处理 | `/workspace/.omnibot/browser/subagent-a86de0ac/client_1785499773780_8cb866ba.go` |
| go-openai error.go | APIError/RequestError 错误定义 | `/workspace/.omnibot/browser/subagent-a86de0ac/error_1785499847223_a3b5258e.go` |
| go-openai chat.go | ChatCompletion 消息/请求结构 | `/workspace/.omnibot/browser/subagent-a86de0ac/chat_1785499911933_82fed967.go` |
| langchaingo llms.go | Model 接口定义 | `/workspace/.omnibot/browser/subagent-a86de0ac/llms_1785499747529_5bd72c6d.go` |
| langchaingo options.go | CallOptions/函数式选项 | `/workspace/.omnibot/browser/subagent-a86de0ac/options_1785499775939_1a7ac643.go` |
| langchaingo errors.go | ErrorCode/Error 统一错误系统 | `/workspace/.omnibot/browser/subagent-a86de0ac/errors_1785499909878_5cc5a61e.go` |
| langchaingo errors_mapper.go | ErrorMapper/Provider 错误映射 | `/workspace/.omnibot/browser/subagent-a86de0ac/errors_mapper_1785499849521_353064f7.go` |

---

## 六、总结

1. **go-openai** 是 Go 生态中最成熟的 OpenAI 客户端，通过 `BaseURL` + `APIType` + `HTTPDoer` 三个扩展点支持多供应商接入，适合需要直接控制 HTTP 层的场景。

2. **langchaingo** 提供了完整的 Provider 抽象层，统一 `Model` 接口 + `CallOptions` 函数式选项 + 标准化 `ErrorMapper` 错误系统，支持 14+ 个 Provider，适合需要在不同 LLM 间切换的复杂应用。

3. **多 Key 轮询** 通过 `atomic` 计数器 + 取模实现；**断路器** 推荐使用 `sony/gobreaker`；**退避重试** 推荐使用 `cenkalti/backoff`；**故障切换** 通过遍历 Provider 列表 + 错误码判断实现。

4. **统一抽象的最佳实践** 是采用 langchaingo 的接口抽象 + go-openai 的底层客户端 + 自定义 HTTPDoer 注入 Key 轮询和断路器，配合标准化的错误码系统实现智能重试和故障切换决策。
