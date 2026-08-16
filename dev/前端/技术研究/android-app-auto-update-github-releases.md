# Android APP 在线更新检查与 APK 下载安装最佳实践

> 基于 2024-2025 年技术资料、GitHub 官方 API 文档、开源项目（gh-proxy、AppUpdater、APKUpdater、Ackpine）及社区实践的综合整理。

---

## 一、GitHub Releases API 对接

### 1.1 核心 API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/repos/{owner}/{repo}/releases` | GET | 列出所有 Release（不含普通 Git tag） |
| `/repos/{owner}/{repo}/releases/latest` | GET | 获取最新 Release（排除 prerelease 和 draft） |
| `/repos/{owner}/{repo}/releases/tags/{tag}` | GET | 按标签名获取特定 Release |

**请求示例：**
```bash
GET https://api.github.com/repos/{owner}/{repo}/releases/latest
Headers:
  Accept: application/vnd.github+json
  Authorization: Bearer <YOUR-TOKEN>       # 可选，提高 rate limit
  X-GitHub-Api-Version: 2022-11-28
```

### 1.2 关键响应字段

```json
{
  "tag_name": "v1.2.3",          // 版本标签，用于版本比较
  "name": "v1.2.3",              // Release 标题
  "body": "Release notes...",     // 更新日志
  "prerelease": false,            // 是否预发布版
  "draft": false,                 // 是否草稿
  "published_at": "2024-01-15T10:00:00Z",
  "assets": [
    {
      "name": "app-release.apk",              // 文件名
      "browser_download_url": "https://github.com/{owner}/{repo}/releases/download/v1.2.3/app-release.apk",
      "size": 15249920,                       // 文件大小（字节）
      "download_count": 1234,
      "content_type": "application/vnd.android.package-archive"
    }
  ]
}
```

### 1.3 Rate Limit 处理（关键！）

| 场景 | 限制 |
|------|------|
| 未认证请求 | **60 次/小时**（按 IP 计） |
| 个人 Token 认证 | 5,000 次/小时 |
| GitHub Actions Token | ~1,000 次/小时 |
| GitHub App (Enterprise) | 15,000 次/小时 |

**最佳实践：**
- 检查更新不应频繁调用，建议 **仅在 App 启动时或用户手动触发时** 调用一次
- 响应头包含 `X-RateLimit-Remaining` 和 `X-RateLimit-Reset`，可用于判断是否触达限制
- 可在高频场景嵌入一个轻量级认证 Token（Fine-grained PAT，仅需 `Contents: read` 权限），将限制提升到 5,000 次/小时
- **缓存策略**：本地缓存上次检查结果，同一会话内不重复请求
- 收到 403/429 时，读取 `X-RateLimit-Reset` 头计算重试时间，不可立即重试

### 1.4 版本号比较最佳实践

```kotlin
/**
 * 建议使用 tag_name（如 "v1.2.3"）进行语义化版本比较
 * 去除 'v' 前缀后按 major.minor.patch 比较
 */
fun compareVersions(remote: String, local: String): Int {
    val remoteParts = remote.removePrefix("v").split(".").map { it.toIntOrNull() ?: 0 }
    val localParts = local.removePrefix("v").split(".").map { it.toIntOrNull() ?: 0 }
    for (i in 0 until maxOf(remoteParts.size, localParts.size)) {
        val r = remoteParts.getOrElse(i) { 0 }
        val l = localParts.getOrElse(i) { 0 }
        if (r != l) return r - l
    }
    return 0
}
```

也可使用 `versionCode`（整数）：将 `versionCode` 写入 Release 的 `body` 或 asset 文件名中解析。

---

## 二、国内镜像加速 — ghproxy 等代理服务的 URL 拼接规则

### 2.1 gh-proxy (hunshcn/gh-proxy) 核心规则

**基础规则：在原始 GitHub URL 前直接拼接代理前缀。**

