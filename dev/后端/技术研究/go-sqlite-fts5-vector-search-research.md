# Go 语言 SQLite + FTS5 全文搜索与向量搜索实现方案

> 基于互联网搜索整理，涵盖 modernc.org/sqlite 纯 Go 驱动、FTS5 中文分词、纯 Go 向量余弦相似度搜索、BM25+向量 RRF 融合搜索。

---

## 1. modernc.org/sqlite 纯 Go SQLite 驱动

### 1.1 概述

- **包名**: `modernc.org/sqlite`
- **最新版本**: v1.55.0 (截至 2026-07-20)
- **底层 SQLite 版本**: 3.53.3
- **许可证**: BSD-3-Clause
- **被引用次数**: 3,518 个包
- **特点**: **完全 CGO-free**，通过将 C SQLite 源码转译(transpile)为 Go 实现
- **赞助商**: Tailscale (企业级赞助)
- **支持平台**: linux/darwin/freebsd/windows (amd64, arm64, 386, arm, loong64, ppc64le, riscv64, s390x)

### 1.2 基本使用方式

```go
import (
    "database/sql"
    _ "modernc.org/sqlite"
)

func main() {
    // 方式1: 直接文件路径
    db, err := sql.Open("sqlite", "/tmp/mydata.sqlite")
    
    // 方式2: URI 带 PRAGMA 参数 (推荐生产使用)
    db, err = sql.Open("sqlite", "file:///tmp/mydata.sqlite?_pragma=journal_mode(WAL)&_pragma=foreign_keys(1)&_pragma=synchronous(NORMAL)&_pragma=busy_timeout(5000)")
    
    // 方式3: 内存数据库
    db, err = sql.Open("sqlite", ":memory:")
    
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
}
```

### 1.3 DSN 参数说明

| 参数 | 别名 | 说明 |
|------|------|------|
| `_busy_timeout` | `_timeout` | PRAGMA busy_timeout (毫秒) |
| `_foreign_keys` | `_fk` | PRAGMA foreign_keys (0/1) |
| `_journal_mode` | `_journal` | PRAGMA journal_mode (WAL/DELETE/...) |
| `_synchronous` | `_sync` | PRAGMA synchronous (OFF/NORMAL/FULL) |
| `_auto_vacuum` | `_vacuum` | PRAGMA auto_vacuum (NONE/FULL/INCREMENTAL) |
| `_pragma` | - | 任意 PRAGMA，可多次指定，&分隔 |
| `_txlock` | - | 事务锁模式 (deferred/immediate/exclusive) |

### 1.4 注册自定义 SQL 函数 (UDF)

这是实现 BM25 自定义打分、向量相似度计算的关键能力：

```go
// 注册标量函数 (用于 SQL 中调用)
sqlite.RegisterScalarFunction("cosine_similarity", 2, 
    func(ctx *sqlite.FunctionContext, args []driver.Value) (driver.Value, error) {
        // args[0], args[1] 是两个向量 (BLOB 类型)
        vec1 := args[0].([]byte)
        vec2 := args[1].([]byte)
        return cosineSimilarity(vec1, vec2), nil
    })

// 注册确定性函数 (可被 SQLite 优化器用于更多场景)
sqlite.RegisterDeterministicScalarFunction("bm25_score", -1,
    func(ctx *sqlite.FunctionContext, args []driver.Value) (driver.Value, error) {
        // 可变参数: bm25_score(term_freq, doc_len, avg_doc_len, doc_freq, total_docs, ...)
        // 实现 BM25 打分逻辑
        return calculateBM25(args), nil
    })

// 注册自定义排序规则
sqlite.RegisterCollationUtf8("chinese_pinyin", func(left, right string) int {
    // 按拼音排序
    return strings.Compare(pinyin(left), pinyin(right))
})

// 注册连接钩子 (每次新连接打开后调用)
sqlite.RegisterConnectionHook(func(conn sqlite.ExecQuerierContext, dsn string) error {
    // 在每个新连接上执行初始化 SQL
    _, err := conn.ExecContext(context.Background(), 
        "PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL;", nil)
    return err
})
```

