# 技巧三十一：Go 语言的 AI 适配实践——写给 Go 开发者的 12 条原则

> 核心思想：Go 的简单性和强约定恰好是 AI 训练数据中最丰富的部分——利用这一点

---

## 为什么 Go 天然适合 AI 协作

Go 语言有几个特性让它成为 AI 编码助手最"擅长"的语言之一：

1. **极简语法**——关键字只有 25 个，AI 不太可能生成语法错误
2. **强约定文化**——`gofmt` 强制统一格式，消除了风格猜测
3. **标准库覆盖率高**——标准库在 AI 训练数据中占比极高，生成质量最好
4. **显式错误处理**——`if err != nil` 模式极度一致，AI 很少在这里出错

但"适合"不等于"随便写"。Go 有一些特定的模式会让 AI 生成质量飙升，也有一些特定的坑会让 AI 频繁犯错。

---

## 第一层：让 AI 读懂你的代码

### 原则一：每个导出符号都要有文档注释

AI 读代码的第一步不是看实现，是看注释。Go 的文档注释是 AI 理解意图的入口：

```go
// UserService 处理用户业务逻辑。它验证输入、协调 store 层、发布领域事件。
type UserService struct { ... }

// CreateUser 验证输入并持久化新用户。
// 如果邮箱已存在，返回 ErrDuplicateEmail。
func (s *UserService) CreateUser(ctx context.Context, req CreateUserRequest) (*User, error) { ... }
```

没有注释的导出符号，AI 只能从签名猜测意图——猜错的概率显著增加。

### 原则二：类型要显式，拒绝 interface{} 和 map[string]any

```go
// 好——AI 知道该填什么
type Server struct {
    Addr     string        // e.g. ":8080"
    Timeout  time.Duration // 读写超时
    Handler  http.Handler
    Logger   *slog.Logger
}

// 差——AI 无法推断 map 的结构
type Server struct {
    Config map[string]interface{}
}
```

AI 不能推断无类型 map 的 shape。给它一个 struct，它就能生成正确的字段赋值。

### 原则三：类型和方法放在同一个文件里

当 AI 能在一个 Read 操作内同时看到类型定义和它的方法时，它生成的方法签名更准确、理解数据模型更快。如果类型在 `model.go`，方法在 `service.go`，AI 需要两次 Read 才能建立完整认知。

---

## 第二层：项目结构——用标准约定喂 AI 最好的数据

### 原则四：遵循标准 Go 项目布局

```
project/
├── cmd/
│   └── myapp/
│       └── main.go
├── internal/
│   ├── handler/
│   ├── service/
│   └── store/
├── pkg/          # 仅限真正公开的 API
├── api/          # OpenAPI/proto 定义
├── go.mod
└── go.sum
```

AI 在训练数据中见过成千上万遵循这个约定的仓库，它能生成正确的代码。如果你用自定义目录结构（`src/`、`lib/`、`core/`），AI 会困惑。

### 原则五：扁平优于嵌套

```
好：
internal/user/service.go
internal/user/store.go

差：
internal/domain/user/application/service.go
internal/domain/user/infrastructure/persistence/store.go
```

深层嵌套迫使 AI 穿越多层目录才能找到文件。扁平结构下，一次 `Glob("internal/user/**")` 就能返回所有相关文件。

### 原则六：internal/ 包保持小而聚焦

每个 `internal/` 子包 5-10 个文件以内，包名用名词描述职责：`store`、`handler`、`validator`。不要把所有逻辑塞进一个巨大的 `internal/service/` 包。

---

## 第三层：AI 最常犯的 Go 错误——提前防范

### 原则七：context 透传，永远不要在非 main 包用 Background

这是 AI 犯得最多的错误：

```go
// AI 可能生成这个——错误
func (s *Service) GetUser(id string) (*User, error) {
    return s.store.FindByID(context.Background(), id)
}

// 正确——透传调用方的 context
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    return s.store.FindByID(ctx, id)
}
```

**CLAUDE.md 中应明确写**：`Always pass context.Context as first parameter. Never use context.Background() in non-main packages.`

### 原则八：goroutine 必须有清理机制

AI 高频生成的另一个错误——启动 goroutine 但不处理退出：

```go
// AI 常见生成——goroutine 泄漏
go s.poll()

// 正确——监听 ctx 取消信号
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            s.poll()
        }
    }
}()
```

### 原则九：保持 receiver 名称一致

AI 分多次生成同一个类型的方法时，receiver 名可能不一致：

```go
func (s *Server) Start() {}     // 第一次生成
func (srv *Server) Stop() {}    // 后来生成——不一致
```

**解决方案**：在类型定义旁注释标明约定的 receiver 名，或者在 CLAUDE.md 中写明：`Receiver names: 1-2 letters, consistent across all methods on a type.`

---

## 第四层：接口与错误——Go 的 AI 友好设计

### 原则十：小接口 + 消费者端定义

AI 倾向于生成臃肿接口，因为它看到方法就想"帮忙"加进去：

```go
// AI 可能生成的臃肿接口
type Repository interface {
    Create(ctx, entity) error
    Read(ctx, id) (entity, error)
    Update(ctx, entity) error
    Delete(ctx, id) error
    List(ctx, opts) ([]entity, error)
    Count(ctx, opts) (int, error)
    Search(ctx, query) ([]entity, error)
}

// 正确做法——消费者只需要什么方法，就定义什么接口
// 在调用方定义，而不是在实现方
type UserCreator interface {
    Create(ctx context.Context, user User) error
}
```

