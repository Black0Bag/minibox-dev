# minibox-APP 开发备忘录

> 本文档仅存本地，**不推送 GitHub**。记录 minibox 2.0 前端 APP 开发过程中与用户商榷的所有想法、结论与待定事项。
> 起始日期：2026-08-01

---

## 一、商榷记录

### 2026-08-01

---

#### 1. 前端 APP 定位（已确认）

- minibox 2.0 前端 = **纯前端客户端**，AI 能力全部由后端（minibox Go 服务）提供
- 技术路线：Jetpack Compose + Material 3 Expressive（替换小万的 Flutter）
- 架构分层：Presentation（Compose UI）/ Domain / Data（Retrofit + SSE + Room）

---

#### 2. 设备代理「眼耳口手」（已确认方案）

- **用户想法**：前端 APP 保留基础硬件操控能力，作为后端服务的眼耳口手，通过协议互通，让后端能自动操作手机
- **结论**：完全可行（小万 vlm_task 已验证技术）
- **架构**：后端=大脑（Agent+视觉LLM），前端=眼耳口手（执行器），WebSocket 长连接（:8086）
- **能力矩阵 18 项**：
  - 👁 眼：截屏(MediaProjection) / UI层级(Accessibility) / 通知(NotificationListener) / 剪贴板 / 设备信息 / 前台应用
  - 🖐 手：点击 / 滑动 / 长按 / 输入文字 / 按键 / 打开应用 / 滚动查找（dispatchGesture）
  - 👄 口：TTS 朗读 / Toast / 本地通知
  - 👂 耳：语音听写(SpeechRecognizer) / 前台监听