### 1.5 虚拟表 (vtab) — 实现向量搜索的关键

modernc.org/sqlite 提供了 `modernc.org/sqlite/vtab` 包，支持用纯 Go 实现 SQLite 虚拟表模块，这是实现向量搜索的基础：

```go
// 向量搜索虚拟表示例 (sqlite-vec 风格)
// SQL: CREATE VIRTUAL TABLE vec_docs USING vec(dim=128, metric="cosine")
//
// 模块读取参数 (dim, metric), 通过 ctx.Declare 声明 schema:
//   CREATE TABLE vec_docs(id, embedding, content HIDDEN)
// 然后通过 BestIndex/Filter 实现搜索逻辑

// modernc.org/sqlite 已内置 sqlite-vec 扩展 (vec/ 目录)
// 但也可以通过 vtab 自定义实现
```

**vtab API 要点**:
- `vtab.RegisterModule(db, name, module)` — 注册模块，仅对新连接生效
- `ctx.Declare("CREATE TABLE ...")` — 在 Create/Connect 中声明 schema
- `BestIndex` — 检查约束条件 (Constraints), 设置 ArgIndex 和 Omit
- `Cursor.Filter(idxNum, idxStr, vals)` — 执行搜索
- 支持的操作符: EQ/NE/GT/GE/LT/LE/MATCH/LIKE/GLOB/REGEXP/FUNCTION/LIMIT/OFFSET

### 1.6 性能特点

- **CGO-free**: 纯 Go 实现，交叉编译简单，无 C 工具链依赖
- **性能**: 相比 mattn/go-sqlite3 (CGO 版)，读性能接近，写性能略慢 (约 1.2-2x)
- **内存**: 转译后的 Go 代码内存占用略高
- **WAL 模式**: 完全支持 WAL，适合并发读场景
- **可插拔页缓存**: v1.53.0+ 支持 `RegisterPageCache` 自定义页缓存策略
- **VolatileArgs 优化**: FunctionImpl 中的零拷贝参数传递选项，减少 UDF 调用开销

### 1.7 内置 sqlite-vec 扩展

modernc.org/sqlite v1.55.0 的 `vec/` 目录中已内置 sqlite-vec 扩展的转译版本，支持向量存储和搜索：

```sql
-- 创建向量表
CREATE VIRTUAL TABLE vec_docs USING vec0(
    embedding float[768]  -- 768维向量
);

-- 插入向量
INSERT INTO vec_docs(rowid, embedding) VALUES (1, ?);

-- KNN 搜索
SELECT rowid, distance 
FROM vec_docs 
WHERE embedding MATCH ? 
ORDER BY distance 
LIMIT 10;
```

---

## 2. Go 中 FTS5 中文分词优化方案

### 2.1 FTS5 内置分词器

SQLite FTS5 提供以下内置分词器：
- **unicode61**: 默认分词器，按 Unicode 字符类别分词，适用于拉丁语系
- **ascii**: 简单 ASCII 分词器
- **porter**: Porter 词干提取算法，用于英文
- **trigram**: 三元组分词器，适用于 CJK 模糊匹配

### 2.2 unicode61 对中文的问题

**核心问题**: `unicode61` 分词器**不支持 CJK (中日韩) 文本**。原因是：
- unicode61 依赖 Unicode 字符类别来识别词边界
- CJK 字符没有空格分隔，unicode61 会将整个中文字符串视为一个 token 或逐字符分词
- 这导致 FTS5 对中文搜索几乎无法使用

