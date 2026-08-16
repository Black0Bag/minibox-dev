# Android Jetpack Compose 浮窗/侧边栏 + SFTP 文件管理最佳实践研究

## 一、Compose 实现 TodoList 浮窗（右侧竖条 + 点击展开）和侧边栏（右滑 Drawer）

### 1.1 Material 3 NavigationDrawer 体系

Jetpack Compose Material 3 提供三种 Drawer 组件：

| 组件 | 特性 | 适用场景 |
|------|------|----------|
| **ModalNavigationDrawer** | 渲染 scrim 遮罩覆盖内容，Drawer 滑入覆盖在屏幕上方 | 标准侧边导航菜单 |
| **DismissibleNavigationDrawer** | Drawer 消失时内容区域平铺，无持久 modal scrim | 需要内容自适应的布局 |
| **PermanentNavigationDrawer** | Drawer 始终可见 | 平板/大屏设备 |

**关键 API：**
- `DrawerState`：通过 `rememberDrawerState(initialValue)` 创建，提供 `open()` / `close()` / `currentValue` 方法
- `gesturesEnabled`：布尔参数（默认 true），控制边缘滑动和点击遮罩关闭手势
- `DrawerValue.Open` / `DrawerValue.Closed`：两种状态枚举

**代码骨架：**
```kotlin
val drawerState = rememberDrawerState(DrawerValue.Closed)
val scope = rememberCoroutineScope()

ModalNavigationDrawer(
    drawerState = drawerState,
    gesturesEnabled = true,  // 允许边缘滑动
    drawerContent = {
        ModalDrawerSheet {
            // 侧边栏内容
        }
    }
) {
    // 主内容
    Scaffold { padding ->
        // 主界面
    }
}
```

### 1.2 类似 OpenCode 的右侧竖条浮窗实现方案

OpenCode 的 TodoList 浮窗特征：屏幕右侧一个垂直竖条（handle），点击或拖拽后展开为完整面板。这需要**自定义 Composable**，不能仅靠标准 Drawer 实现。

#### 推荐架构：Box + AnimatedVisibility + pointerInput

```kotlin
@Composable
fun FloatingTodoOverlay() {
    var expanded by remember { mutableStateOf(false) }
    var dragOffset by remember { mutableStateOf(0f) }
    val animatedWidth by animateDpAsState(
        targetValue = if (expanded) 320.dp else 8.dp,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessMedium
        )
    )

    Box(
        modifier = Modifier
            .fillMaxHeight()
            .width(animatedWidth)
            .align(Alignment.CenterEnd)
            .background(MaterialTheme.colorScheme.surfaceVariant)
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragStart = { },
                    onDragEnd = {
                        if (dragOffset < -50f || dragOffset > 50f) {
                            expanded = !expanded
                        }
                        dragOffset = 0f
                    }
                ) { change, dragAmount ->
                    dragOffset += dragAmount
                    change.consume()
                }
            }
            .clickable { expanded = !expanded }
    ) {
        if (expanded) {
            // 展开后的 TodoList 内容
            TodoListContent()
        } else {
            // 收起时的竖条 handle
            VerticalHandleBar()
        }
    }
}
```

#### 关键技术点：

1. **手势识别 - `pointerInput` + `detectHorizontalDragGestures`**：
   - `detectDragGestures`：通用拖拽检测，通过 `dragAmount` 参数获取滑动量
   - `detectHorizontalDragGestures`：专门检测水平拖拽，适合左右滑手势
   - `detectTapGestures`：检测点击/长按
   - 通过 `change.consume()` 消费事件防止冒泡

2. **边缘触发**：
   - 使用透明边缘区域布置 `pointerInput` 检测区域
   - 记录 `PointerInputChange.position` 判断是否从屏幕边缘发起
   - 可设定阈值（如边缘 24dp 内开始才触发 Drawer）

3. **动画过渡**：
   - `animateDpAsState` + `spring` 弹簧动画：适合宽度变化
   - `AnimatedVisibility`：配合 `expandHorizontally` / `shrinkHorizontally` 过渡
   - `Crossfade`：内容切换淡入淡出
   - `Transition` + `createAnimatedVisibility`：多状态动画串联
   - 建议参数：`stiffness = 350`, `dampingRatio = 0.35f`（与项目已有动画风格一致）

4. **浮窗层级**：
   - 使用 `Box` 叠加在主内容上方
   - 或使用 `Popup` / `Dialog` 实现系统层级悬浮
   - 推荐在 `Scaffold` 外层用 `Box` 包裹，确保浮窗在最上层