```
原始URL + 代理前缀 = 加速URL
```

| 资源类型 | 原始 URL | 加速 URL |
|---------|---------|---------|
| Release 文件 | `https://github.com/{u}/{r}/releases/download/v1.0.0/app.apk` | `{代理前缀}https://github.com/{u}/{r}/releases/download/v1.0.0/app.apk` |
| 分支源码 | `https://github.com/{u}/{r}/archive/master.zip` | `{代理前缀}https://github.com/{u}/{r}/archive/master.zip` |
| Release 源码 | `https://github.com/{u}/{r}/archive/v0.1.0.tar.gz` | `{代理前缀}https://github.com/{u}/{r}/archive/v0.1.0.tar.gz` |
| raw 文件 | `https://raw.githubusercontent.com/{u}/{r}/master/file` | `{代理前缀}https://raw.githubusercontent.com/{u}/{r}/master/file` |

### 2.2 可用的镜像代理前缀列表（2024-2025 验证）

| 镜像站 | 代理前缀 URL | 特点 |
|--------|-------------|------|
| gh-proxy.com | `https://gh-proxy.com/` | 支持批量加速，稳定 |
| ghproxy.net | `https://ghproxy.net/` | 自动识别文件类型，支持断点续传 |
| ghproxy.homeboyc.cn | `https://ghproxy.homeboyc.cn/` | 适合大体积 Release 包 |
| moeyy.cn | `https://moeyy.cn/gh-proxy/` | 专注于文件下载加速 |
| toolwa.com | `http://toolwa.com/github/` | 带文件大小显示 |
| gh.api.99988866.xyz | `https://gh.api.99988866.xyz/` | gh-proxy 官方演示站 |

### 2.3 URL 拼接代码示例

```kotlin
/**
 * 将 GitHub 下载 URL 转为镜像加速 URL
 * @param originalUrl 原始 browser_download_url
 * @param mirrorPrefix 镜像代理前缀，如 "https://ghproxy.net/"
 */
fun buildMirrorUrl(originalUrl: String, mirrorPrefix: String): String {
    return mirrorPrefix + originalUrl
}

// 示例：
// 原始: https://github.com/user/repo/releases/download/v1.0.0/app.apk
// 加速: https://ghproxy.net/https://github.com/user/repo/releases/download/v1.0.0/app.apk
```

### 2.4 域名替换型镜像（非前缀拼接型）

| 镜像站 | 替换规则 |
|--------|---------|
| kkgithub.com | `github.com` → `kkgithub.com` |
| bgithub.xyz | `github.com` → `bgithub.xyz` |
| gitclone.com | `github.com` → `gitclone.com` |

这类镜像直接替换域名，不需要前缀拼接：
```
https://kkgithub.com/user/repo/releases/download/v1.0.0/app.apk
```

---

## 三、多镜像源自动测速选最快方案

### 3.1 设计思路

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ 镜像源列表   │────▶│ 并发 HEAD 请求 │────▶│ 选择最快可用源   │
│ (5-8 个候选) │     │ 测量 TTFB     │     │ 构建下载 URL     │
└─────────────┘     └──────────────┘     └─────────────────┘
```

### 3.2 测速实现方案

```kotlin
data class MirrorSource(
    val name: String,
    val prefix: String,       // 代理前缀或域名替换规则
    val type: MirrorType,      // PREFIX_APPEND 或 DOMAIN_REPLACE
)

enum class MirrorType {
    PREFIX_APPEND,    // 前缀拼接型: prefix + originalUrl
    DOMAIN_REPLACE    // 域名替换型: 原始URL中 github.com → prefix
}