参考: Stack Overflow 讨论 (https://stackoverflow.com/questions/52422437/) 和多个 GitHub issue 确认此问题。

### 2.3 中文分词优化方案

#### 方案 A: 预分词 + unicode61 (推荐，纯 Go 可行)

在写入 FTS5 之前，先用中文分词库对文本进行分词，用空格分隔词语：

```go
import (
    "github.com/wangbin/jiebago"  // Go 中文分词库
    // 或 github.com/yanyiwu/gojieba (结巴分词 Go 绑定)
)

// 使用 gojieba 分词
import "github.com/yanyiwu/gojieba"

var seg = gojieba.NewJieba()

// 分词后用空格连接，再写入 FTS5
func TokenizeChinese(text string) string {
    words := seg.Cut(text, true) // true = HMM 模式，更精准
    return strings.Join(words, " ")
}

// 写入数据库
func InsertDocument(db *sql.DB, id int64, content string) error {
    tokenized := TokenizeChinese(content)
    _, err := db.Exec(
        "INSERT INTO docs_fts(rowid, content) VALUES (?, ?)",
        id, tokenized,
    )
    return err
}

// 查询时也需要对查询词分词
func SearchFTS5(db *sql.DB, query string) ([]SearchResult, error) {
    tokenizedQuery := TokenizeChinese(query)
    // FTS5 的 unicode61 分词器会用空格分割 tokenizedQuery
    // 然后进行 BM25 打分
    rows, err := db.Query(`
        SELECT rowid, content, bm25(docs_fts) as score
        FROM docs_fts 
        WHERE docs_fts MATCH ?
        ORDER BY score
        LIMIT 20
    `, tokenizedQuery)
    // ...
}
```

```sql
-- 创建 FTS5 表，使用 unicode61 分词器
CREATE VIRTUAL TABLE docs_fts USING fts5(
    content,
    tokenize = 'unicode61'
);
```

#### 方案 B: trigram 分词器 (SQLite 3.34+)

```sql
-- trigram 分词器将文本按 3 个字符的滑动窗口分词
-- 对中文有一定效果，因为中文常用词多为 2-3 字
CREATE VIRTUAL TABLE docs_fts USING fts5(
    content,
    tokenize = 'trigram'
);

-- 查询
SELECT * FROM docs_fts WHERE docs_fts MATCH '搜索引擎';
```

**trigram 优缺点**:
- 优点: 无需外部分词库，纯 SQLite 内置
- 缺点: 索引体积大，短词 (1-2字) 搜索效果差，精确度不如专业分词

#### 方案 C: 自定义 FTS5 分词器 (通过 modernc.org/sqlite UDF)

利用 modernc.org/sqlite 的自定义函数注册能力，在 Go 层面实现分词逻辑：

```go
// 方案: 创建一个 "tokenize_text" SQL 函数
// 在写入和查询时调用该函数进行分词
sqlite.RegisterDeterministicScalarFunction("tokenize_cn", 1,
    func(ctx *sqlite.FunctionContext, args []driver.Value) (driver.Value, error) {
        text := args[0].(string)
        words := jiebaSegmenter.Cut(text, true)
        return strings.Join(words, " "), nil
    })

// SQL 使用
// INSERT INTO docs_fts(content) VALUES (tokenize_cn('中文文本内容'));
// SELECT * FROM docs_fts WHERE docs_fts MATCH tokenize_cn('搜索关键词');
```

#### 方案 D: bigram 分词 (简化方案)

对于不需要精确分词的场景，可以使用二元组：

```go
// 将中文文本转为二元组序列
func BigramTokenize(text string) string {
    runes := []rune(text)
    if len(runes) < 2 {
        return text
    }
    var tokens []string
    for i := 0; i < len(runes)-1; i++ {
        // 跳过标点和空格
        if !unicode.IsLetter(runes[i]) || !unicode.IsLetter(runes[i+1]) {
            continue
        }
        tokens = append(tokens, string(runes[i])+string(runes[i+1]))
    }
    return strings.Join(tokens, " ")
}
```

### 2.4 FTS5 + BM25 打分

FTS5 内置 `bm25()` 函数，返回负值 (越小越相关)：

```sql
-- BM25 搜索，使用 FTS5 内置打分
SELECT 
    rowid,
    content,
    -bm25(docs_fts) as bm25_score  -- 取正值，越大越相关
FROM docs_fts 
WHERE docs_fts MATCH ?
ORDER BY bm25_score
LIMIT 20;

-- 带权重: bm25(docs_fts, 10.0, 1.0) 表示第一列权重10，第二列权重1
SELECT rowid, title, content, -bm25(docs_fts, 10.0, 1.0) as score
FROM docs_fts
WHERE docs_fts MATCH ?
ORDER BY score
LIMIT 20;
```

---

## 3. Go 中向量余弦相似度搜索 (纯 Go 方案)

### 3.1 余弦相似度公式

```
cos(A, B) = (A · B) / (||A|| × ||B||)

其中:
A · B = Σ(A[i] × B[i])           (点积)
||A|| = √(Σ(A[i]²))              (L2 范数)
```

### 3.2 纯 Go 余弦相似度实现

```go
package vector

import (
    "math"
)

// CosineSimilarity 计算两个 float32 向量的余弦相似度
func CosineSimilarity(a, b []float32) float32 {
    if len(a) != len(b) {
        return 0
    }
    var dotProduct, normA, normB float32
    for i := 0; i < len(a); i++ {
        dotProduct += a[i] * b[i]
        normA += a[i] * a[i]
        normB += b[i] * b[i]
    }
    if normA == 0 || normB == 0 {
        return 0
    }
    return dotProduct / (float32(math.Sqrt(float64(normA))) * float32(math.Sqrt(float64(normB))))
}

// 预归一化向量的余弦相似度 (更快，因为省去范数计算)
// 如果向量已预归一化 (||A|| = ||B|| = 1)，则 cos(A,B) = A · B
func DotProduct(a, b []float32) float32 {
    var sum float32
    for i := 0; i < len(a); i++ {
        sum += a[i] * b[i]
    }
    return sum
}

// L2Normalize 将向量归一化为单位向量
func L2Normalize(vec []float32) []float32 {
    var sum float64
    for _, v := range vec {
        sum += float64(v) * float64(v)
    }
    norm := math.Sqrt(sum)
    if norm == 0 {
        return vec
    }
    result := make([]float32, len(vec))
    for i, v := range vec {
        result[i] = float32(float64(v) / norm)
    }
    return result
}
```

### 3.3 暴力搜索 (Brute Force) — 适合小规模数据

```go
package vector

import "sort"

type VectorRecord struct {
    ID       int64
    Vector   []float32
    Metadata map[string]interface{}
}

type SearchResult struct {
    ID       int64
    Score    float32
    Metadata map[string]interface{}
}

// BruteForceSearch 暴力搜索 Top-K 最相似的向量
func BruteForceSearch(records []VectorRecord, query []float32, topK int) []SearchResult {
    results := make([]SearchResult, 0, len(records))
    for _, rec := range records {
        score := CosineSimilarity(query, rec.Vector)
        results = append(results, SearchResult{
            ID:       rec.ID,
            Score:    score,
            Metadata: rec.Metadata,
        })
    }
    // 降序排序
    sort.Slice(results, func(i, j int) bool {
        return results[i].Score > results[j].Score
    })
    if topK > len(results) {
        topK = len(results)
    }
    return results[:topK]
}
```

### 3.4 使用 SQLite 存储向量 + Go 计算 (推荐方案)

```go
package search

import (
    "database/sql"
    "encoding/binary"
    "math"
    "sort"
)

// 向量序列化为 BLOB 存储
func VectorToBytes(vec []float32) []byte {
    buf := make([]byte, 4*len(vec))
    for i, v := range vec {
        binary.LittleEndian.PutUint32(buf[i*4:], math.Float32bits(v))
    }
    return buf
}

// 从 BLOB 反序列化
func BytesToVector(data []byte) []float32 {
    n := len(data) / 4
    vec := make([]float32, n)
    for i := 0; i < n; i++ {
        vec[i] = math.Float32frombits(binary.LittleEndian.Uint32(data[i*4:]))
    }
    return vec
}

// VectorStore 基于 SQLite 的向量存储
type VectorStore struct {
    db *sql.DB
}

func NewVectorStore(db *sql.DB) (*VectorStore, error) {
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS vectors (
            id INTEGER PRIMARY KEY,
            embedding BLOB NOT NULL,
            content TEXT,
            metadata TEXT
        )
    `)
    if err != nil {
        return nil, err
    }
    return &VectorStore{db: db}, nil
}

