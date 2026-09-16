# 项目架构（Structure）

## 目录结构

```text
minibox/                     工作区根目录
├── minibox/                 Go 后端源码 + CI/CD + 协议文档
│   ├── cmd/minibox/         程序入口
│   ├── internal/app/        组合根 + 域 handler（10 个域文件）
│   ├── internal/domain/     领域类型和接口
│   ├── internal/infrastructure/  SQLite/LLM/工具/调度
│   ├── internal/transport/   HTTP/SSE/WebSocket
│   ├── .github/workflows/   CI（test/race/lint/audit）+ Release
│   ├── .goreleaser.yaml     跨平台发布配置
│   ├── VERSION              唯一版本源
│   └── CHANGELOG.md         版本变更记录
├── minibox-android/         Android 工程（当前仅文档）
├── minibox-dev/             产品决策和路线图
│   ├── docs/                工作流核心文档（goal/plan/rules/structure）
│   └── dev/                 路线图、系统设计、前端基准
├── minibox-test/            唯一稳定验证实例
└── skills/                  已安装 Skill 集合
```

## 模块划分

- 代码归代码仓库（`minibox/` + `minibox-android/`）
- 产品决策归 Dev（`minibox-dev/`）
- 运行数据归测试目录（`minibox-test/`）
- 总路线图：产品定位、边界和阶段状态
- 后端路线图：维护边界和已知限制
- 前端路线图：Android 可执行顺序
- 系统设计：跨前后端概念，不重复源码 API

## 数据流/调用关系

后端 `docs/`（协议事实）→ Dev 路线图（产品排序）→ Android 仓库（实现与测试）。

版本流：VERSION 文件 → CI 版本校验 → git tag v{VERSION} → GoReleaser 发布 → Release 不可变。

## 依赖与外部接口

- Go 后端：SQLite（modernc.org/sqlite）、OpenAI 兼容 LLM、WebSocket JSON-RPC、SSE。
- Android 前端：REST + SSE + WebSocket 三通道客户端（待实现）。
- CI/CD：GitHub Actions（test/race/lint/audit）+ GoReleaser。
- 本仓库没有运行时依赖或凭据。

## 关键入口文件

- `README.md`
- `docs/goal.md`、`docs/plan.md`、`docs/rules.md`、`docs/structure.md`
- `dev/01_项目总路线图.md`
- `dev/03_前端开发路线图.md`

## 高风险模块

- 文档漂移：旧 API/选型被误当当前事实。
- 隐私：测试配置或日志被复制进文档。
- 产品承诺：把预留方向写成已实现能力。
- 版本管理：VERSION 与 Tag 不一致将导致 CI 失败。
- GoReleaser 发布：已发布 Tag/Release 不可变，内容变化只能推进新版本。