Go 谚语："The bigger the interface, the weaker the abstraction." 对 AI 尤其如此——小接口让 AI 能精确推断实现要求。

### 原则十一：错误处理保持 Go 惯用模式

Go 的错误处理在 AI 训练数据中极度一致。偏离惯用模式会导致 AI 生成混乱：

```go
// 标准模式——AI 生成最可靠
if err != nil {
    return fmt.Errorf("parse config %s: %w", path, err)
}

// 哨兵错误用 errors.Is 检查
var ErrDuplicateEmail = errors.New("duplicate email")

// 自定义错误类型用 errors.As 检查
var ve *ValidationError
if errors.As(err, &ve) { ... }
```

**关键**：
- 错误字符串小写开头，无尾部标点
- 用 `%w` 包装错误（不是 `%v`）
- 哨兵错误命名：`var ErrXxx = errors.New(...)`

### 原则十二：table-driven 测试是 AI 的舒适区

这是 Go 标准库和训练数据中最具代表性的测试模式。AI 生成 table-driven 测试非常可靠：

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    Config
        wantErr bool
    }{
        {name: "valid config", input: "testdata/ok.json", want: defaultConfig()},
        {name: "missing file", input: "missing.json", wantErr: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseConfig(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseConfig() error = %v, wantErr %v", err, tt.wantErr)
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Fatalf("ParseConfig() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

同时添加 `example_test.go`——Go 的 `Example*` 函数对 AI 极有价值，因为它展示了期望的输入输出模式。

---

## 第五层：工具链——用 linter 兜底 AI 的错误

### golangci-lint 关键 linter

AI 生成的 Go 代码**必须**过 linter。以下是针对 AI 常见错误的 linter 配置：

| Linter | 捕获的 AI 错误 |
|--------|--------------|
| `errcheck` | 忘记处理错误 |
| `bodyclose` | 忘记关闭 HTTP 响应体 |
| `sqlclosecheck` | 忘记关闭 SQL Rows |
| `gosimple` | 生成冗余代码 |
| `ineffassign` | 无用赋值 |
| `nilerr` | 返回 nil 而非错误 |
| `err113` | 错误包装不规范 |
| `revive` | Go 惯例违规 |
| `staticcheck` | nil 解引用、错误比较 |

### CLAUDE.md 模板——Go 项目专用

```markdown
# Project Conventions

## Build & Test
- Build: `go build ./...`
- Test: `go test ./...`
- Lint: `golangci-lint run`

## Code Style
- Follow Effective Go and Google Go Style Guide
- All exported symbols must have doc comments
- Receiver names: 1-2 letters, consistent across all methods on a type
- Error strings: lowercase, no trailing punctuation
- Always pass context.Context as first parameter
- Never use context.Background() in non-main packages
- Use %w for error wrapping with fmt.Errorf
- Prefer table-driven tests

## Forbidden
- No global mutable state
- No init() functions
- No interface{} or reflect
- No panics for recoverable errors
- No generated code in git (use go:generate)
```

---

## Go 特有的 AI 协作优势

| Go 特性 | 对 AI 的好处 |
|---------|------------|
| `gofmt` 强制格式 | 消除风格猜测，AI 不会在格式上犯错 |
| 接口消费者端定义 | AI 只需理解调用方需求，不需要猜测完整接口 |
| 显式错误处理 | `if err != nil` 模式训练数据极多，生成质量高 |
| 标准目录布局 | AI 见过成千上万同样布局的仓库 |
| Table-driven 测试 | AI 生成最可靠的 Go 测试模式 |
| `go generate` 注释 | 向 AI 发出"这里有代码生成"的信号 |
| `//go:embed` | 明确的资源嵌入语义，AI 不会误解 |
| `internal/` 可见性 | 编译器强制包边界，AI 不会越权引用 |

---

## 总结：Go + AI 的 12 条原则速查表

| # | 原则 | 核心原因 |
|---|------|---------|
| 1 | 每个导出符号写文档注释 | AI 的第一信息源 |
| 2 | 类型显式，拒绝 interface{} | AI 无法推断无类型结构 |
| 3 | 类型和方法放同一个文件 | 一次 Read 建立完整认知 |
| 4 | 标准 Go 项目布局 | AI 训练数据中最多 |
| 5 | 扁平优于嵌套 | 减少搜索链深度 |
| 6 | internal/ 包小而聚焦 | 单次 Glob 返回全部文件 |
| 7 | context 透传 | AI 最高频的错误 |
| 8 | goroutine 有清理机制 | AI 第二高频的错误 |
| 9 | receiver 名称一致 | 避免跨次生成不一致 |
| 10 | 小接口 + 消费者端定义 | 避免 AI 生成臃肿接口 |
| 11 | 错误处理保持惯用模式 | 偏离导致 AI 生成混乱 |
| 12 | table-driven 测试 | AI 最可靠的测试模式 |

---

## 关键洞察

> **Go 是 AI 最"舒适"的语言之一——但舒适的前提是你遵循 Go 社区的约定。Go 的约定恰好是 AI 训练数据中最丰富的模式。偏离约定不是"个性化"，是"给 AI 制造噪音"。写标准的 Go，就是写 AI 友好的 Go。**

---

## 给抖音文案的启发

> "Go 可能是 AI 时代最聪明的语言选择。不是因为它性能好、并发强——而是因为 Go 社区强迫所有人写一样的代码。gofmt 强制格式，Effective Go 统一风格，标准布局人人遵守。这些约定恰好是 AI 训练数据中最多的模式。你写的 Go 越'标准'，AI 就越准确。换句话说：Go 社区的'无聊'，恰好是 AI 时代的'聪明'。"