func (vs *VectorStore) Insert(id int64, vec []float32, content string) error {
    _, err := vs.db.Exec(
        "INSERT OR REPLACE INTO vectors(id, embedding, content) VALUES (?, ?, ?)",
        id, VectorToBytes(vec), content,
    )
    return err
}

// Search 加载所有向量到内存，用 Go 计算余弦相似度
func (vs *VectorStore) Search(query []float32, topK int) ([]SearchResult, error) {
    rows, err := vs.db.Query("SELECT id, embedding, content FROM vectors")
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var results []SearchResult
    for rows.Next() {
        var id int64
        var embeddingBytes []byte
        var content string
        if err := rows.Scan(&id, &embeddingBytes, &content); err != nil {
            return nil, err
        }
        vec := BytesToVector(embeddingBytes)
        score := CosineSimilarity(query, vec)
        results = append(results, SearchResult{
            ID:      id,
            Score:   score,
            Content: content,
        })
    }

    sort.Slice(results, func(i, j int) bool {
        return results[i].Score > results[j].Score
    })

    if topK > len(results) {
        topK = len(results)
    }
    return results[:topK], nil
}
```

### 3.5 使用 SQLite UDF 在 SQL 层计算 (利用 modernc.org/sqlite)

```go
// 注册余弦相似度函数到 SQLite
sqlite.RegisterDeterministicScalarFunction("vec_cosine", 2,
    func(ctx *sqlite.FunctionContext, args []driver.Value) (driver.Value, error) {
        blob1, ok1 := args[0].([]byte)
        blob2, ok2 := args[1].([]byte)
        if !ok1 || !ok2 {
            return 0.0, fmt.Errorf("expected BLOB arguments")
        }
        vec1 := BytesToVector(blob1)
        vec2 := BytesToVector(blob2)
        return CosineSimilarity(vec1, vec2), nil
    })