val mirrorSources = listOf(
    MirrorSource("ghproxy.net", "https://ghproxy.net/", MirrorType.PREFIX_APPEND),
    MirrorSource("gh-proxy.com", "https://gh-proxy.com/", MirrorType.PREFIX_APPEND),
    MirrorSource("kkgithub", "https://kkgithub.com", MirrorType.DOMAIN_REPLACE),
    MirrorSource("moeyy", "https://moeyy.cn/gh-proxy/", MirrorType.PREFIX_APPEND),
    MirrorSource("direct", "", MirrorType.PREFIX_APPEND), // 直连 GitHub 作为兜底
)

/**
 * 对每个镜像源发送 HEAD 请求，测量首个字节到达时间(TTFB)
 * 选择 TTFB 最小且 HTTP 状态码为 200 的源
 */
suspend fun selectFastestMirror(
    originalUrl: String,
    sources: List<MirrorSource>,
    timeoutMs: Long = 5000
): MirrorSource {
    return coroutineScope {
        val results = sources.map { source ->
            async {
                val testUrl = when (source.type) {
                    MirrorType.PREFIX_APPEND -> source.prefix + originalUrl
                    MirrorType.DOMAIN_REPLACE -> originalUrl.replace("github.com", source.prefix.removePrefix("https://"))
                }
                try {
                    val startTime = System.currentTimeMillis()
                    val response = withTimeout(timeoutMs) {
                        OkHttpClient().newCall(
                            Request.Builder().url(testUrl).head().build()
                        ).execute()
                    }
                    val ttfb = System.currentTimeMillis() - startTime
                    if (response.isSuccessful || response.code in 301..302) {
                        source to ttfb
                    } else {
                        source to Long.MAX_VALUE
                    }
                } catch (e: Exception) {
                    source to Long.MAX_VALUE
                }
            }
        }.awaitAll()

        results.minByOrNull { it.second }?.first ?: sources.last()
    }
}
```

### 3.3 测速最佳实践

- **HEAD 请求优先**：用 HEAD 而非 GET，避免下载完整文件
- **超时设置**：单个镜像超时 3-5 秒即可
- **并发控制**：使用 Kotlin Coroutine 的 `async` 并发测速，`awaitAll` 等待全部完成
- **结果缓存**：测速结果在同一次 App 会话内缓存（如 5 分钟），避免频繁重测
- **降级策略**：所有镜像都失败时，直连 GitHub 作为最后兜底
- **文件大小预获取**：从 GitHub API 的 `assets[].size` 字段获取，用于显示下载大小
- **Content-Type 检测**：HEAD 响应中确认 `Content-Type` 为 `application/vnd.android.package-archive`

---

## 四、DownloadManager vs OkHttp 下载方案对比

### 4.1 详细对比表

| 特性 | DownloadManager (系统服务) | OkHttp + 协程 (自实现) |
|------|---------------------------|----------------------|
| **进度回调** | 需轮询 Cursor 查询状态（延迟大） | 实时流式回调（精确） |
| **断点续传** | 系统自动支持 | 需手动实现 Range 请求 |
| **后台/断网恢复** | 系统级后台下载，断网自动重试 | 需自行实现重试逻辑 |
| **通知栏显示** | 内置系统通知 | 需自建 Notification |
| **权限要求** | Android 10+ 存储到 Downloads 无需权限 | 需 WRITE_EXTERNAL_STORAGE（targetSdk < 29） |
| **自定义性** | 低（系统行为不可控） | 高（完全自定义） |
| **下载文件路径** | 系统决定，需通过 `getUriForDownloadedFile` 获取 | 完全可控 |
| **大数据文件** | 适合大文件（系统管理） | 需注意内存和文件流处理 |
| **Huawei/MIUI 适配** | 部分定制 ROM 有兼容问题 | 无兼容问题 |
| **监听完成** | 需注册 `DownloadManager.ACTION_DOWNLOAD_COMPLETE` 广播 | 回调直接通知 |
| **取消下载** | `remove(id)` | `call.cancel()` |

### 4.2 方案推荐

**推荐方案：OkHttp + WorkManager（最佳实践）**

理由：
1. **进度精确控制**：OkHttp 的 `Interceptor` 可精确追踪下载进度，UI 实时更新
2. **镜像测速融合**：OkHttp 已在测速环节使用，复用连接池
3. **后台可靠性**：WorkManager 保证下载任务在后台可靠执行
4. **自定义通知**：可自定义下载进度通知，体验更好
5. **下载路径可控**：直接写入 `context.getExternalFilesDir()` 或 `cacheDir`，无需额外权限

### 4.3 OkHttp 下载实现要点

```kotlin
/**
 * OkHttp 下载 APK，支持进度回调和取消
 */