### 1.3 右滑 Drawer 侧边栏

**方案 A：使用 ModalNavigationDrawer（右侧）**

M3 的 `ModalNavigationDrawer` 默认从左侧滑出。要实现右侧 Drawer，需要自定义：

```kotlin
ModalNavigationDrawer(
    drawerState = drawerState,
    drawerContent = {
        ModalDrawerSheet(
            modifier = Modifier.align(Alignment.End)  // 右对齐
        ) { /* ... */ }
    )
)
```

**方案 B：自定义 DismissibleNavigationDrawer**

```kotlin
DismissibleNavigationDrawer(
    drawerState = drawerState,
    drawerContent = {
        DismissibleDrawerSheet {
            // 侧边导航内容
        }
    }
) {
    // 主内容
}
```

**方案 C：完全自定义（最灵活）**

适合需要类似 OpenCode 的特殊交互（非标准 Drawer 模式）：

```kotlin
var drawerOpen by remember { mutableStateOf(false) }
val drawerOffset by animateFloatAsState(
    targetValue = if (drawerOpen) 0f else 1f,
    animationSpec = tween(300)
)

Box(Modifier.fillMaxSize()) {
    // 主内容
    MainContent()

    // 遮罩
    if (drawerOpen) {
        Box(
            Modifier
                .fillMaxSize()
                .background(Color.Black.copy(alpha = (1f - drawerOffset) * 0.5f))
                .clickable { drawerOpen = false }
        )
    }

    // 右侧 Drawer 面板
    Box(
        Modifier
            .fillMaxHeight()
            .width(300.dp)
            .offset(x = ((1f - drawerOffset) * 300).dp)  // 从右侧滑入
            .align(Alignment.CenterEnd)
    ) {
        DrawerContent()
    }

    // 右侧边缘触发区
    Box(
        Modifier
            .fillMaxHeight()
            .width(24.dp)
            .align(Alignment.CenterEnd)
            .pointerInput(Unit) {
                detectHorizontalDragGestures { change, dragAmount ->
                    if (dragAmount < -20f) {
                        drawerOpen = true
                    }
                    change.consume()
                }
            }
    )
}
```

### 1.4 手势识别深度分析

#### pointerInput 修饰符体系

根据 Android 官方文档和社区最佳实践（2024-2025）：

1. **`Modifier.pointerInput(Unit)`**：
   - 最底层的手势检测入口
   - 在协程中运行，支持 `awaitPointerEventScope`
   - 可同时监听多个手势检测器

2. **检测器函数**：
   - `detectDragGestures(onDragStart, onDragEnd, onDragCancel)`：通用拖拽
   - `detectHorizontalDragGestures`：水平拖拽，适合左右滑手势
   - `detectVerticalDragGestures`：垂直拖拽
   - `detectTapGestures(onTap, onDoubleTap, onLongPress)`：点击手势
   - `awaitEachGesture`：Compose 1.6+ 的自定义手势构建器

3. **事件分发机制**：
   - Compose 采用"父子链式分发"：先到子节点再到父节点
   - `change.consume()` 消费事件防止冒泡
   - 多层 `pointerInput` 通过 `awaitPointerEventScope` 按层级分发
   - 注意嵌套滚动冲突需用 `Modifier.nestedScroll` 处理

4. **边缘检测实现**：
   ```kotlin
   .pointerInput(Unit) {
       val width = size.width.toFloat()
       detectDragGestures(
           onDragStart = { offset ->
               // 判断是否从右边缘开始
               if (offset.x > width - edgeThreshold) {
                   isEdgeSwipe = true
               }
           }
       )
   }
   ```

### 1.5 动画过渡最佳实践

| 场景 | 推荐 API | 参数建议 |
|------|----------|----------|
| 宽度展开/收起 | `animateDpAsState` + `spring` | stiffness=350, dampingRatio=0.35 |
| 透明度渐变 | `animateFloatAsState` + `tween` | durationMillis=300 |
| 内容切换 | `AnimatedVisibility` + `expandHorizontally` | |
| 多状态动画 | `updateTransition` + `createAnimatedVisibility` | |
| 手势跟随 | `Modifier.offset` + 实时 `dragAmount` | 手指松开后 snap 到最终态 |
| 形状变形 | `animateColorAsState` + `Crossfade` | ~300ms |

**弹簧动画推荐参数（与项目已有风格统一）：**
```kotlin
val springSpec = spring<Float>(
    dampingRatio = 0.35f,   // MediumBouncy
    stiffness = 350f        // Medium
)
```