// SQL 中使用
// SELECT id, content, vec_cosine(embedding, ?) AS similarity
// FROM vectors
// WHERE vec_cosine(embedding, ?) > 0.5
// ORDER BY similarity DESC
// LIMIT 10;
```

### 3.6 性能优化策略

1. **预归一化**: 入库时归一化向量，查询时只需点积，省去范数计算
2. **批量加载**: 一次性加载所有向量到内存，避免逐行查询
3. **SIMD 优化**: 使用 `github.com/klauspost/cpuid` 检测 CPU 指令集，用汇编加速点积
4. **HNSW 索引**: 大规模数据使用 HNSW (Hierarchical Navigable Small World) 图索引
5. **PQ 量化**: 乘积量化压缩向量，减少内存占用

---

## 4. BM25 + 向量 RRF 融合搜索算法

### 4.1 BM25 算法

BM25 (Best Matching 25) 是信息检索中最经典的排序算法，FTS5 内置支持。

**BM25 公式**:

```
BM25(D, Q) = Σ_{t ∈ Q} IDF(t) × [ f(t,D) × (k₁ + 1) ] / [ f(t,D) + k₁ × (1 - b + b × |D| / avgdl) ]

其中:
- f(t,D): 词 t 在文档 D 中的词频 (term frequency)
- |D|: 文档 D 的长度 (token 数)
- avgdl: 语料库中文档的平均长度
- k₁: 词频饱和参数，通常 1.2 ~ 2.0 (默认 1.2)
- b: 长度归一化参数，通常 0.75

IDF(t) = ln[ (N - n(t) + 0.5) / (n(t) + 0.5) + 1 ]

其中:
- N: 语料库中文档总数
- n(t): 包含词 t 的文档数
```

**FTS5 中的 BM25**: FTS5 的 `bm25()` 函数返回负值 (负 BM25 分数)，使用时取负值即可。

### 4.2 RRF (Reciprocal Rank Fusion) 算法

RRF 是一种简单但有效的多路搜索结果融合方法，由 Cormack et al. (2009) 提出。

**RRF 公式**:

```
RRF_score(d) = Σ_{r ∈ R} 1 / (k + rank_r(d))