suspend fun downloadApk(
    url: String,
    destFile: File,
    onProgress: (Int) -> Unit
) {
    val client = OkHttpClient.Builder()
        .connectTimeout(15, TimeUnit.SECONDS)
        .readTimeout(60, TimeUnit.SECONDS)
        .build()

    val request = Request.Builder().url(url).build()
    val response = client.newCall(request).execute()

    if (!response.isSuccessful) throw IOException("Download failed: ${response.code}")

    val body = response.body ?: throw IOException("Empty response body")
    val totalBytes = body.contentLength()
    var downloadedBytes = 0L

    destFile.parentFile?.mkdirs()

    body.byteStream().use { input ->
        destFile.outputStream().use { output ->
            val buffer = ByteArray(8192)
            var bytesRead: Int
            while (input.read(buffer).also { bytesRead = it } != -1) {
                output.write(buffer, 0, bytesRead)
                downloadedBytes += bytesRead
                if (totalBytes > 0) {
                    val progress = ((downloadedBytes * 100) / totalBytes).toInt()
                    onProgress(progress)
                }
            }
            output.flush()
        }
    }
}
```

### 4.4 DownloadManager 方案（备选/简单场景）

```kotlin
/**
 * 使用系统 DownloadManager 下载 APK
 */
fun downloadWithDownloadManager(context: Context, url: String, fileName: String): Long {
    val request = DownloadManager.Request(Uri.parse(url))
        .setMimeType("application/vnd.android.package-archive")
        .setTitle("正在下载更新")
        .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE)
        .setDestinationInExternalFilesDir(context, Environment.DIRECTORY_DOWNLOADS, fileName)

    val dm = context.getSystemService(DownloadManager::class.java)
    return dm.enqueue(request)  // 返回 downloadId 用于追踪
}

// 监听下载完成
// AndroidManifest.xml: <receiver android:name=".DownloadCompleteReceiver">
//     <intent-filter><action android:name="android.intent.action.DOWNLOAD_COMPLETE"/></intent-filter>
// </receiver>
```

---

## 五、APK 安装 — 静默安装与用户确认安装的权限处理

### 5.1 Android 版本演进与关键权限

| Android 版本 | 关键变化 | 所需权限 |
|-------------|---------|---------|
| < 7.0 (API < 24) | 直接 `ACTION_INSTALL_PACKAGE` Intent 即可 | 无特殊权限 |
| 7.0+ (API 24+) | 需用 `FileProvider` 提供 APK URI | `REQUEST_INSTALL_PACKAGES` |
| 8.0+ (API 26+) | 强制要求 `REQUEST_INSTALL_PACKAGES` 权限 | `REQUEST_INSTALL_PACKAGES` |
| 10+ (API 29+) | 严格分区存储，APK 须存于应用专属目录 | 无额外存储权限 |
| 11+ (API 30+) | `PackageInstaller` API 成为主流推荐方式 | `REQUEST_INSTALL_PACKAGES` |
| 14+ (API 34+) | 部分行为收紧，`USER_ACTION_REQUIRED` 默认行为 | `REQUEST_INSTALL_PACKAGES` |

### 5.2 用户确认安装（标准方案，推荐）

**AndroidManifest.xml：**
```xml
<!-- 安装 APK 所需权限 -->
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />

<!-- FileProvider 用于分享 APK 文件 -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

