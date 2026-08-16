# minibox 开发文档中心

> 一个以 **Go 后端为中枢大脑、安卓 APP 为眼耳口手** 的私有化强 Agent 系统。
> 本仓库是 **minibox 项目的文档中心**——所有设计资料、技术调研、路线图、注意事项的统一存放地，供在任何设备上浏览、分析与继续编辑。

---

## 🧭 从这里开始（阅读顺序）

| 顺序 | 文档 | 作用 |
|---|---|---|
| 1 | **[01_项目总路线图.md](./01_项目总路线图.md)** | **最高基准**：项目定位、总体架构、四大核心理念、已定决策、关键技术选型、全局红线 |
| 2 | **[02_后端开发路线图.md](./02_后端开发路线图.md)** | 后端（Go 中枢）逐 Phase 开发计划、技术决策速查、踩坑红线 |
| 3 | **[03_前端开发路线图.md](./03_前端开发路线图.md)** | 前端（安卓 APP）逐 Phase 开发计划、UI 交互决策、注意事项 |
| 4 | 各子目录 `00_README.md` | 对应分类下的设计资料与研究报告索引 |

> 想了解某方面的**设计过程与原始讨论**（而非结论）？去 `系统设计/`、`后端/`、`前端/` 目录看对应原始文档。

---

## 🗂 文档地图

```
minibox-dev/                          ← 本仓库（文档中心）
├── README.md                         ← 本文件（总览 + 核心注意事项）
├── 01_项目总路线图.md                ← 最高基准（纯净版，从这里读起）
├── 02_后端开发路线图.md              ← 后端细化计划
├── 03_前端开发路线图.md              ← 前端细化计划
│
├── 系统设计/                         ← 系统级设计资料
│   ├── 00_README.md
│   ├── 01_需求梳理与路线图.md        ← 原始需求梳理（含全部历史决策过程与滚动追加）
│   ├── 02_团队协作系统设计.md        ← 「服务型公司」项目模式专项设计
│   └── 03_设备代理方案.md            ← 前端 APP 作为后端眼耳口手（审计概念源头）
│
├── 后端/                             ← 后端设计资料
│   ├── 00_README.md
│   ├── 01_开发备忘录.md              ← 后端内核需求基准
│   ├── 02_开发参考.md                ← 架构蓝图与代码级参考
│   ├── 03_开发经验.md                ← 踩坑记录（7 大坑/五重守卫）
│   ├── 04_后端路线图_原始.md         ← 原始后端路线图（T/N 系列论证过程）
│   └── 技术研究/                     ← 8 份后端技术调研
│       ├── 00_README.md
│       ├── go-agent-frameworks-research.md
│       ├── go-multi-provider-llm-research.md
│       ├── go-sqlite-fts5-vector-search-research.md
│       ├── go-sse-http-api-best-practices.md
│       ├── go-self-update-research.md
│       ├── agent_loop_research.md
│       ├── ai-agent-frameworks-research.md
│       └── ai-agent-memory-systems-research.md
│
└── 前端/                             ← 前端设计资料
    ├── 00_README.md
    ├── 01_APP开发备忘录.md           ← 前端需求记录
    ├── 02_APP开发参考文件.md         ← 前端技术实现参考
    ├── 03_APP开发流程List.md         ← 前端开发执行清单
    ├── 04_前端架构规划报告.md        ← 前端架构规划
    ├── 05_前端解耦与架构优化方案.md  ← 前后端解耦方案
    ├── 06_前端深度解耦与架构优化.md  ← 源码级解耦分析
    ├── 07_小万能力参考文档.md        ← 能力对标参考
    ├── 08_前端路线图_原始.md         ← 原始前端路线图
    └── 技术研究/                     ← 3 份前端技术调研
        ├── 00_README.md
        ├── android-app-auto-update-github-releases.md
        ├── android-server-monitoring-dashboard-best-practices.md
        └── compose-drawer-floating-overlay-sftp.md
```

---

## ⚠️ 核心注意事项（全局红线速览）

1. **中文优先**：所有非机器语言一律简体中文。
2. **零 CGO 铁律**：后端所有依赖必须纯 Go（交叉编译路由器 arm64 全靠它）。
3. **知识库 = 唯一记忆**：一切信息入库，不另用散装文本文件。
4. **权限三层级**：人类直接操作 > 代码级防火墙 > agent 自主决策；确认按钮只在自家界面。
5. **文档滚动追加原则**：本仓库文档只允许末尾追加带时间戳条目，不删改已有内容。
6. **端口统一 8086**，`/api/v1` 前缀，RFC 7807 错误格式。
7. **前端永不直接访问数据库**（SSH/SFTP 两个直连例外除外）。
8. **TDD 优先**：先失败测试 → 实现 → 通过 → 提交。
9. **编译走 CI**：本地不装 Go/Android 环境，push 后由 GitHub Actions 编译 + 质量检查 + 自动发布 Release。

---

## 📦 关联仓库

| 仓库 | 说明 |
|---|---|
| [`Black0Bag/minibox`](https://github.com/Black0Bag/minibox) | 后端代码（Go 中枢大脑） |
| [`Black0Bag/minibox-android`](https://github.com/Black0Bag/minibox-android) | 前端代码（安卓 APP） |
| [`Black0Bag/minibox-dev`](https://github.com/Black0Bag/minibox-dev) | **本仓库**（文档中心） |
| `Black0Bag/minibox-legacy` / `minibox-android-legacy` | 旧版代码归档（更名保留） |
| `Black0Bag/AndroidLLMRouter` | LLM 路由器参考实现（多渠道/多Key/熔断） |
| `Black0Bag/sage-wiki-plus` | 知识库内核参考（理念借用，代码重写不留痕迹） |

---

## ✍️ 维护约定

- 新增/修改文档：只追加，不删改历史。
- 新增技术调研：放入对应 `技术研究/` 子目录并在 `00_README.md` 登记。
- 路线图更新：先在讨论中定稿，再回写 `01/02/03` 三份纯净路线图。