- **协议**：WebSocket + JSON-RPC 2.0（command/result/event/心跳/配对码）
- **自动操作闭环**：截图 → 视觉LLM分析 → 点击 → 再截图验证 → 循环直到完成
- **后端对接**：新增 internal/device/（Hub+注册+配对+审计）+ 18 个 MCP 设备_* 工具 + /api/v1/device/* 路由
- **安全**：配对码 / 权限分级 / 危险操作 HITL 确认 / 截图敏感区模糊 / Keystore 令牌 / 双端审计 / 心跳失控保护

---

#### 3. 前后端文档分离（已执行）

- **决策**：minibox 项目（含 GitHub 云端）只保留后端内容；所有前端文档移至工作根目录独立管理
- **前端设计文档目录**：`/workspace/minibox-前端设计/`

| 文档 | 新路径 | 云端状态 |
|---|---|---|
| 前端架构规划报告 | `/workspace/minibox-前端设计/前端架构规划报告.md` | ✅ 已从 GitHub 移除 |
| 前端深度解耦与架构优化 | `/workspace/minibox-前端设计/前端深度解耦与架构优化.md` | 从未推送 |
| 前端解耦与架构优化方案 | `/workspace/minibox-前端设计/前端解耦与架构优化方案.md` | ✅ 已从 GitHub 移除 |
| 设备代理方案 | `/workspace/minibox-前端设计/设备代理方案.md` | ✅ 已从 GitHub 移除 |
| **本备忘录** | `/workspace/minibox-前端设计/minibox-APP开发备忘录.md` | 🔒 仅本地 |

- minibox 仓库 docs/ 现仅保留：`后端使用说明.md`（后端契约）
- 后续商榷内容**只写本备忘录**（本地），不再推送 GitHub

---

#### 4. 聊天窗口「历史消息后撤（rewind）」（已确认 4 项决策）

**用户需求**：
- 小万现状：只能编辑最后一条用户输入（`_canEditUserMessage` → `_isLatestUserMessage` 硬限制）
- 想要：像 OpenCode 那样可对**历史任意一条**用户输入后撤
- 后撤到指定条目后，把**之后的对话从上下文摘离**
- **保留两轮对话缓存**留有后悔余地
- 第二轮新对话后 → **彻底移除**被覆盖的历史记录

**4 项决策确认**：
1. 后撤入口：**OpenCode 式悬停操作栏**（长按消息出现行内操作栏，非弹出菜单）
2. 恢复时机：**保留两轮对话的后悔窗口**——后撤后可恢复，第一轮新对话后仍可恢复，第二轮新对话后才彻底删除
3. 自动回填：**是**，后撤时自动将该消息回填到输入框
4. 工具调用记录：**一起隐藏**，保证上下文干净

**消息三态 + 两轮后悔**：
| 状态 | 含义 | 可见性 |
|---|---|---|
| `normal` | 正常消息 | 正常显示 |
| `rewound` | 被后撤摘离（已移出上下文） | 折叠隐藏 + 分隔条提示 |
| `deleted` | 彻底删除（第二轮新对话已产生） | 不可见 |

后悔窗口生命周期：
```
后撤 → rewound（保留 snapshot_1）
  ├─ 恢复 → rewound 还原为 normal（后悔窗口关闭，snapshot_1 清空）
  ├─ 第一轮新对话 → 摘离段仍保留（snapshot_1 → snapshot_2）
  │   ├─ 恢复 → 回到后撤前原始状态（一轮后悔用完）
  │   └─ 第二轮新对话 → 摘离段彻底删除 → 后悔窗口抹除
  └─ 不操作 → 悬浮后悔窗口持续可见
```

**交互流程**：
```
① 长按任意用户消息 → 行内操作栏浮现（OpenCode 风格）
   [复制] [✂️ 后撤到此] [🔄 重新生成]

② 点击「后撤到此」
   - 之后所有消息（含 AI 回复 + 工具调用记录）→ 标记 rewound（折叠淡出）
   - 该消息自动回填到输入框，光标聚焦可编辑
   - 顶部悬浮条：「已回退，之后 N 条已移出上下文 · 剩 2 次恢复机会 [↩ 恢复]」

③ 第一轮新对话发送
   - 摘离段仍保留缓存
   - 悬浮条更新：「已从后撤点继续 · 剩 1 次恢复机会 [↩ 恢复]」

④ 第二轮新对话发送
   - 摘离段彻底删除 → 后悔窗口抹除 → 悬浮条消失

⑤ 任意时刻点击「恢复」
   - 第一次恢复：回到后撤前原始状态（摘离段还原）
   - 恢复后后悔窗口关闭，不可再次后撤恢复同一段
```

**数据模型（Room）**：
```sql
ALTER TABLE messages ADD COLUMN status TEXT DEFAULT 'normal';   -- normal/rewound/deleted
ALTER TABLE messages ADD COLUMN rewound_from TEXT;
ALTER TABLE messages ADD COLUMN rewound_at INTEGER;

ALTER TABLE conversations ADD COLUMN rewind_point_message_id TEXT;
ALTER TABLE conversations ADD COLUMN rewind_snapshot_1 TEXT;        -- 第一轮摘离段 JSON
ALTER TABLE conversations ADD COLUMN rewind_snapshot_2 TEXT;        -- 第二轮摘离段 JSON
ALTER TABLE conversations ADD COLUMN rewind_recovery_count INTEGER DEFAULT 0;
ALTER TABLE conversations ADD COLUMN rewind_new_dialog_count INTEGER DEFAULT 0;
```

**与后端协作**：
- 后撤时仅前端操作（本地状态），**不通知后端**
- 发送新消息时：`POST /conversations/{id}/messages` 携带 `rewind_to_message_id` 参数
- ⚠️ 需后端新增支持：截断上下文从后撤点重新生成

**UI/动画**：
- 长按气泡 → 行内操作栏浮现（M3 波纹 + 弹性缩放）
- 后撤折叠：高度收缩 + 淡出（~300ms），工具卡片一起折叠
- 恢复：展开 + 淡入
- 悬浮条：Liquid Glass 毛玻璃，两轮后自动消失（~200ms 淡出）
- 分隔条：后撤点显示「✂️ 以下内容已移出上下文」虚线分隔

---

#### 5. 聊天界面微调（已确认）

##### 5a. 移除对话定位悬浮窗 ❌
- 小万右下角对话条目定位悬浮窗，用户完全不使用 → **直接移除**

##### 5b. 新增 TodoList 竖条悬浮窗

- 位置：对话界面**右侧居中**竖向细长条悬浮窗
- 无 list 时 → **隐藏**（Gone 不占位）
- 后端调用 list 功能 → **激活显示**，从右侧滑入（~300ms spring）
- 里面只有和 list 数量对应的**空方框**（☐），完成一个 → 变成**有对号方框**（☑）
- 尽可能窄（~36-40dp），少占对话面积
- 点击 → **向左展开**显示文字（毛玻璃面板，最长 200dp，~250ms spring）

**规格**：
- 方框 24×24dp，圆角 6dp，间距 8dp
- 未完成 = M3 `outlineVariant`；已完成 = M3 `primary`
- 对号 Material Symbols `check`，描边动画 ~250ms
- 无 list：width→0 + alpha→0（~200ms 淡出）

**数据流**：
- ⚠️ 需后端新增 SSE `todo_update` 事件：`{ "items": [{"id":1, "text": "...", "done": false}] }`
- 收到 → 激活竖条 → 渲染方框 → 状态变化时 ☐→☑ 动画

---

#### 6. 浏览器代理：前端 APP 内嵌浏览器分担后端硬件限制（已确认方案）

- **用户想法**：后端在路由器上无法集成浏览器引擎，把浏览器集成进手机 APP 前端
- **结论**：完全可行（小万已有 browser_use 工具链）
- **架构**：前端内嵌 Android WebView，作为后端 Agent 的浏览器代理
- **复用设备代理同一 WebSocket**（:8086），method 前缀 `浏览器_*`
- **16 个 MCP 浏览器工具**：导航/截图/点击/输入/提取文本/提取骨架/滚动/执行JS/获取Cookie/设置UA/标签管理/前进后退/按键/等待元素/网络拦截/收集滚动
- **优势**：手机 8GB RAM + GPU 硬件渲染 vs 路由器跑不动 Chrome；真实 Android Chrome UA 不易被风控；手机已登录网站天然带 Cookie；用户可手动处理验证码

**安全**：禁文件访问/白名单 URL/JS 签名验证/Cookie 脱敏/风控 HITL

---

#### 7. 浏览器代理技术选型优化（已确认采用方案 A）

**用户痛点**：
- 小万浏览器**卡顿**：JS 异步回调无流控，大页面阻塞
- **不打开浮窗就不执行**：WebView 不可见时系统降级渲染 → JS 暂停 → 指令积压 → Agent 超时报错

**5 条路线对比后推荐方案 A：WebView + CDP LocalSocket**

| 痛点 | 小万（纯 JS 注入） | 方案 A（CDP） |
|---|---|---|
| 卡顿 | JS 异步回调阻塞 | CDP WebSocket 二进制流式 |
| 不可见不执行 | WebView INVISIBLE → 系统暂停渲染 | Headless Service 独立保活，不依赖 UI |
| 截图 | View.draw 模糊 | Page.captureScreenshot 原生 |
| 网络拦截 | 不支持 | Fetch.requestIntercepted |
| Cookie | 有限 | Network.getCookies 完整 |

**关键实现**：
- Headless WebView Service（独立 Service，INVISIBLE 但活跃）
- `setWebContentsDebuggingEnabled(true)` 开启 CDP
- LocalSocket 连 `chrome_devtools_remote` → WebSocket 握手 → CDP 通道
- 后端 MCP 工具 → CDP 命令映射（Page.navigate / Page.captureScreenshot / Input.dispatchMouseEvent 等 16+）
- 权限：无需 root/ADB，只需 INTERNET + 前台 Service 通知

---

#### 8. 聊天界面三条 UI 细节优化（已确认）

##### 8a. 右上角 Plan / Build 双模式切换

| 模式 | 图标 | 配色主题 | 后端对应 |
|---|---|---|---|
| **Plan（规划）** | `lightbulb` 灯泡 | 冷色调（蓝/青/灰）平静理性 | Agent 仅思考+工具预演 |
| **Build（执行）** | `build` 锤子 | 暖色调（橙/琥珀/红）活跃炽烈 | Agent 思考+工具实执行 |

- 点击图标 → M3 SplitButtonLayout 切换菜单
- 切换动画：图标 Morph（lightbulb↔build ~300ms）+ 全局配色渐变（~500ms）
- 当前模式脉冲呼吸光效（Plan 冷蓝 / Build 暖橙）
- ⚠️ 需后端支持会话级 `mode` 字段

##### 8b. 顶部条状浮窗功能对接

```
[SSH: 192.168.x.x ✓]  [🌐 浏览器]  [📋 Plan/Build]  [⚙️]
```

- **SSH**：接入后端服务所在服务器（设置里配置主机/端口/认证），实现库 JSch/sshj，前端本地功能
- **环境变量**：❌ 完全删除
- **浏览器**：常驻显示，用户主动控制（输入 URL/复制/导航/截图），Agent 操作时浮窗自动更新，用户操作优先级 > Agent
- ⚠️ 需 SSH 连接配置设置页

##### 8b 补充：浏览器插件可行性

- Android WebView **不原生支持 Chrome 扩展**（缺 chrome.* API / service workers / manifest 解析器）
- **替代方案**：CDP `Page.addScriptToEvaluateOnNewDocument` 每页面加载前自动注入 JS → **兼容油猴脚本（userscript）生态**
- DOM 操作 / CSS 注入 / 请求拦截 / Cookie 管理与扩展无差距
- 缺右键菜单和弹出窗口 → 需自建
- **结论：不支持 .crx 但可完美兼容油猴脚本，对 AI 浏览器代理场景已足够**

##### 8c. 左滑 → 后端服务器文件管理（SFTP）

| 维度 | 小万 | minibox 2.0 |
|---|---|---|
| 左滑目标 | 本地 proot 工作目录 | SFTP 连后端服务器文件系统 |
| 协议 | 本地文件 I/O | SFTP / SCP |
| 连接 | 不需要 | 复用 8b SSH 配置 |

- 双通道：SFTP 直连（快）+ 后端 API 文件操作（安全审计）
- 支持：浏览/下载/上传/删除/重命名/编辑文本文件
- ⚠️ 需设置页新增「服务器文件管理」配置

---

#### 9. 文件长文本读取截断问题 → 后端承载文件读取（已确认可行）

**痛点**：小万手机端直接读文件受内存/message buffer 限制静默截断 → LLM 理解错误

**方案**：文件读取全部交给后端，前端不直接读文件内容
- 后端服务器文件 → `GET /api/v1/workspace/files/:path`（后端直读完整）
- APP 本地文件 → SFTP 上传后端 tmp/ → 后端完整读取
- 超大文件 → 后端智能分片（token 预算/语义段落）
- **截断防护**：后端返回 total_bytes/total_lines 元数据，前端始终显示，绝不静默截断

---

#### 10. 右滑侧边栏重构：服务器资源监控 + 历史对话下移（已确认）

**布局**：
```
┌───────────────────────────────┐
│  服务器资源监控（新增 ~120dp）    │
│  ┌─────┐ ┌─────┐ ┌─────┐      │
│  │ CPU │ │ RAM │ │磁盘 │      │
│  │ 45% │ │ 62% │ │ 78% │      │
│  │ ◐━  │ │ ◐━  │ │ ◐━  │      │
│  └─────┘ └─────┘ └─────┘      │
│  192.168.x.x · N1 · 在线 ✓    │
├───────────────────────────────┤
│  历史对话（整体下移）           │
│  🔍 搜索...                    │
│  📌 会话列表...                │
├───────────────────────────────┤
│  ⚙️ 设置 │ 📋 模式 │ ℹ️ 关于   │
└───────────────────────────────┘
```

**圆形弧线进度条**：
- 270° 开口圆弧（Apple Watch Activity Ring 风格），底部 90° 缺口
- 颜色编码：0-60% 绿 / 61-80% 琥珀 / 81-100% 红，平滑过渡 ~400ms
- 中心百分比 bold 24sp，弧线 6dp，组件 64×64dp，每秒刷新
- 离线：全灰 + "离线" 文字 + 灰色心跳

**后端数据来源**：
- ⚠️ 需后端新增 `GET /api/v1/system/stats` + SSE `system_stats` 事件每秒推送

**主题联动**：监控区背景板色调随 Plan/Build 模式，圆弧颜色保持绿/琥珀/红语义不变

---

#### 11. 「关于」页面：检查更新 + 国内镜像下载源（已确认）

**检查更新**：
- `GET https://api.github.com/repos/Black0Bag/minibox-android/releases/latest`（无需认证，60 次/小时）
- 解析 tag_name / body（更新日志 Markdown） / assets（APK 文件）
- 与本地 `BuildConfig.VERSION_NAME` 对比 → 有新版弹更新弹窗

**下载源 4 个镜像**：

| # | 下载源 | 说明 |
|---|---|---|
| 1 | GitHub Releases（默认） | 官方源，国内可能慢 |
| 2 | ghproxy 加速 | `https://ghproxy.com/https://github.com/...` |
| 3 | gh.ddlc.top 加速 | 备选代理 |
| 4 | mirror.ghproxy.com 加速 | 备选代理 |

**更新弹窗**：版本号 + 更新日志 + 下载源选择（4 选 1 或自动测速）+ 文件名大小 + 下载安装按钮

**设置页新增**：
```
设置 > 关于
├── 当前版本
├── 检查更新（点击触发）
├── 默认下载源选择（4 个 + 自动测速）
├── 自动检查更新开关
├── 更新检查频率（每次启动 / 每天 / 手动）
└── 更新历史记录
```

- 自动测速：并行 HEAD 各镜像取最快 TTFB，3 秒超时回退 GitHub，缓存 24 小时
- 下载安装：通知栏进度 → `ACTION_INSTALL_PACKAGE` → 用户确认
- 权限：`REQUEST_INSTALL_PACKAGES`
- 与后端无关：纯前端 APP 自身更新机制

---

## 二、需后端新增的支持（汇总）

| # | 需求 | 涉及条目 |
|---|---|---|
| 1 | `POST /conversations/{id}/messages` 新增 `rewind_to_message_id` 参数 | #4 rewind |
| 2 | `POST /conversations/{id}/messages` 新增 `mode` 字段（plan/build） | #8a 模式切换 |
| 3 | SSE 新增 `todo_update` 事件（to-do-list 状态推送） | #5b TodoList |
| 4 | SSE 新增 `system_stats` 事件（每秒推送 CPU/RAM/磁盘） | #10 服务器监控 |
| 5 | `GET /api/v1/system/stats` 端点 | #10 服务器监控 |
| 6 | MCP 注册 18 个 `设备_*` 工具 + WebSocket Hub（:8086） | #2 设备代理 |
| 7 | MCP 注册 16 个 `浏览器_*` 工具（复用 :8086） | #6 #7 浏览器代理 |
| 8 | `GET /api/v1/workspace/files/:path` 支持 `?full=true` 完整读取 | #9 文件截断 |

---

## 三、待商榷事项（开放问题）

- [ ] 设备代理：权限引导交互细节（逐次确认 vs 一次性授权）
- [ ] 设备代理：截图回传的压缩/模糊策略（隐私）
- [ ] 设备代理：多设备管理 UI（后端侧）
- [ ] 设备代理：HITL 确认弹窗的触发规则
- [ ] 聊天界面：流式渲染动画偏好
- [ ] 悬浮球/宠物猫：是否保留、形态
- [ ] 主题：深色默认 vs 浅色默认
- [ ] 设置页：后端服务设置组的具体交互

---

## 四、备注

- 所有后续想法先在备忘录记录，逐条商榷确认后再进入开发
- 不推送 GitHub 是用户的明确要求（2026-08-01）
- 当前处于**前端设计阶段**


#### 12. Subagent 浏览器隔离：多标签页互不争抢（2026-08-01，实际开发中发现）

**问题来源**：本对话中并行分派 6 个搜索 subagent，其中 2 个因浏览器争抢导致读取错误失败。根因：所有 subagent 共享同一个浏览器上下文，互相导航覆盖对方的页面。

**用户需求**：
- 必须利用浏览器的多窗口/多标签页功能区
- 规定每个 subagent 使用自己的 ID 只能控制自己创建的浏览器窗口
- 做到浏览器多窗口，subagent 互相隔离

**设计方案**：

**标签页分配机制**：
```
subagent_dispatch
  ├─ subagent-001 → browser.new_tab() → tab_id=1 → 只操作 tab_id=1
  ├─ subagent-002 → browser.new_tab() → tab_id=2 → 只操作 tab_id=2
  ├─ subagent-003 → browser.new_tab() → tab_id=3 → 只操作 tab_id=3
  └─ ...（最多并发 6 个 = 最多 6 个标签页）
```

**隔离规则**：
- 每个 subagent 首次调用 `browser_use` 时自动创建一个独立标签页（`new_tab`）
- 后续所有操作（navigate/screenshot/click/type/get_text 等）自动作用在该 subagent 专属 `tab_id` 上
- subagent 无法访问或操作其他 subagent 的标签页
- subagent 任务完成后自动关闭其标签页（`close_tab`）

**与 Omnibot 现有实现对照**：
- Omnibot `browser_use` 已支持 `tab_id` 参数和 `new_tab`/`close_tab`/`list_tabs` 操作，最多 3 个标签页
- 当前限制：最多 3 个标签页 → 并行 subagent 最多 3 个可使用浏览器
- 超过 3 个的 subagent 需排队等待空闲标签页，或退化为不支持浏览器的能力

**参数设计**：
- `subagent_dispatch` 新增可选参数 `browserTabs: int`：指定可分配给浏览器的标签页数量（默认 3，上限 6）
- 每个 subagent 的 `browser_use` 调用自动注入其专属 `tab_id`
- subagent 框架在分配任务时为每个 subagent 预分配标签页

**对 minibox-APP 的启示**：
- minibox 后端 Go 服务如果也支持 subagent 并行，需在设计之初就实现浏览器实例隔离
- 建议后端浏览器代理模块使用 **per-agent browser context**（类似 Playwright 的 BrowserContext）
- 每个 agent 分配独立的 BrowserContext（独立 cookies/storage/标签页），互不干扰


#### 13. 用户确认事项（2026-08-01 下午）

1. **宠物猫功能移除**：minibox 不保留小万的宠物猫功能 ✅
2. **默认深色主题**：APP 首次启动即为深色模式，Plan（冷色调）/ Build（暖色调）两套深色配色 ✅
3. **流式渲染**：待用户从 typewriter 逐字 vs 滑入揭示 中选择（已向用户解释）
4. **minSdk 26**（Android 8.0+）覆盖 95%+ 设备 ✅

**追加确认：构建方式**
- minibox-android 从空工程开始，用 Kotlin + Jetpack Compose 全新编写
- 小万源码（Flutter/Dart）不直接复用代码，仅作为 UI 交互和 API 调用模式的设计参考
- 语言不同（Dart → Kotlin）、框架不同（Flutter → Compose），无法复用代码
