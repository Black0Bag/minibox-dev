# 实施计划（Plan）

## 里程碑

1. 后端版本管理对齐：VERSION 文件 + CI 校验 + GoReleaser 集成（已完成 v0.1.0-v0.1.2）
2. 工作区文档治理：Skill 门禁对齐 + 开发链路规则回填（已完成）
3. 后端深度优化打磨：http_handlers.go 拆分 + 4 包补测试（已完成 v0.2.0-v0.2.1）
4. Android 前端工程化：基于稳定后端进行前端开发（待启动）

## 任务拆解

### M2：文档治理（已完成）

- 目标：工作区四文档 + Go 仓库四文档全部通过 ensure_core_docs.py 门禁
- 输入：Vibe-coding-workflow-zh Skill 模板要求 + minibile 开发链路经验
- 输出：全部文档 EXISTS + 无 INVALID

### M3：后端深度优化（已完成）

- 目标：http_handlers.go 拆分、4 包补测试、GoReleaser 发布流程验证
- 输入：后端体检报告（2026-08-30）+ 当前代码实测
- 输出：CI 全绿 + Release v0.2.0/v0.2.1 发布成功

### M4：Android 前端工程化（待启动）

- 目标：Gradle 脚手架 + CI + 首版 APK
- 输入：后端稳定契约（v0.2.1）+ minibile T03A cloud_build_release 方案
- 输出：Android CI 全绿 + 首版 APK Release

## 验收节点

- M2：ensure_core_docs.py 全 EXISTS，无 INVALID ✅
- M3：CI 四阶段全绿，Release v0.2.1 产出跨平台二进制 ✅
- M4：Android CI 全绿，首版 APK 签名发布

## 风险与回滚

- 风险：文档治理中误删有效规则
- 回滚策略：Git 历史恢复
- 风险：后端重构引入回归
- 回滚策略：每个拆分独立提交 + 完整测试门禁
- 风险：前端启动过早导致契约漂移
- 回滚策略：严格在后端 v0.2.1 发布后再启动前端