### 1.6 架构建议

```
Scaffold
  └── Box (fillMaxSize)
      ├── MainContent (NavHost / LazyColumn)
      ├── FloatingTodoBar (竖条/展开面板)
      │   └── pointerInput (detectHorizontalDragGestures + detectTapGestures)
      │   └── AnimatedVisibility / animateDpAsState
      └── DrawerOverlay (可选)
          └── ModalNavigationDrawer / 自定义 Drawer
```

---

## 二、Android SFTP 文件管理方案对比（JSch vs SSHJ vs Apache Commons VFS）

### 2.1 核心对比

| 维度 | JSch (mwiede/jsch fork) | SSHJ | Apache Commons VFS |
|------|------------------------|------|-------------------|
| **Android 适用性** | ⭐⭐⭐⭐⭐ 最佳 | ⭐⭐⭐ 一般 | ⭐ 差 |
| **底层实现** | 原生实现 | 原生实现 | 底层使用 JSch |
| **依赖体积** | 轻量（~400KB） | 重（需 slf4j + Bouncy Castle） | 很重（完整 VFS 框架） |
| **现代加密支持** | ✅ Ed25519, rsa-sha2-256/512 | ✅ 支持 | ❌ 继承底层 JSch 限制 |
| **API 风格** | 低级，过程式 | 高级，面向对象 | 高级，URI 风格 |
| **维护状态** | 活跃（mwiede fork） | 活跃 | 活跃但 VFS 对 Android 不优化 |
| **Android 稳定性** | ✅ 优秀 | ⚠️ 有不稳定报告 | ❌ 臃肿 |
| **ProGuard/R8 兼容** | ✅ 无问题 | ⚠️ 签名 JAR 冲突 | ⚠️ 依赖链复杂 |

### 2.2 各方案详细分析

#### 1️⃣ JSch（推荐使用 mwiede/jsch fork）— **首选**

**背景：**
- 原始 `com.jcraft:jsch` 已停止维护，导致现代服务器 "Algorithm negotiation fail" 错误
- **`com.github.mwiede:jsch`** 是活跃 fork，完全兼容原 API，添加了现代加密支持

**Android 优势：**
- 超小 footprint，零第三方依赖 → 不会膨胀 APK
- 不引入 Bouncy/Spongy Castle → 避免 ProGuard/R8 签名 JAR 冲突
- 完美适配后台线程操作

**依赖：**
```gradle
implementation 'com.github.mwiede:jsch:0.2.20'
// 或 Maven Central:
implementation 'com.github.mwiede:jsch:0.2.x'
```

**典型用法：**
```kotlin
val session = JSch().getSession(user, host, port)
session.setPassword(password)
session.setConfig("StrictHostKeyChecking", "no")
session.connect(timeout)

val channel = session.openChannel("sftp") as ChannelSftp
channel.connect()

// 列目录
channel.ls("/path").forEach { entry ->
    println((entry as LsEntry).getFilename())
}

// 上传
channel.put(FileInputStream(localFile), "/remote/path/file")

// 下载
channel.get("/remote/path/file", FileOutputStream(localFile))

channel.disconnect()
session.disconnect()
```

**缺点：**
- API 风格较老（过程式 channel 操作）
- 需要手动管理 session/channel 生命周期

#### 2️⃣ SSHJ — **备选**

**优势：**
- 现代、优雅的面向对象 API
- 在后端 Java 环境中备受好评

**Android 劣势：**
- 依赖 slf4j 日志框架
- 需要 Bouncy Castle / Spongy Castle 安全提供者
- Bouncy Castle 与 Android 构建系统的签名冲突问题
- **Material Files 等开源 Android 文件管理器报告了 SFTP 在 Android 上的不稳定问题**，最终迁移回 JSch

**Material Files 迁移记录（GitHub Issue #1156, 2024-03-12）：**
> "SSHJ has been a bit unstable, and doesn't support #1152. Commons VFS is actually also using JSch."

**依赖：**
```gradle
implementation 'com.hierynomus:sshj:0.38.0'
implementation 'org.slf4j:slf4j-android:1.7.36'
```

#### 3️⃣ Apache Commons VFS — **不推荐用于 Android**

**劣势：**
- 整个虚拟文件系统框架，极其臃肿
- 对 Android 完全不优化
- 底层实现实际上也是使用 JSch
- 引入大量传递依赖，膨胀 APK

**唯一适用场景：** 需要统一的文件系统抽象层（同时支持 FTP/SFTP/WebDAV/本地等），但 Android 上不推荐。