其中:
- d: 文档
- R: 排序列表集合 (如 BM25 排序列表、向量搜索排序列表)
- rank_r(d): 文档 d 在排序列表 r 中的排名 (1-based)
- k: 平滑常数，通常 k = 60 (原始论文推荐值)
```

**RRF 的优点**:
1. **无需分数校准**: 只用排名，不需要不同打分系统的分数可比
2. **参数少**: 只有一个参数 k
3. **鲁棒性强**: 对异常值不敏感
4. **实现简单**: 几行代码即可实现

### 4.3 Go 实现 BM25 + 向量 RRF 融合搜索

```go
package hybrid

import (
    "sort"
)

// RankedItem 表示一个排序结果项
type RankedItem struct {
    ID    int64
    Score float64 // 原始分数 (BM25 分数或余弦相似度)
    Extra map[string]interface{}
}

// RRFResult 表示 RRF 融合后的结果
type RRFResult struct {
    ID         int64
    RRFScore   float64
    BM25Rank   int    // 在 BM25 列表中的排名 (0 表示未出现)
    VectorRank int    // 在向量列表中的排名 (0 表示未出现)
    Extra      map[string]interface{}
}

// RRF 融合搜索
// bm25Results: BM25 搜索结果列表 (已按分数降序排列)
// vectorResults: 向量搜索结果列表 (已按相似度降序排列)
// k: RRF 平滑常数，推荐 60
// topK: 返回结果数
func ReciprocalRankFusion(
    bm25Results []RankedItem,
    vectorResults []RankedItem,
    k float64,
    topK int,
) []RRFResult {
    
    scores := make(map[int64]*RRFResult)
    
    // 处理 BM25 排序列表
    for rank, item := range bm25Results {
        if scores[item.ID] == nil {
            scores[item.ID] = &RRFResult{
                ID:    item.ID,
                Extra: item.Extra,
            }
        }
        // rank 是 0-based，转换为 1-based
        rrfRank := float64(rank + 1)
        scores[item.ID].RRFScore += 1.0 / (k + rrfRank)
        scores[item.ID].BM25Rank = rank + 1
    }
    
    // 处理向量搜索排序列表
    for rank, item := range vectorResults {
        if scores[item.ID] == nil {
            scores[item.ID] = &RRFResult{
                ID:    item.ID,
                Extra: item.Extra,
            }
        }
        rrfRank := float64(rank + 1)
        scores[item.ID].RRFScore += 1.0 / (k + rrfRank)
        scores[item.ID].VectorRank = rank + 1
    }
    
    // 转为切片并排序
    results := make([]RRFResult, 0, len(scores))
    for _, r := range scores {
        results = append(results, *r)
    }
    
    sort.Slice(results, func(i, j int) bool {
        return results[i].RRFScore > results[j].RRFScore
    })
    
    if topK > len(results) {
        topK = len(results)
    }
    return results[:topK]
}
```

### 4.4 加权 RRF 变体

```go
// WeightedRRF 支持对不同搜索通道赋不同权重
// weights[i] 对应 rankLists[i] 的权重
func WeightedRRF(
    rankLists [][]RankedItem,
    weights []float64,
    k float64,
    topK int,
) []RRFResult {
    scores := make(map[int64]*RRFResult)
    
    for listIdx, list := range rankLists {
        weight := 1.0
        if listIdx < len(weights) {
            weight = weights[listIdx]
        }
        
        for rank, item := range list {
            if scores[item.ID] == nil {
                scores[item.ID] = &RRFResult{
                    ID:    item.ID,
                    Extra: item.Extra,
                }
            }
            rrfRank := float64(rank + 1)
            scores[item.ID].RRFScore += weight / (k + rrfRank)
        }
    }
    
    results := make([]RRFResult, 0, len(scores))
    for _, r := range scores {
        results = append(results, *r)
    }
    
    sort.Slice(results, func(i, j int) bool {
        return results[i].RRFScore > results[j].RRFScore
    })
    
    if topK > len(results) {
        topK = len(results)
    }
    return results[:topK]
}
```

### 4.5 完整的混合搜索流程

```go
package search

import (
    "context"
    "database/sql"
)

