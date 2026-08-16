# minibox-APP 开发参考文件

> 基于互联网全方位搜索整理，覆盖 8 个核心模块的最佳实践与优化方案。
> 创建日期：2026-08-01
> 配套文件：minibox-APP开发备忘录.md（需求设计）、research/ 目录（详细研究报告）

---

## 目录

1. [圆形弧线进度条（Activity Ring）](#1-圆形弧线进度条activity-ring)
2. [对话历史 Rewind（树形分叉 + Undo/Redo）](#2-对话历史-rewind树形分叉--undoredo)
3. [服务器资源监控 Dashboard](#3-服务器资源监控-dashboard)
4. [Material 3 Expressive 设计语言](#4-material-3-expressive-设计语言)
5. [APP 在线更新 + 国内镜像下载](#5-app-在线更新--国内镜像下载)
6. [侧边栏 Drawer + TodoList 浮窗 + SFTP 文件管理](#6-侧边栏-drawer--todolist-浮窗--sftp-文件管理)
7. [浏览器代理方案优化](#7-浏览器代理方案优化)
8. [全局性能与电量优化](#8-全局性能与电量优化)

---

## 1. 圆形弧线进度条（Activity Ring）

> 对应备忘录 #10：右滑侧边栏服务器资源监控

### 1.1 核心 API：Canvas drawArc

```kotlin
drawArc(
    color: Color,          // 或 brush: Brush（渐变）
    startAngle: Float,     // 起始角度，0° = 3点钟方向，顺时针为正
    sweepAngle: Float,     // 扫过角度（正值顺时针）
    useCenter: Boolean,    // false = 纯弧线（进度条用此值）
    topLeft: Offset,       // 外接矩形左上角
    size: Size,            // 外接矩形尺寸
    style: DrawStyle        // Stroke(width, cap, join) 或 Fill
)
```

### 1.2 270° 开口实现

- `startAngle = 135f`（约 7:30 方向），`sweepAngle = 270f * progress`
- 底部正中留 90° 缺口（Apple Watch Activity Ring 风格）
- 圆角端点：`StrokeCap.Round`
- 两层绘制：背景轨道弧（270° 灰色）+ 前景进度弧（progress × 270° 彩色）

### 1.3 颜色渐变（绿→琥珀→红）

**推荐方案：sweepGradient + colorStops**

```kotlin
val brush = Brush.sweepGradient(
    colorStops = arrayOf(
        0.0f to Color(0xFF4CAF50),   // 绿
        0.5f to Color(0xFFFFC107),   // 琥珀
        0.75f to Color(0xFFFF5722),  // 橙红
        1.0f to Color(0xFFF44336)    // 红
    ),
    center = Offset(size.width / 2, size.height / 2)
)
```

**简单方案：Color.lerp 线性插值**（单色弧，颜色随进度变化）

```kotlin
val currentColor = when {
    progress < 0.5f -> lerp(Color.Green, Color(0xFFFFC107), progress * 2f)
    else -> lerp(Color(0xFFFFC107), Color.Red, (progress - 0.5f) * 2f)
}
```

### 1.4 ⚠️ 性能优化（最关键部分）

Compose 三阶段：Composition → Layout → Drawing。**必须在 draw lambda 内部读取动画值，避免每帧重组**。

| 模式 | 触发阶段 | 重组？ | 适用场景 |
|------|---------|--------|---------|
| `Canvas { drawArc(animatedProgress...) }` 顶层用 `by` | Composition | 🔴 是 | **反模式，应避免** |
| `Modifier.drawBehind { … }` | Drawing | 🟢 否 | 简单绘制 |
| `Modifier.drawWithCache { onDrawBehind { … } }` | Drawing | 🟢 否 | **最佳：含 Brush/Path** |

**最佳实践代码模式：**

```kotlin
@Composable
fun ActivityRingProgress(
    progress: Float,        // 0f ~ 1f
    modifier: Modifier = Modifier,
    strokeWidth: Dp = 6.dp,
    startAngle: Float = 135f,
    maxSweep: Float = 270f
) {
    // 1. 动画状态——不用 by 委托！
    val animatedProgress = animateFloatAsState(
        targetValue = progress,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        ),
        label = "ring"
    )

    // 2. drawWithCache 缓存渐变 Brush
    Box(modifier.size(64.dp).drawWithCache {
        val gradientBrush = Brush.sweepGradient(
            colorStops = arrayOf(
                0f to Color(0xFF4CAF50), 0.5f to Color(0xFFFFC107),
                0.75f to Color(0xFFFF5722), 1f to Color(0xFFF44336)
            ),
            center = Offset(size.width / 2, size.height / 2)
        )
        onDrawBehind {
            // 3. 背景轨道
            drawArc(
                color = Color.Gray.copy(alpha = 0.2f),
                startAngle = startAngle, sweepAngle = maxSweep,
                useCenter = false,
                style = Stroke(width = strokeWidth.toPx(), cap = StrokeCap.Round)
            )
            // 4. 前景进度——在此读取 .value，仅触发 Draw 阶段
            drawArc(
                brush = gradientBrush,
                startAngle = startAngle,
                sweepAngle = animatedProgress.value * maxSweep,
                useCenter = false,
                style = Stroke(width = strokeWidth.toPx(), cap = StrokeCap.Round)
            )
        }
    })
}
```

**核心原则**：
1. **延迟状态读取**：在 draw lambda 内读取 `.value`，不在顶层用 `by` 解包
2. **避免 GC 抖动**：Brush/Path 对象用 `drawWithCache` 缓存，不要每帧分配
3. **droidcon London 2024 要点**：Canvas 动画的 #1 性能杀手就是在 composition 阶段读取动画 state

### 1.5 动画参数推荐

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `dampingRatio` | `MediumBouncy` (≈0.5) | 弹性回弹效果 |
| `stiffness` | `StiffnessLow` | 300-800ms 收敛 |
| 进度变化周期 | 1000ms | 原需求每秒刷新 |
| 弧线粗细 | 6dp | 原需求 |
| 组件尺寸 | 64×64dp | 原需求 |

---

## 2. 对话历史 Rewind（树形分叉 + Undo/Redo）

> 对应备忘录 #4：rewind 后撤

### 2.1 核心数据结构：树形对话历史

**不用扁平列表，用树（Tree）+ Map<id, Node>**

ChatGPT/OpenCode 的"编辑后重发"本质上是对话分叉（branching），而非覆盖。扁平列表编辑会丢失原始消息，无法 undo。

```kotlin
data class MessageNode(
    val id: String,               // UUID
    val parentId: String?,        // 父节点 ID，根消息为 null
    val childrenIds: List<String>, // 所有子节点（编辑/重发产生的新版本）
    val text: String,
    val role: MessageRole,         // USER / ASSISTANT / SYSTEM
    val state: MessageState,
    val timestamp: Long,
    val metadata: Map<String, Any>?
)

data class ConversationState(
    val nodes: Map<String, MessageNode>,
    val rootId: String,
    val activeLeafId: String,     // 当前活跃路径的叶节点 ID（核心指针）
    val activePath: List<String>  // root → activeLeaf 路径（derivedStateOf 推导）
)
```

- **`activeLeafId`** 是 rewind/undo/redo 的核心——指向"当前对话尽头"
- **`activePath`** 通过 `derivedStateOf` 自动推导，作为 LazyColumn 渲染数据源
- 编辑消息时新建**兄弟节点**（共享 parentId），形成分叉

### 2.2 消息状态机

```kotlin
sealed class MessageState {
    object Idle : MessageState()
    object Sending : MessageState()
    object Streaming : MessageState()
    object Sent : MessageState()
    object Failed : MessageState()
    object Cancelled : MessageState()
    data class Editing(val originalId: String) : MessageState()
}
```

**状态转换**：
```
Idle → Sending → Streaming → Sent
                ↓
              Failed → (重试) → Sending
                ↓
            Cancelled (中断) → Idle

Sent → (编辑) → Editing → (提交) → Sending (新建分支)
Sent → (rewind) → 回到祖先节点的活跃路径
```

### 2.3 Undo/Redo = 树指针移动

```kotlin
// Undo: activeLeafId 上移到父节点
fun undo(state: ConversationState): ConversationState {
    val current = state.nodes[state.activeLeafId]!!
    if (current.parentId != null) {
        return state.copy(activeLeafId = current.parentId!!)
    }
    return state
}

// Redo: 从 redoStack 弹出
fun redo(state: ConversationState, redoStack: List<String>): ConversationState {
    if (redoStack.isEmpty()) return state
    return state.copy(activeLeafId = redoStack.last())
}
```

- 维护 `redoStack: List<String>`，undo 时压栈，redo 时弹栈
- **新操作清空 redoStack**（与 ChatGPT 行为一致）

### 2.4 MVI/Reducer 模式管理状态

```kotlin
sealed class ConversationIntent {
    data class SendMessage(val text: String) : ConversationIntent()
    data class EditMessage(val nodeId: String, val newText: String) : ConversationIntent()
    object Undo : ConversationIntent()
    object Redo : ConversationIntent()
    data class RewindTo(val nodeId: String) : ConversationIntent()
    data class RetryMessage(val nodeId: String) : ConversationIntent()
    object CancelStreaming : ConversationIntent()
    data class SwitchBranch(val nodeId: String, val direction: BranchDirection) : ConversationIntent()
}
```

### 2.5 Compose UI 实现

```kotlin
@Composable
fun ConversationScreen(state: ConversationState, onIntent: (ConversationIntent) -> Unit) {
    val activeMessages by remember(state) {
        derivedStateOf { state.activePath.map { state.nodes[it]!! } }
    }

    LazyColumn(reverseLayout = true) {
        items(
            items = activeMessages.reversed(),
            key = { it.id }  // ⚠️ 必须提供稳定唯一 key
        ) { message ->
            MessageBubble(
                message = message,
                onEdit = { onIntent(ConversationIntent.EditMessage(message.id, it)) },
                onRewind = { onIntent(ConversationIntent.RewindTo(message.id)) },
                modifier = Modifier.animateItem(
                    fadeInSpec = tween(250),
                    fadeOutSpec = tween(200),
                    placementSpec = spring(stiffness = 300f)
                )
            )
        }
    }
}
```

### 2.6 动画方案

| 场景 | 方案 | 说明 |
|------|------|------|
| 列表项增删 | `Modifier.animateItem()`（Compose 1.7.0+） | 取代已废弃的 `animateItemPlacement()` |
| 消息状态切换 | `AnimatedContent` | Sending→Streaming→Sent→Failed |
| Rewind 隐藏后续消息 | `slideOutVertically + fadeOut` | 向上消散动画 |
| 分叉切换 `<` `>` | `Crossfade` 或 `AnimatedContent` | 不同版本间淡入淡出 |
| 输入框回填 | 底部滑入聚焦 + 预填充文本 | spring-based motion |

**⚠️ 注意事项**：
1. `items()` 必须传稳定的 `message.id` 作为 key
2. 不要在 LazyColumn 内部嵌套 `AnimatedVisibility` 处理增删——exit 动画会被截断
3. ViewModel 增删改时必须 emit 新的 List 引用

### 2.7 分叉导航 UI

在编辑过的消息气泡上显示 `< 2/3 >` 切换器：
- 数字表示该节点的 children 中用户创建的版本数
- 点击 `<` `>` 在兄弟分支间切换 activeLeafId

### 2.8 参考项目

| 项目 | 说明 |
|------|------|
| **OpenChamber** (GitHub, 2025) | Branchable chat timeline，支持从任意回合分叉/撤销/重做 |
| **junerver/ComposeHooks** | React 风格 Compose hooks 库，含 `useUndoRedo` 和 `useOpenAIChat` |
| **NextAlone/Nagram** | 第三方 Telegram 客户端，实现了消息 Undo/Redo |

---

## 3. 服务器资源监控 Dashboard

> 对应备忘录 #10：服务器资源监控

### 3.1 实时数据刷新策略

| 策略 | 推荐度 | 适用场景 |
|------|--------|---------|
| **SSE**（okhttp-eventsource） | ⭐⭐⭐⭐⭐ | **首选**：单向推送，内置 Last-Event-ID 自动重连，标准 HTTP 穿透性好 |
| WebSocket | ⭐⭐⭐ | 需双向交互（远程命令执行）或极低延迟 |
| Polling | ⭐⭐ | 低频场景或网络降级后备 |

**关键策略**：
- 连接绑定 ViewModel 生命周期
- 前台时维持 SSE 连接，**后台时立即断开**省 radio 电量
- Doze 模式下降级为 FCM 推送（仅告警）

### 3.2 图表库选型

| 库 | 推荐度 | 说明 |
|------|--------|------|
| **MPAndroidChart** | ❌ 不推荐 | 已废弃，无 Compose 原生支持 |
| **Vico** | ⭐⭐⭐⭐ 首选 | 纯 Kotlin/Compose Multiplatform，活跃维护，`CartesianChartModelProducer` 后台线程处理 |
| **Compose Canvas 自绘** | ⭐⭐⭐⭐ 高频补充 | Reddit 实测 5ms 心跳下 Vico 性能不足，自绘 Canvas 绕过 layout/recomposition 直接 GPU 绘制最优 |

**性能优化**：
- 节流降采样至 15-30Hz
- 仅渲染可见窗口（最近 50 点）
- `CartesianChartModelProducer` 提升到 ViewModel 层
- Vico 可用 `snap()` 关闭 diff 动画提升高频性能

### 3.3 数据管线

```
SSE → Flow → ViewModel StateFlow → 节流(throttle 1s) → Vico/Canvas 渲染
```

### 3.4 CPU/RAM/Disk 可视化最佳实践

| 指标 | 可视化方式 | 说明 |
|------|-----------|------|
| CPU | 圆形仪表 + 实时折线图 | 0-100% 渐变色编码（绿→黄→红），多核可叠加 |
| RAM | 环形进度条三段式 | 总量/已用/可用 + Top N 进程横向柱状图 |
| 磁盘 | 分区堆叠柱状图 + I/O 双折线 | 读写速率可视化 |

**UI 布局**：卡片式 Dashboard（LazyVerticalGrid），可折叠详情，暗色主题优先

### 3.5 与备忘录设计的对接

- **快手场景**（侧边栏三指标圆弧）：用 `Modifier.drawWithCache` 自绘 Canvas（见第 1 节）
- **详细监控页**（如未来扩展）：用 Vico 图表库
- **数据来源**：复用备忘录设计的 `SSE system_stats` 事件

---

## 4. Material 3 Expressive 设计语言

> 对应备忘录 #8a：Plan/Build 双模式

### 4.1 M3 Expressive vs 标准 M3

| 特性 | 标准 M3 | M3 Expressive |
|------|---------|---------------|
| 动画 | 静态 tween | **spring physics 弹性物理动画** |
| 形状 | 固定圆角 | **35 种 Material Shapes + shape morphing** |
| 颜色 | dynamicColor | dynamicColor + **更丰富的色彩混合** |
| 图标 | 静态 | **AnimatedIcon + spring physics** |
| 运动方案 | 无 | **MotionScheme.expressive()** |

### 4.2 主题设置

```kotlin
@Composable
fun MiniboxTheme(content: @Composable () -> Unit) {
    val colorScheme = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        if (isDark) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
    } else {
        // Plan/Build 自定义配色
        if (mode == Mode.PLAN) PlanColorScheme else BuildColorScheme
    }

    MaterialExpressiveTheme(
        colorScheme = colorScheme,
        motionScheme = MotionScheme.expressive(),  // 弹性动画全局生效
        content = content
    )
}
```

### 4.3 Spring Motion Physics

- 通过 `MaterialTheme.motionScheme.defaultSpatialSpec()` 获取物理动画 specs
- `MotionScheme.expressive()` 自动为 scale、translation、rotation 应用弹性回弹
- **300ms 是同时变化 size 和 shape 的最佳时长**
- 替代所有静态 `tween` 动画为 `spring`

### 4.4 Shape Morphing（形状变形）

- 35 种 Material 形状（如 `Cookie9Sided`、`Pill`）
- 配合 `remember { Morph(restingShape, targetShape) }` + shared element transitions
- 按压/选中时流畅变形

### 4.5 Plan/Build 双模式切换实现

```kotlin
// Plan 模式：冷色调（蓝/青/紫）
val PlanColorScheme = darkColorScheme(
    primary = Color(0xFF82B1FF),
    secondary = Color(0xFF80CBC4),
    surface = Color(0xFF1A1B2E),
    // ...
)

// Build 模式：暖色调（橙/琥珀/红）
val BuildColorScheme = darkColorScheme(
    primary = Color(0xFFFFB74D),
    secondary = Color(0xFFFF8A65),
    surface = Color(0xFF2E1A1A),
    // ...
)
```

**切换动画**：用 `animateColorAsState` 平滑过渡配色，~400ms

### 4.6 新组件

| 组件 | 说明 |
|------|------|
| `LoadingIndicator` / `ContainedLoadingIndicator` | 新加载指示器，可自定义动画 |
| `AnimatedIcon` | 用 spring physics 替代静态图标 |
| `MaterialShapes` | 35 种可变形状 |

### 4.7 关键参考

- **Android Developers Blog** (2025-05-20)：Androidify: Building delightful UIs with Compose
- **Material Design** (2025-05-13)：Start building with Material 3 Expressive
- **Philipp Lackner** YouTube：The New Material3 Expressive Explained In 7 Minutes
- **Android Developers** YouTube：Build next-level UX with Material 3 Expressive

---

## 5. APP 在线更新 + 国内镜像下载

> 对应备忘录 #11：检查更新 + 下载源

### 5.1 GitHub Releases API 对接

```
GET https://api.github.com/repos/{owner}/{repo}/releases/latest
```

**关键字段**：`tag_name`（版本比较）、`body`（更新日志）、`assets[].browser_download_url`、`assets[].size`

⚠️ **Rate limit**：未认证 60 次/小时/IP，认证后 5,000 次/小时。建议嵌入 Fine-grained PAT 且缓存检查结果。

### 5.2 国内镜像加速 URL 拼接规则

**两种类型**：

| 类型 | 规则 | 示例 |
|------|------|------|
| **前缀拼接型** | `{代理前缀}/` + 原始 GitHub URL | `https://ghproxy.net/https://github.com/...` |
| **域名替换型** | 直接替换 `github.com` | `https://kkgithub.com/...` |

**可用镜像源（6+ 个）**：

| # | 镜像 | 类型 | URL |
|---|------|------|-----|
| 1 | ghproxy.net | 前缀拼接 | `https://ghproxy.net/https://github.com/...` |
| 2 | gh-proxy.com | 前缀拼接 | `https://gh-proxy.com/https://github.com/...` |
| 3 | ghproxy.homeboyc.cn | 前缀拼接 | `https://ghproxy.homeboyc.cn/https://github.com/...` |
| 4 | moeyy.cn/gh-proxy | 前缀拼接 | `https://moeyy.cn/gh-proxy/https://github.com/...` |
| 5 | toolwa.com | 前缀拼接 | `https://toolwa.com/https://github.com/...` |
| 6 | kkgithub.com | 域名替换 | `https://kkgithub.com/...` |
| 7 | bgithub.xyz | 域名替换 | `https://bgithub.xyz/...` |

### 5.3 多镜像自动测速

- 并发 HEAD 请求测量 TTFB（Time To First Byte）
- 选择最快可用源
- 测试结果会话内缓存（5 分钟）
- 直连 GitHub 作为兜底

### 5.4 下载方案对比

| 方案 | 推荐度 | 优势 | 劣势 |
|------|--------|------|------|
| **OkHttp + WorkManager** | ⭐⭐⭐⭐⭐ | 进度精确、镜像测速复用、自定义通知、路径可控 | 需自行实现进度通知 |
| DownloadManager | ⭐⭐⭐ | 系统内置、简单 | 定制 ROM 兼容问题、进度不精确 |

### 5.5 安装权限处理

| 权限 | 级别 | 说明 |
|------|------|------|
| `INSTALL_PACKAGES` | 系统级 | 普通 App 无法获取 |
| `REQUEST_INSTALL_PACKAGES` | 普通 | 仅允许用户确认安装 |

**标准方案**：FileProvider + `ACTION_VIEW` Intent → 弹出系统安装界面

**精细方案**：`PackageInstaller` API → 获得安装状态回调

**准静默安装**：[Ackpine](https://github.com/solrudev/Ackpine) 库支持 Shizuku/Root 后端

### 5.6 整体架构

```
检查更新（GitHub API）
  → 有新版本
  → 弹窗显示更新日志
  → 用户选择下载源（或自动测速）
  → OkHttp + WorkManager 下载
  → 通知栏进度条
  → 下载完成
  → FileProvider + ACTION_VIEW
  → 系统安装界面
  → 用户确认安装
```

### 5.7 注意事项

1. Google Play 政策风险：除商店更新外的安装方式可能违反政策（侧载分发场景不适用）
2. FileProvider 配置：必须在 `AndroidManifest.xml` 中正确配置 `file_paths.xml`
3. APK 完整性校验：下载后校验 SHA256 与 Release 页公布的一致性
4. 网络超时：镜像测速 3 秒超时，下载 30 秒超时重试

> 📄 详细研究报告：[research/android-app-auto-update-github-releases.md](omnibot://workspace/research/android-app-auto-update-github-releases.md)

---

## 6. 侧边栏 Drawer + TodoList 浮窗 + SFTP 文件管理

> 对应备忘录 #5b、#10、#8c

### 6.1 TodoList 浮窗（右侧竖条 + 点击展开）

**不能仅靠标准 Drawer 实现**，需自定义组合：

| 组件 | API |
|------|-----|
| 容器 | `Box` 叠层 |
| 手势识别 | `Modifier.pointerInput { detectHorizontalDragGestures { ... } }` |
| 点击展开 | `detectTapGestures { ... }` |
| 边缘触发 | 记录 `PointerInputChange.position` 判断是否从屏幕边缘发起 |
| 展开/收起动画 | `animateDpAsState` + `spring(stiffness=350, dampingRatio=0.35)` |

```kotlin
// 核心手势模式
Box(
    modifier = Modifier
        .pointerInput(Unit) {
            detectHorizontalDragGestures(
                onDragStart = { offset ->
                    // 判断是否从边缘开始
                    isFromEdge = offset.x < edgeThreshold || offset.x > screenWidth - edgeThreshold
                },
                onHorizontalDrag = { change, dragAmount ->
                    // 跟踪拖拽距离，控制展开宽度
                },
                onDragEnd = {
                    // 根据拖拽距离决定展开/收起
                }
            )
        }
        .pointerInput(Unit) {
            detectTapGestures { /* 点击展开 */ }
        }
)
```

### 6.2 右滑 Drawer 侧边栏

**标准 M3 方案**：

```kotlin
ModalNavigationDrawer(
    drawerState = drawerState,
    drawerContent = { /* 侧边栏内容 */ },
    gesturesEnabled = true  // 启用边缘手势
)
```

**更灵活方案**（自定义右对齐）：
- 完全自定义 `Box` 叠层 + `offset` 动画 + 透明边缘检测区
- `DrawerState.open()/close()` 配合 `CoroutineScope` 程序化控制

**关键 API 体系**：
- `pointerInput` → `detectDragGestures` / `detectHorizontalDragGestures` / `detectTapGestures` / `awaitEachGesture`（Compose 1.6+）

### 6.3 SFTP 文件管理

**推荐：`com.github.mwiede:jsch`（JSch 活跃 fork）**

| 对比 | JSch (mwiede) | SSHJ | Apache Commons VFS |
|------|--------------|------|-------------------|
| Android 适用性 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| 依赖体积 | 轻量(~400KB) | 重(slf4j+Bouncy Castle) | 很重(完整VFS) |
| 现代加密(Ed25519) | ✅ | ✅ | ❌ |
| Android 稳定性 | 优秀 | 有不稳定报告 | 臃肿 |

**关键佐证**：开源文件管理器 Material Files（GitHub Issue #1156, 2024-03-12）从 SSHJ 迁移回 JSch，"SSHJ has been a bit unstable on Android"。Commons VFS 底层也用 JSch。

**实现建议**：用 Kotlin Coroutines `Dispatchers.IO` 封装 JSch Session/Channel 生命周期，配合"双通道"架构（SFTP 直连快速传输 + API 安全审计）。

> 📄 详细研究报告：[research/compose-drawer-floating-overlay-sftp.md](omnibot://workspace/research/compose-drawer-floating-overlay-sftp.md)

---

## 7. 浏览器代理方案优化

> 对应备忘录 #6、#7

### 7.1 推荐架构：CDP LocalSocket + Headless Service

基于备忘录已确认的方案，搜索结果补充以下优化建议：

| 层级 | 方案 | 说明 |
|------|------|------|
| 渲染层 | WebView Compose | `com.google.accompanist:accompanist-webview` 或原生 AndroidView |
| 通信层 | Chrome DevTools Protocol (CDP) | 通过 LocalSocket 连接系统 WebView 的 DevTools |
| 代理层 | Headless Service | 后端发送 CDP 指令，前端 WebView 执行 |
| 连接复用 | WebSocket :8086 | 与设备代理共享同一连接 |

### 7.2 WebView 性能优化

- 启用 WebView 硬件加速：`android:layerType="hardware"`
- 预加载 WebView Pool：APP 启动时创建 WebView 实例池
- 内存优化：页面不可见时 `webView.onPause()` + `webView.disableJavaScript()`
- User-Agent 管理：可切换桌面/移动 UA

### 7.3 油猴脚本兼容

- 注入 `evaluateJavascript()` 在 `onPageFinished` 回调中
- 脚本管理：本地存储脚本列表，按 URL 匹配自动注入
- 支持备注