### 2.3 项目实际经验印证

从项目记忆中可以看到，minibox 项目已在实际使用 SFTP 进行远程文件管理：
- 通过 SSH + SFTP 连接路由器/服务器执行远程更新
- SFTP 上传二进制 → 备份旧文件 → 替换 → 重启服务的流程已验证成功
- 外网 IPv6（blackbag.dynv6.net:22）SFTP 连接稳定
- 用户设计了"双通道"架构：SFTP 直连（快速传输）+ API 文件操作（安全审计）

### 2.4 推荐方案：mwiede/jsch + Kotlin Coroutines 封装

```kotlin
class SftpClient(
    private val host: String,
    private val port: Int = 22,
    private val user: String,
    private val password: String? = null,
    private val keyPath: String? = null
) {
    private var session: Session? = null
    private var channel: ChannelSftp? = null

    suspend fun connect() = withContext(Dispatchers.IO) {
        val jsch = JSch()
        if (keyPath != null) jsch.addIdentity(keyPath)

        session = jsch.getSession(user, host, port).apply {
            if (password != null) setPassword(password)
            setConfig("StrictHostKeyChecking", "no")
            connect(30000)
        }

        channel = (session?.openChannel("sftp") as ChannelSftp).apply {
            connect(30000)
        }
    }

    suspend fun listFiles(path: String): List<RemoteFile> = withContext(Dispatchers.IO) {
        channel?.ls(path)?.mapNotNull { entry ->
            (entry as? LsEntry)?.let {
                RemoteFile(
                    name = it.filename,
                    isDir = it.attrs.isDir,
                    size = if (it.attrs.isDir) 0 else it.attrs.size,
                    lastModified = it.attrs.mTime
                )
            }
        } ?: emptyList()
    }

    suspend fun upload(localPath: String, remotePath: String) = withContext(Dispatchers.IO) {
        channel?.put(FileInputStream(localPath), remotePath)
    }

    suspend fun download(remotePath: String, localPath: String) = withContext(Dispatchers.IO) {
        channel?.get(remotePath, FileOutputStream(localPath))
    }

    suspend fun disconnect() = withContext(Dispatchers.IO) {
        channel?.disconnect()
        session?.disconnect()
    }
}
```

### 2.5 其他备选方案

| 库 | 说明 | 适用场景 |
|----|------|----------|
| **Apache MINA SSHD** | 100% pure Java，支持 SSH 客户端+服务端 | 需要在 Android 上跑 SSH server 的场景 |
| **Maverick Synergy** | 现代化 SSH API，注重性能和安全 | 正在评估的新方案 |
| **sshj-android** | SSHJ 的 Android 封装版 | 不推荐，仍有底层依赖问题 |

---

## 三、综合架构建议

### 整体技术栈推荐

```
UI 层: Jetpack Compose + Material 3 Expressive
  ├── ModalNavigationDrawer / 自定义右滑 Drawer
  ├── 自定义 FloatingTodoBar (pointerInput + AnimatedVisibility)
  └── 动画: spring(stiffness=350, dampingRatio=0.35)

网络层: OkHttp + Retrofit (API 通信)
  ├── SSE: okhttp-eventsource (实时推送)
  └── SFTP: mwiede/jsch (文件传输)

架构: MVVM + Hilt DI + Room (本地缓存)
  ├── SFTP 操作: Kotlin Coroutines + Flow 封装
  ├── 双通道: SFTP 直连（快速）+ API（安全审计）
  └── 后台任务: WorkManager (SFTP 连接/文件传输)
```

### 关键决策

1. **Drawer 方案**：标准场景用 `ModalNavigationDrawer`（M3），特殊交互（如 OpenCode 式竖条浮窗）用自定义 `Box + pointerInput + AnimatedVisibility`
2. **手势**：`pointerInput` + `detectHorizontalDragGestures` 做边缘触发，`clickable` 做点击展开
3. **动画**：`spring` 弹簧动画统一参数（stiffness=350, dampingRatio=0.35），与项目已有风格一致
4. **SFTP**：直接选 `com.github.mwiede:jsch`，用 Kotlin Coroutines `Dispatchers.IO` 封装，避免 SSHJ 和 Commons VFS

---

*研究来源：Android Developers 官方文档、Google AI 概览、Material Files GitHub Issue #1156 (2024)、Medium/Itsuki 文章 (2024-03)、Stack Overflow 社区讨论 (2024-2025)、项目记忆 (2026-07-30/08-01)*