type HybridSearcher struct {
    db          *sql.DB
    vectorStore *VectorStore
    segmenter   *Segmenter  // 中文分词器
}

// HybridSearch 执行 BM25 + 向量 RRF 融合搜索
func (hs *HybridSearcher) HybridSearch(
    ctx context.Context,
    queryText string,
    queryVector []float32,
    topK int,
) ([]RRFResult, error) {
    
    // 1. 对查询文本进行中文分词
    tokenizedQuery := hs.segmenter.Cut(queryText)
    
    // 2. BM25 搜索 (通过 FTS5)
    bm25Rows, err := hs.db.QueryContext(ctx, `
        SELECT rowid, content, -bm25(docs_fts) as score
        FROM docs_fts
        WHERE docs_fts MATCH ?
        ORDER BY score DESC
        LIMIT ?
    `, tokenizedQuery, topK*2)  // 多取一些用于融合
    if err != nil {
        return nil, err
    }
    defer bm25Rows.Close()
    
    var bm25Results []RankedItem
    for bm25Rows.Next() {
        var id int64
        var content string
        var score float64
        bm25Rows.Scan(&id, &content, &score)
        bm25Results = append(bm25Results, RankedItem{
            ID:    id,
            Score: score,
        })
    }
    
    // 3. 向量搜索
    vectorResults, err := hs.vectorStore.Search(queryVector, topK*2)
    if err != nil {
        return nil, err
    }
    
    // 转换为 RankedItem 格式
    var vecRanked []RankedItem
    for _, vr := range vectorResults {
        vecRanked = append(vecRanked, RankedItem{
            ID:    vr.ID,
            Score: float64(vr.Score),
        })
    }
    
    // 4. RRF 融合
    rrfResults := ReciprocalRankFusion(bm25Results, vecRanked, 60.0, topK)
    
    // 5. 可选: 根据融合结果 ID 回查完整文档内容
    return rrfResults, nil
}
```

### 4.6 Alpha 加权融合 (替代方案)

除了 RRF，还可以使用线性加权融合 (Pinecone 文章中介绍的方法):

```go
// AlphaWeightedFusion 线性加权融合
// score(d) = α × normalize(bm25_score(d)) + (1-α) × normalize(cosine_score(d))
// 需要先将不同打分系统的分数归一化到 [0, 1]
func AlphaWeightedFusion(
    bm25Results []RankedItem,
    vectorResults []RankedItem,
    alpha float64,  // 0 = 纯向量, 1 = 纯BM25, 0.5 = 均衡
    topK int,
) []RRFResult {
    // 归一化 BM25 分数
    bm25Norm := normalizeScores(bm25Results)
    // 归一化向量分数
    vecNorm := normalizeScores(vectorResults)
    
    scores := make(map[int64]float64)
    extra := make(map[int64]map[string]interface{})
    
    for _, item := range bm25Norm {
        scores[item.ID] += alpha * item.Score
        extra[item.ID] = item.Extra
    }
    for _, item := range vecNorm {
        scores[item.ID] += (1 - alpha) * item.Score
        if extra[item.ID] == nil {
            extra[item.ID] = item.Extra
        }
    }
    
    results := make([]RRFResult, 0, len(scores))
    for id, score := range scores {
        results = append(results, RRFResult{
            ID:       id,
            RRFScore: score,
            Extra:    extra[id],
        })
    }
    
    sort.Slice(results, func(i, j int) bool {
        return results[i].RRFScore > results[j].RRFScore
    })
    
    if topK > len(results) {
        topK = len(results)
    }
    return results[:topK]
}