**res/xml/file_paths.xml：**
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-files-path name="downloads" path="Download/" />
    <cache-path name="cache" path="/" />
</paths>
```

**触发安装 Intent：**
```kotlin
fun installApk(context: Context, apkFile: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        apkFile
    )

    val intent = Intent(Intent.ACTION_VIEW).apply {
        setDataAndType(uri, "application/vnd.android.package-archive")
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
    }

    // 检查是否有安装权限
    if (context.packageManager.canRequestPackageInstalls()) {
        context.startActivity(intent)
    } else {
        // 引导用户去设置开启"安装未知应用"权限
        val settingsIntent = Intent(
            Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES,
            Uri.parse("package:${context.packageName}")
        )
        settingsIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        context.startActivity(settingsIntent)
    }
}
```

### 5.3 PackageInstaller API（API 21+，更精细的安装控制）

```kotlin
/**
 * 使用 PackageInstaller 进行安装
 * 优点：获得安装状态回调，不需启动新的 Activity
 * 注意：仅有 REQUEST_INSTALL_PACKAGES 权限时，默认行为为 USER_ACTION_REQUIRED
 */
fun installWithPackageInstaller(
    context: Context,
    apkFile: File,
    // 安装完成后的回调
    onInstallResult: (Int) -> Unit
) {
    val packageInstaller = context.packageManager.packageInstaller
    val params = PackageInstaller.SessionParams(
        PackageInstaller.SessionParams.MODE_FULL_INSTALL
    )
    // 默认 USER_ACTION_REQUIRED，会弹出系统安装确认界面
    params.setOriginatingUid(Process.myUid())

    val sessionId = packageInstaller.createSession(params)
    val session = packageInstaller.openSession(sessionId)

    apkFile.inputStream().use { input ->
        session.openWrite("apk", 0, apkFile.length()).use { output ->
            input.copyTo(output)
            session.fsync(output)
        }
    }

    // PendingIntent 接收安装结果回调
    val intent = Intent(context, InstallResultReceiver::class.java)
    val pendingIntent = PendingIntent.getBroadcast(
        context,
        sessionId,
        intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
    )

    session.commit(pendingIntent.intentSender)
}
```

### 5.4 静默安装（需特殊权限，普通 App 不可用）

| 方案 | 所需条件 | 适用场景 |
|------|---------|---------|
| `INSTALL_PACKAGES` 权限 | 系统 App 或 /system/app 预装 | ROM 内置应用 |
| Shizuku / Root | 用户已 Root 或启动 Shizuku | 高级用户工具 |
| 设备管理器 (MDM/EMM) | 企业设备管理 | 企业部署 |
| Ackpine 库 (solrudev/Ackpine) | 支持 Shizuku 和 Root 后端 | 开源库，封装了多种安装方式 |

**关键限制：**
- **普通第三方 App 无法实现真正的静默安装**，`INSTALL_PACKAGES` 是 `signature|system` 级权限
- 仅有 `REQUEST_INSTALL_PACKAGES` 权限时，`PackageInstaller.SessionParams` 未指定时默认为 `USER_ACTION_REQUIRED`，即**必须弹出系统安装确认界面**
- Android 11+ (API 30+) 且仅有 `REQUEST_INSTALL_PACKAGES` 时，首先收到 `STATUS_PENDING_USER_ACTION` 状态码
- Google Play 政策：`REQUEST_INSTALL_PACKAGES` 权限要求 App 核心功能包含"发送/接收 App 包"且"启用用户主动安装"

### 5.5 推荐安装流程（用户体验最佳）

```
检测到新版本
  ↓
显示更新对话框（版本号 + 更新日志 + 下载大小）
  ↓ 用户点击"立即更新"
开始下载（带进度条/通知）
  ↓ 下载完成
检查安装权限 → 无权限 → 引导开启"安装未知应用"设置
  ↓ 有权限
弹出系统安装确认界面
  ↓ 用户确认安装
安装完成 → 可选：旧版本自动被新版本覆盖
```

---

## 六、整体架构推荐

```
┌─────────────────────────────────────────────────┐
│                  UpdateManager                  │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. 检查更新                                     │
│     ├─ GitHub Releases API (latest release)      │
│     ├─ Rate limit 缓存与退避                      │
│     └─ 版本号比较 (semantic versioning)           │
│                                                 │
│  2. 镜像测速与选择                                │
│     ├─ 多镜像源并发 HEAD 测速                      │
│     ├─ TTFB 排序选最快                            │
│     └─ 测速结果会话内缓存                          │
│                                                 │
│  3. 下载 APK                                    │
│     ├─ OkHttp + WorkManager 后台下载              │
│     ├─ 进度回调 → 通知栏 / 对话框进度条              │
│     ├─ 断点续传 (Range header)                    │
│     └─ 文件存于 app cacheDir / externalFilesDir   │
│                                                 │
│  4. 安装 APK                                    │
│     ├─ FileProvider + ACTION_VIEW (标准)          │
│     ├─ PackageInstaller (精细回调)                 │
│     ├─ 权限检查: canRequestPackageInstalls()       │
│     └─ 无权限时引导设置页面                         │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 推荐开源参考库

| 库名 | 地址 | 说明 |
|------|------|------|
| AppUpdater (javiersantos) | github.com/javiersantos/AppUpdater | 支持 GitHub/Play/Amazon/F-Droid 多源更新检查 |
| APKUpdater (rumboalla) | github.com/rumboalla/apkupdater | 多源聚合更新检查，Jetpack Compose 重写 |
| github_apk_updater (Flutter) | pub.dev/packages/github_apk_updater | Flutter 版 GitHub Releases 自动更新 |
| Ackpine (solrudev) | github.com/solrudev/Ackpine | Android 安装库，支持 Shizuku/Root 后端 |
| gh-proxy (hunshcn) | github.com/hunshcn/gh-proxy | GitHub 文件加速代理服务，可自部署 |
| GHProxy (NotNoneX) | github.com/NotNoneX/GHProxy | Go 实现的高性能 GitHub 代理加速 |

---

## 七、注意事项与坑点总结

1. **GitHub API 的 `browser_download_url` 会 302 重定向**：OkHttp 默认跟随重定向，但某些镜像代理可能不支持重定向，需测试验证
2. **APK 文件完整性校验**：下载完成后应校验文件大小（与 API 返回的 `assets[].size` 对比），有条件可做 MD5/SHA256 校验
3. **targetSdk 29+ 存储权限**：下载 APK 到 `context.getExternalFilesDir(null)` 不需要 `WRITE_EXTERNAL_STORAGE` 权限
4. **FileProvider 路径配置**：确保 `file_paths.xml` 中包含的实际路径与下载文件存储路径一致
5. **安装界面被覆盖**：安装 Intent 使用 `FLAG_ACTIVITY_NEW_TASK`，注意 App 可能被系统杀死（低内存场景），下载文件应存在持久化路径
6. **MIUI/EMUI 适配**：部分定制 ROM 的 `DownloadManager` 行为不一致，使用 OkHttp 方案可避免此类问题
7. **prerelease 过滤**：检查更新时应过滤 `prerelease == true` 的 Release，除非 App 明确需要测试版
8. **ghproxy 类镜像的可用性不稳定**：公共镜像站可能随时不可用，务必实现多源降级机制
9. **Android 14+ 变化**：`PackageInstaller.SessionParams` 未指定 `setInstallReason()` 时行为可能变化，需关注最新 API 文档
10. **Google Play 分发限制**：通过 Google Play 分发的 App 如果使用 `REQUEST_INSTALL_PACKAGES` 权限做自更新，可能违反 Play 政策导致下架；建议仅在非 Play 渠道（如 GitHub Releases 分发）使用