// Min-Max 归一化
func normalizeScores(items []RankedItem) []RankedItem {
    if len(items) == 0 {
        return items
    }
    minScore := items[0].Score
    maxScore := items[0].Score
    for _, item := range items {
        if item.Score < minScore {
            minScore = item.Score
        }
        if item.Score > maxScore {
            maxScore = item.Score
        }
    }
    rangeScore := maxScore - minScore
    if rangeScore == 0 {
        rangeScore = 1
    }
    result := make([]RankedItem, len(items))
    for i, item := range items {
        result[i] = RankedItem{
            ID:    item.ID,
            Score: (item.Score - minScore) / rangeScore,
            Extra: item.Extra,
        }
    }
    return result
}
```

---

## 5. 架构总结

### 5.1 推荐技术栈

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| SQLite 驱动 | `modernc.org/sqlite` v1.55+ | 纯 Go，CGO-free，内置 sqlite-vec |
| 全文搜索 | SQLite FTS5 + `unicode61` | 内置 BM25 打分 |
| 中文分词 | `github.com/yanyiwu/gojieba` | 结巴分词 Go 绑定，预分词方案 |
| 向量搜索 | 纯 Go 暴力搜索 / sqlite-vec | 小规模用暴力搜索，大规模用 sqlite-vec |
| 融合算法 | RRF (k=60) | 简单有效，无需分数校准 |
| Embedding | OpenAI API / 本地模型 | 生成查询和文档向量 |

### 5.2 数据库 Schema 设计

```sql
-- 1. 原始文档表
CREATE TABLE documents (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    metadata TEXT,  -- JSON 格式
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 2. FTS5 全文索引表 (存储分词后的内容)
CREATE VIRTUAL TABLE documents_fts USING fts5(
    title,
    content,
    tokenize = 'unicode61',
    content = 'documents',
    content_rowid = 'id'
);

-- 3. 向量表 (存储 Embedding 向量)
CREATE TABLE document_vectors (
    doc_id INTEGER PRIMARY KEY REFERENCES documents(id),
    embedding BLOB NOT NULL,  -- 序列化的 float32 数组
    model TEXT NOT NULL,      -- 记录使用的模型
    dim INTEGER NOT NULL      -- 向量维度
);

-- 4. 触发器: 自动同步 FTS 索引
CREATE TRIGGER documents_ai AFTER INSERT ON documents BEGIN
    INSERT INTO documents_fts(rowid, title, content) 
    VALUES (new.id, tokenize_cn(new.title), tokenize_cn(new.content));
END;

CREATE TRIGGER documents_ad AFTER DELETE ON documents BEGIN
    INSERT INTO documents_fts(documents_fts, rowid, title, content) 
    VALUES ('delete', old.id, old.title, old.content);
END;

CREATE TRIGGER documents_au AFTER UPDATE ON documents BEGIN
    INSERT INTO documents_fts(documents_fts, rowid, title, content) 
    VALUES ('delete', old.id, old.title, old.content);
    INSERT INTO documents_fts(rowid, title, content) 
    VALUES (new.id, tokenize_cn(new.title), tokenize_cn(new.content));
END;
```

### 5.3 搜索流程

```
用户查询
   │
   ├──→ [中文分词] ──→ [FTS5 BM25 搜索] ──→ BM25 排序列表
   │                                          │
   ├──→ [Embedding 模型] ──→ [向量搜索] ──→ 向量排序列表
   │                                          │
   └──────────────────────────────────────────┘
                      │
                [RRF 融合 (k=60)]
                      │
              ┌───────┴───────┐
              │  融合排序列表  │
              └───────┬───────┘
                      │
                [返回 Top-K 结果]
```

---

## 6. 关键参考来源

1. **modernc.org/sqlite 官方文档**: https://pkg.go.dev/modernc.org/sqlite (v1.55.0, SQLite 3.53.3)
2. **FTS5 中文分词讨论**: https://stackoverflow.com/questions/52422437/ (unicode61 不支持 CJK)
3. **sqlite-icu-tokenizer**: https://github.com/tkys/sqlite-icu-tokenizer (ICU 分词扩展)
4. **Pinecone 混合搜索指南**: https://www.pinecone.io/learn/hybrid-search-intro/ (Alpha 加权融合方法)
5. **RRF 原始论文**: Cormack, G.V., Clarke, C.L.A., & Büttcher, S. (2009). "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods." SIGIR 2009.
6. **SQLite FTS5 文档**: https://sqlite.org/fts5.html (BM25, 分词器配置)
7. **gojieba 分词**: https://github.com/yanyiwu/gojieba (结巴分词 Go 绑定)
