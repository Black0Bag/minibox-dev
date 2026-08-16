# Go 语言二进制自升级（Self-Update）方案研究报告

## 概述

Go 程序因编译为单一二进制文件，非常适合实现自升级。本报告调研了主流的自更新库、GitHub Release API 下载方案、二进制热替换/回滚模式、版本对比方案及跨平台注意事项。

---

## 一、主流自更新库对比

### 1. github.com/minio/selfupdate

- **GitHub**: https://github.com/minio/selfupdate
- **pkg.go.dev**: https://pkg.go.dev/github.com/minio/selfupdate
- **版本**: v0.6.0 (2022-10-19)
- **License**: Apache-2.0
- **被引用**: 119 个项目
- **来源**: fork 自 `github.com/inconshreveable/go-update`，为 MinIO 项目定制增强

#### 核心 API

| 函数 | 说明 |
|------|------|
| `Apply(update io.Reader, opts Options) error` | 核心函数：用 io.Reader 的内容替换当前可执行文件 |
| `CommitBinary(opts Options) error` | 将新二进制移动到当前可执行文件位置（v0.5.0+） |
| `PrepareAndCheckBinary(update io.Reader, opts Options) error` | 准备并校验新二进制（v0.5.0+，分两步更新的第一步） |
| `RollbackError(err error) error` | 从更新错误中提取回滚错误 |
| `NewBSDiffPatcher() Patcher` | 创建 bsdiff 补丁处理器 |
| `NewVerifier() *Verifier` | 创建签名验证器 |
| `(*Options).CheckPermissions() error` | 检查文件权限 |

#### Options 结构体关键字段

```go
type Options struct {
    TargetPath string    // 目标文件路径（默认当前可执行文件）
    Hash       crypto.Hash // 校验算法（默认 SHA256）
    Checksum   []byte    // 期望的校验和
    Patcher    Patcher   // 补丁处理器（可选）
    Verifier   *Verifier // 签名验证器（可选）
}
```

#### 代码示例：从 URL 更新

```go
import (
    "fmt"
    "net/http"
    "github.com/minio/selfupdate"
)

func doUpdate(url string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    err = selfupdate.Apply(resp.Body, selfupdate.Options{})
    if err != nil {
        // 回滚失败处理
        if rerr := selfupdate.RollbackError(err); rerr != nil {
            fmt.Printf("Failed to rollback from bad update: %v", rerr)
        }
    }
    return err
}
```

#### 代码示例：带校验和验证的更新

```go
import (
    "crypto"
    _ "crypto/sha256"
    "encoding/hex"
    "io"
    "github.com/minio/selfupdate"
)

func updateWithChecksum(binary io.Reader, hexChecksum string) error {
    checksum, err := hex.DecodeString(hexChecksum)
    if err != nil {
        return err
    }
    err = selfupdate.Apply(binary, selfupdate.Options{
        Hash:     crypto.SHA256, // 默认值
        Checksum: checksum,
    })
    return err
}
```

#### 代码示例：bsdiff 二进制补丁

```go
func updateWithPatch(patch io.Reader) error {
    err := selfupdate.Apply(patch, selfupdate.Options{
        Patcher: selfupdate.NewBSDiffPatcher(),
    })
    return err
}
```

#### 回滚机制

minio/selfupdate 的 `Apply` 内部实现了两阶段提交：
1. **PrepareAndCheckBinary**：将新二进制写入临时文件，进行校验和/签名验证
2. **CommitBinary**：将旧二进制重命名为备份，将新二进制重命名到目标位置

如果第二步失败，`RollbackError()` 会尝试恢复旧二进制。但文档指出：**回滚不是 100% 保证成功**，应用需要检查错误并通知用户手动恢复。

#### 特性

- ✅ 跨平台支持（含 Windows）
- ✅ 二进制补丁（bsdiff）
- ✅ 校验和验证（SHA256 默认，可插拔）
- ✅ 代码签名验证
- ✅ 支持更新任意文件（通过 `TargetPath`）
- ✅ 分两步更新（Prepare → Commit），支持更灵活的错误处理

---

### 2. github.com/inconshreveable/go-update

- **GitHub**: https://github.com/inconshreveable/go-update
- **pkg.go.dev**: https://pkg.go.dev/github.com/inconshreveable/go-update
- **版本**: v0.0.0-...-8152e7e (2016-01-12，已停止维护)
- **License**: Apache-2.0
- **被引用**: 446 个项目（历史最多）
- **地位**: 原始库，minio/selfupdate 的上游

#### 核心 API

| 函数 | 说明 |
|------|------|
| `Apply(update io.Reader, opts Options) error` | 核心更新函数 |
| `RollbackError(err error) error` | 回滚错误处理 |
| `(*Options).CheckPermissions() error` | 权限检查 |
| `(*Options).SetPublicKeyPEM(pembytes)` | 设置 PEM 公钥 |
| `NewBSDiffPatcher() Patcher` | bsdiff 补丁 |
| `NewDSAVerifier() Verifier` | DSA 签名验证 |
| `NewECDSAVerifier() Verifier` | ECDSA 签名验证 |
| `NewRSAVerifier() Verifier` | RSA 签名验证 |

#### 代码示例

```go
import (
    "fmt"
    "net/http"
    "github.com/inconshreveable/go-update"
)

func doUpdate(url string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    err = update.Apply(resp.Body, update.Options{})
    if err != nil {
        if rerr := update.RollbackError(err); rerr != nil {
            fmt.Printf("Failed to rollback from bad update: %v", rerr)
        }
    }
    return err
}
```

#### 与 minio/selfupdate 的区别

| 特性 | go-update | minio/selfupdate |
|------|-----------|------------------|
| 维护状态 | ❌ 已停止（2016） | ✅ 活跃 |
| PrepareAndCheckBinary | ❌ | ✅ |
| CommitBinary | ❌ | ✅ |
| 签名验证类型 | DSA/ECDSA/RSA | 通用 Verifier 接口 |
| 被引用数 | 446 | 119 |
| Windows 支持 | ✅ | ✅ |

#### 回滚机制

与 minio 版本相同：`Apply` 内部写入临时文件 → 重命名替换，失败时尝试恢复旧文件。`RollbackError()` 用于提取回滚过程中的错误。

#### Equinox.io

go-update 上游有 Equinox.io（https://equinox.io）作为完整解决方案，提供：
- 托管更新服务
- 更新通道（stable/beta/nightly）
- 动态二进制 diff
- 自动密钥生成和代码签名
- 更新/下载指标

> ⚠️ Equinox.io 服务已停止运营。

---

### 3. github.com/creativeprojects/go-selfupdate

- **GitHub**: https://github.com/creativeprojects/go-selfupdate
- **pkg.go.dev**: https://pkg.go.dev/github.com/creativeprojects/go-selfupdate
- **版本**: v1.6.0
- **License**: MIT
- **被引用**: 84 个项目
- **来源**: fork 自 `github.com/rhysd/go-github-selfupdate`（后者又基于 go-update）
- **定位**: 功能最完整的自更新方案，内置 GitHub/GitLab/Gitea/HTTP 源支持

#### 核心 API

| 函数/方法 | 说明 |
|-----------|------|
| `DetectLatest(ctx, repository) (*Release, bool, error)` | 检测最新版本 |
| `DetectVersion(ctx, repository, version) (*Release, bool, error)` | 检测指定版本 |
| `UpdateSelf(ctx, current, repository) (*Release, error)` | 更新自身 |
| `UpdateCommand(ctx, cmdPath, current, repository) (*Release, error)` | 更新指定命令 |
| `UpdateTo(ctx, assetURL, assetFileName, cmdPath) error` | 下载并替换指定路径的二进制 |
| `ExecutablePath() (string, error)` | 获取当前可执行文件路径 |
| `DecompressCommand(src, url, cmd, os, arch) (io.Reader, error)` | 解压下载的资产 |
| `ParseSlug(slug) RepositorySlug` | 解析 `owner/repo` 格式 |
| `NewRepositorySlug(owner, repo) RepositorySlug` | 创建仓库标识 |

#### Release 类型方法（版本比较）

```go
func (r Release) LessThan(other string) bool
func (r Release) LessOrEqual(other string) bool
func (r Release) Equal(other string) bool
func (r Release) GreaterThan(other string) bool
func (r Release) GreaterOrEqual(other string) bool
func (r Release) Version() string
```

#### 代码示例：完整的自更新流程

```go
package main

import (
    "context"
    "fmt"
    "log"
    "runtime"

    "github.com/creativeprojects/go-selfupdate"
)

func update(version string) error {
    // 1. 检测最新版本
    latest, found, err := selfupdate.DetectLatest(
        context.Background(),
        selfupdate.ParseSlug("creativeprojects/resticprofile"),
    )
    if err != nil {
        return fmt.Errorf("error occurred while detecting version: %w", err)
    }
    if !found {
        return fmt.Errorf("latest version for %s/%s could not be found",
            runtime.GOOS, runtime.GOARCH)
    }

    // 2. 版本比较
    if latest.LessOrEqual(version) {
        log.Printf("Current version (%s) is the latest", version)
        return nil
    }

    // 3. 获取当前可执行文件路径
    exe, err := selfupdate.ExecutablePath()
    if err != nil {
        return errors.New("could not locate executable path")
    }

    // 4. 下载并替换
    if err := selfupdate.UpdateTo(
        context.Background(),
        latest.AssetURL,
        latest.AssetName,
        exe,
    ); err != nil {
        return fmt.Errorf("error occurred while updating binary: %w", err)
    }

    log.Printf("Successfully updated to version %s", latest.Version())
    return nil
}
```

#### 代码示例：使用 Updater 自定义配置

```go
updater, err := selfupdate.NewUpdater(selfupdate.Config{
    // 使用 goreleaser 的统一 checksum 文件
    Validator: &selfupdate.ChecksumValidator{
        UniqueFilename: "checksums.txt",
    },
})
if err != nil {
    log.Fatal(err)
}

release, found, err := updater.DetectLatest(
    context.Background(),
    selfupdate.NewRepositorySlug("owner", "repo"),
)
if err != nil {
    log.Fatal(err)
}
if !found {
    log.Print("Release not found")
    return
}

exe, _ := selfupdate.ExecutablePath()
err = updater.UpdateTo(context.Background(), release, exe)
```

#### 代码示例：GitLab 私有实例

```go
source, _ := selfupdate.NewGitLabSource(selfupdate.GitLabConfig{
    BaseURL: "https://private.instance.on.gitlab.com/",
})
updater, _ := selfupdate.NewUpdater(selfupdate.Config{
    Source:   source,
    Validator: &selfupdate.ChecksumValidator{UniqueFilename: "checksums.txt"},
})
release, found, _ := updater.DetectLatest(
    context.Background(),
    selfupdate.NewRepositorySlug("owner", "cli-tool"),
)
```

#### 代码示例：HTTP 自托管

```go
source, _ := selfupdate.NewHttpSource(selfupdate.HttpConfig{
    BaseURL: "https://example.com/",
})
updater, _ := selfupdate.NewUpdater(selfupdate.Config{
    Source:   source,
    Validator: &selfupdate.ChecksumValidator{UniqueFilename: "checksums.txt"},
})
```

#### 支持的源提供者

| 提供者 | 说明 |
|--------|------|
| GitHub | 默认，通过 GitHub Releases API |
| GitLab | 支持私有实例，需 Generic Package Registry |
| Gitea | 支持自托管 Gitea |
| HTTP | 通用 HTTP 源，配合 goreleaser-http-repo-builder |

#### 命名规则

发布二进制文件必须遵循格式：`{cmd}_{goos}_{goarch}{.ext}`

示例（cmd=foo-bar, linux/amd64）：
- `foo-bar_linux_amd64`（原始二进制）
- `foo-bar_linux_amd64.zip`
- `foo-bar_linux_amd64.tar.gz`
- `foo-bar_linux_amd64.xz`
- `foo-bar-linux-amd64.tar.gz`（也支持 `-` 分隔符）

Windows: `foo-bar_windows_amd64.exe.zip`

支持的压缩格式：`.zip`, `.gzip`, `.bz2`, `.tar.gz`, `.tar.xz`

#### 版本标签规则

- 使用 Git tag 名称（非 Release 标题）
- 支持语义版本号：`1.2.3` 或 `v1.2.3`
- 前缀自动去除：`ver1.2.3`、`release-1.2.3` 也可
- 不含版本号的 tag 被忽略（如 `nightly`）
- Pre-release 被忽略

#### 回滚机制

go-selfupdate 内部使用 go-update 的替换机制（下载到临时文件 → 重命名替换），失败时自动回滚。文档明确指出："Update the binary with rollback support on failure"。

---

## 二、GitHub Release API 下载最新二进制的 Go 实现

### GitHub Releases REST API

```
# 获取最新 release
GET https://api.github.com/repos/{owner}/{repo}/releases/latest

# 获取所有 releases
GET https://api.github.com/repos/{owner}/{repo}/releases

# 获取指定版本
GET https://api.github.com/repos/{owner}/{repo}/releases/tags/{tag}
```

### 原生 Go 实现（不使用第三方库）

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "runtime"
)

type GitHubRelease struct {
    TagName  string  `json:"tag_name"`
    Name     string  `json:"name"`
    Assets   []Asset `json:"assets"`
}

type Asset struct {
    Name               string `json:"name"`
    BrowserDownloadURL string `json:"browser_download_url"`
    Size               int    `json:"size"`
}

func getLatestRelease(owner, repo string) (*GitHubRelease, error) {
    url := fmt.Sprintf("https://api.github.com/repos/%s/%s/releases/latest", owner, repo)

    req, err := http.NewRequestWithContext(context.Background(), "GET", url, nil)
    if err != nil {
        return nil, err
    }
    req.Header.Set("Accept", "application/vnd.github+json")
    // 私有仓库需要 token
    // req.Header.Set("Authorization", "Bearer "+token)

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("GitHub API returned status %d", resp.StatusCode)
    }

    var release GitHubRelease
    if err := json.NewDecoder(resp.Body).Decode(&release); err != nil {
        return nil, err
    }
    return &release, nil
}

func downloadAsset(url, destPath string) error {
    resp, err := http.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    out, err := os.Create(destPath)
    if err != nil {
        return err
    }
    defer out.Close()

    _, err = io.Copy(out, resp.Body)
    return err
}

// 选择匹配当前平台的 asset
func selectAsset(assets []Asset) (*Asset, error) {
    goos := runtime.GOOS   // "linux", "windows", "darwin"
    goarch := runtime.GOARCH // "amd64", "arm64", "arm", "386"

    for i := range assets {
        name := assets[i].Name
        if strings.Contains(name, goos) && strings.Contains(name, goarch) {
            return &assets[i], nil
        }
    }
    return nil, fmt.Errorf("no matching asset for %s/%s", goos, goarch)
}
```

### 使用 google/go-github 库

```go
import "github.com/google/go-github/v62/github"

func getLatestRelease(client *github.Client, owner, repo string) (*github.RepositoryRelease, error) {
    release, _, err := client.Repositories.GetLatestRelease(context.Background(), owner, repo)
    return release, err
}

func downloadAsset(client *github.Client, owner, repo string, assetID int64) (io.ReadCloser, error) {
    // 对于私有仓库，需要使用 API 下载（重定向到 CDN）
    body, _, err := client.Repositories.DownloadReleaseAsset(
        context.Background(), owner, repo, assetID,
        http.DefaultClient,
    )
    return body, err
}
```

---

## 三、二进制热替换/备份/回滚的 Shell 脚本模式

### 模式 1：经典备份-替换-回滚脚本

```bash
#!/bin/bash
set -euo pipefail

BINARY="/usr/local/bin/myapp"
BACKUP="${BINARY}.bak"
NEW_BINARY="/tmp/myapp_new"
DOWNLOAD_URL="https://github.com/owner/repo/releases/latest/download/myapp_linux_amd64"

# 1. 下载新版本
echo "Downloading new version..."
if ! curl -fSL -o "$NEW_BINARY" "$DOWNLOAD_URL"; then
    echo "ERROR: Download failed"
    exit 1
fi

# 2. 设置可执行权限
chmod +x "$NEW_BINARY"

# 3. 可选：校验 SHA256
EXPECTED_SHA256="abc123..."  # 从 checksums.txt 获取
ACTUAL_SHA256=$(sha256sum "$NEW_BINARY" | awk '{print $1}')
if [ "$EXPECTED_SHA256" != "$ACTUAL_SHA256" ]; then
    echo "ERROR: Checksum mismatch"
    rm -f "$NEW_BINARY"
    exit 1
fi

# 4. 备份当前版本
echo "Backing up current version..."
cp "$BINARY" "$BACKUP"

# 5. 原子替换（使用 mv 实现原子操作）
echo "Replacing binary..."
mv "$NEW_BINARY" "$BINARY"

# 6. 验证新版本
echo "Verifying new version..."
if "$BINARY" --version; then
    echo "Update successful!"
    rm -f "$BACKUP"
else
    echo "ERROR: New version verification failed, rolling back..."
    mv "$BACKUP" "$BINARY"
    exit 1
fi
```

### 模式 2：符号链接切换（推荐用于 systemd 服务）

```bash
#!/bin/bash
set -euo pipefail

APP_NAME="myapp"
INSTALL_DIR="/opt/${APP_NAME}"
SYMLINK="/usr/local/bin/${APP_NAME}"
CURRENT_VERSION=$("${SYMLINK}" --version 2>/dev/null || echo "unknown")

# 1. 下载新版本到版本化目录
NEW_VERSION="1.2.3"
NEW_DIR="${INSTALL_DIR}/${NEW_VERSION}"
mkdir -p "$NEW_DIR"
curl -fSL -o "${NEW_DIR}/${APP_NAME}" "https://github.com/owner/repo/releases/download/v${NEW_VERSION}/${APP_NAME}_linux_amd64"
chmod +x "${NEW_DIR}/${APP_NAME}"

# 2. 原子切换符号链接
ln -sfn "${NEW_DIR}/${APP_NAME}" "${SYMLINK}.tmp"
mv -Tf "${SYMLINK}.tmp" "$SYMLINK"

# 3. 重启服务
systemctl restart "$APP_NAME"

# 4. 验证（可配合 systemd watchdog 或健康检查）
sleep 5
if systemctl is-active --quiet "$APP_NAME"; then
    echo "Update successful"
    # 保留上一个版本用于回滚
    PREV_DIR="${INSTALL_DIR}/${CURRENT_VERSION}"
    if [ -d "$PREV_DIR" ]; then
        echo "Previous version kept at $PREV_DIR for rollback"
    fi
else
    echo "ERROR: Service not running, rolling back..."
    ln -sfn "${INSTALL_DIR}/${CURRENT_VERSION}/${APP_NAME}" "${SYMLINK}.tmp"
    mv -Tf "${SYMLINK}.tmp" "$SYMLINK"
    systemctl restart "$APP_NAME"
fi
```

### 模式 3：Go 程序内的自替换逻辑（不使用第三方库）

```go
package selfupdate

import (
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
)

func SelfUpdate(downloadURL string) error {
    // 1. 获取当前可执行文件路径
    exePath, err := os.Executable()
    if err != nil {
        return fmt.Errorf("cannot find executable: %w", err)
    }
    exePath, _ = filepath.EvalSymlinks(exePath)

    // 2. 下载新二进制到临时文件（同目录，确保同文件系统可原子 rename）
    dir := filepath.Dir(exePath)
    tmpFile, err := os.CreateTemp(dir, ".update-*")
    if err != nil {
        return err
    }
    tmpPath := tmpFile.Name()
    defer os.Remove(tmpPath) // 清理临时文件

    // 3. 下载
    resp, err := http.Get(downloadURL)
    if err != nil {
        tmpFile.Close()
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        tmpFile.Close()
        return fmt.Errorf("download failed: HTTP %d", resp.StatusCode)
    }

    _, err = io.Copy(tmpFile, resp.Body)
    tmpFile.Close()
    if err != nil {
        return err
    }

    // 4. 设置可执行权限
    if err := os.Chmod(tmpPath, 0755); err != nil {
        return err
    }

    // 5. 备份旧版本
    backupPath := exePath + ".bak"
    if err := os.Rename(exePath, backupPath); err != nil {
        return fmt.Errorf("backup failed: %w", err)
    }

    // 6. 原子替换
    if err := os.Rename(tmpPath, exePath); err != nil {
        // 回滚
        os.Rename(backupPath, exePath)
        return fmt.Errorf("replace failed, rolled back: %w", err)
    }

    // 7. 清理备份
    os.Remove(backupPath)

    return nil
}
```

### 关键设计原则

| 原则 | 说明 |
|------|------|
| **原子替换** | 使用 `os.Rename`（Linux/Unix 上的 rename 是原子操作）|
| **同目录临时文件** | 临时文件必须在同一文件系统，否则 rename 跨文件系统会失败 |
| **先备份后替换** | `cp current → backup` 然后 `mv new → current` |
| **验证后清理** | 新版本验证通过后才删除备份 |
| **符号链接模式** | 版本化目录 + 符号链接，可保留多版本，回滚即切换链接 |

---

## 四、版本检测与远程版本对比方案

### 方案 A：使用 go-selfupdate 的内置版本比较

```go
import "github.com/creativeprojects/go-selfupdate"

func checkAndUpdate(currentVersion string) error {
    latest, found, err := selfupdate.DetectLatest(
        context.Background(),
        selfupdate.ParseSlug("owner/repo"),
    )
    if err != nil {
        return err
    }
    if !found {
        return fmt.Errorf("release not found")
    }

    // 内置 semver 比较
    if latest.LessOrEqual(currentVersion) {
        log.Printf("Already up to date (current: %s)", currentVersion)
        return nil
    }

    log.Printf("Update available: %s → %s", currentVersion, latest.Version())
    // 执行更新...
    return nil
}
```

### 方案 B：使用 semver 库手动比较

```go
import "golang.org/x/mod/semver"

func compareVersions(current, remote string) bool {
    // 确保有 v 前缀
    if !strings.HasPrefix(current, "v") {
        current = "v" + current
    }
    if !strings.HasPrefix(remote, "v") {
        remote = "v" + remote
    }

    // semver.Compare 返回 -1/0/1
    return semver.Compare(remote, current) > 0
}
```

### 方案 C：嵌入版本信息（编译时注入）

```go
// version.go
package main

var (
    Version   = "dev"        // 通过 -ldflags 注入
    BuildTime = "unknown"    // 通过 -ldflags 注入
    Commit    = "none"       // 通过 -ldflags 注入
)

// 编译命令：
// go build -ldflags "-X main.Version=1.2.3 -X main.BuildTime=$(date -u +%Y%m%d%H%M%S) -X main.Commit=$(git rev-parse --short HEAD)" -o myapp
```

### 方案 D：通过 HTTP 端点检测版本

```go
type VersionInfo struct {
    Version string `json:"version"`
    URL     string `json:"url"`
    SHA256  string `json:"sha256"`
}

func checkRemoteVersion(endpoint string) (*VersionInfo, error) {
    resp, err := http.Get(endpoint + "/latest-version")
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var info VersionInfo
    if err := json.NewDecoder(resp.Body).Decode(&info); err != nil {
        return nil, err
    }
    return &info, nil
}

func needsUpdate(current, remote string) bool {
    cv := semver.Canonical("v" + strings.TrimPrefix(current, "v"))
    rv := semver.Canonical("v" + strings.TrimPrefix(remote, "v"))
    return semver.Compare(rv, cv) > 0
}
```

### 方案 E：使用 goreleaser 的 checksums.txt

goreleaser 生成的 `checksums.txt` 格式：
```
a1b2c3d4...  myapp_linux_amd64.tar.gz
e5f6g7h8...  myapp_linux_arm64.tar.gz
i9j0k1l2...  myapp_windows_amd64.zip
```

```go
func verifyChecksum(binaryData []byte, expectedHash, filename string, checksumsContent string) error {
    // 解析 checksums 文件
    for _, line := range strings.Split(checksumsContent, "\n") {
        parts := strings.Fields(line)
        if len(parts) == 2 && parts[1] == filename {
            actualHash := fmt.Sprintf("%x", sha256.Sum256(binaryData))
            if actualHash != parts[0] {
                return fmt.Errorf("checksum mismatch for %s", filename)
            }
            return nil
        }
    }
    return fmt.Errorf("checksum not found for %s", filename)
}
```

---

## 五、跨平台（Linux ARM/x86/Windows）自升级注意事项

### 1. 文件替换策略差异

| 平台 | 替换方式 | 注意事项 |
|------|----------|----------|
| **Linux/macOS** | `os.Rename()` 原子替换 | ✅ rename 在同一文件系统内是原子操作 |
| **Windows** | `MoveFileEx` with `MOVEFILE_REPLACE_EXISTING` | ⚠️ 正在运行的 .exe 文件不能直接覆盖 |

#### Windows 特殊处理

Windows 上正在运行的二进制文件被锁定，无法直接覆盖。常见解决方案：

```go
// 方案 1：重命名当前运行的 exe（Windows 允许重命名运行中的文件）
os.Rename(exePath, exePath+".old")
// 写入新文件到原路径
os.Rename(newFile, exePath)
// 删除旧文件（下次重启时删除，因为当前仍被锁定）
// 或使用 MoveFileEx with MOVEFILE_DELAY_UNTIL_REBOOT
```

```go
// 方案 2：使用 Windows API 延迟删除
//go:build windows
package selfupdate

import (
    "golang.org/x/sys/windows"
)

func deleteOnReboot(path string) error {
    p, err := windows.UTF16PtrFromString(path)
    if err != nil {
        return err
    }
    return windows.MoveFileEx(p, nil, windows.MOVEFILE_DELAY_UNTIL_REBOOT)
}
```

minio/selfupdate 和 go-update 都已内置处理 Windows 的文件锁定问题。

### 2. ARM 架构处理

go-selfupdate 对 ARM 有特殊处理：

```
ARM 架构搜索顺序（以 armv6 为例）：
1. armv6  ← 精确匹配
2. armv5  ← 向下兼容
3. arm    ← 最后回退
```

**关键点**：
- 检测的是**编译时的目标架构**（`runtime.GOARCH` + `GOARM`），不是硬件架构
- 在 armv7 CPU 上运行 armv6 编译的二进制，会按 armv6 搜索更新
- goreleaser 生成 ARM 二进制时使用 `armv5`/`armv6`/`armv7` 作为名称

```go
// 获取 ARM 版本
// Go 1.18+: 使用 runtime.GOARM
// 但 runtime.GOARM 在非 arm 架构上为空
```

### 3. macOS 通用二进制（Universal Binary）

go-selfupdate 支持 macOS 通用二进制回退：

```go
updater, _ := selfupdate.NewUpdater(selfupdate.Config{
    UniversalArch: "all", // 当原生架构未找到时，回退到 universal binary
})
```

### 4. 文件权限

| 平台 | 权限要求 |
|------|----------|
| Linux/macOS | 需要对目标目录有写权限；更新后保持可执行权限 (0755) |
| Windows | 不需要 Unix 式权限，但可能需要管理员权限（取决于安装位置） |

```go
// minio/selfupdate 提供 CheckPermissions 方法
opts := selfupdate.Options{}
if err := opts.CheckPermissions(); err != nil {
    // 权限不足，可能需要 sudo 或管理员权限
}
```

### 5. 进程重启

更新完成后通常需要重启进程：

```go
// Linux: 使用 systemd 自动重启或 execve 重启
// 方案 1：退出进程，让 systemd 自动拉起
os.Exit(0)

// 方案 2：使用 syscall.Exec 原地重启（仅 Linux）
func restart() error {
    exe, _ := os.Executable()
    return syscall.Exec(exe, os.Args, os.Environ())
}

// Windows: 需要启动新进程后退出旧进程
func restartWindows() error {
    exe, _ := os.Executable()
    cmd := exec.Command(exe, os.Args[1:]...)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.Start()
    os.Exit(0)
    return nil
}
```

### 6. 平台识别与 Asset 匹配

```go
func platformAssetName(cmd string) string {
    goos := runtime.GOOS   // linux, windows, darwin
    goarch := runtime.GOARCH // amd64, arm64, arm, 386

    ext := ""
    if goos == "windows" {
        ext = ".exe"
    }

    // ARM 特殊处理
    arch := goarch
    if goarch == "arm" {
        // 需要 GOARM 环境变量或编译时信息
        arch = "arm" // 或 armv6, armv7 等
    }

    return fmt.Sprintf("%s_%s_%s%s", cmd, goos, arch, ext)
}
```

### 7. systemd 集成最佳实践

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=always
RestartSec=5
WatchdogSec=30  # 看门狗：30秒无心跳则重启

# 安全限制
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/myapp /var/lib/myapp

[Install]
WantedBy=multi-user.target
```

配合符号链接更新策略，systemd 会在 `Restart=always` 下自动使用新二进制重启。

---

## 六、方案选型建议

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| **使用 GitHub Releases 发布** | creativeprojects/go-selfupdate | 内置 GitHub API 集成、版本比较、ARM 支持 |
| **需要最大控制和灵活性** | minio/selfupdate | 活跃维护、分两步更新、签名验证 |
| **简单场景，少量代码** | 自实现 + os.Rename | 无外部依赖，~50 行代码 |
| **systemd 服务管理** | 符号链接切换 + systemd | 保留多版本，回滚秒级 |
| **Windows 为主** | minio/selfupdate 或 go-update | 内置 Windows 文件锁定处理 |
| **需要私有仓库支持** | creativeprojects/go-selfupdate | 支持 GitHub Token、GitLab、Gitea、HTTP |
| **需要 bsdiff 增量补丁** | minio/selfupdate | 内置 NewBSDiffPatcher |

---

## 七、相关项目链接

| 项目 | 地址 | 说明 |
|------|------|------|
| minio/selfupdate | https://github.com/minio/selfupdate | go-update 的活跃 fork |
| inconshreveable/go-update | https://github.com/inconshreveable/go-update | 原始库（已停止维护） |
| creativeprojects/go-selfupdate | https://github.com/creativeprojects/go-selfupdate | 最完整的自更新方案 |
| rhysd/go-github-selfupdate | https://github.com/rhysd/go-github-selfupdate | creativeprojects 的上游 fork |
| google/go-github | https://github.com/google/go-github | GitHub API Go 客户端 |
| goreleaser | https://goreleaser.com | 构建和发布工具，生成符合命名规则的二进制 |
| flynn/go-tuf | https://github.com/flynn/go-tuf | TUF（The Update Framework）实现 |
| equinox.io | https://equinox.io | 已停止运营的托管更新服务 |
| jteeuwen/go-bindata | https://github.com/jteeuwen/go-bindata | 将静态资源嵌入 Go 二进制 |

---

## 八、安全注意事项

1. **始终验证下载的二进制**：使用 SHA256 校验和 + 代码签名
2. **使用 HTTPS**：所有下载必须通过 HTTPS
3. **验证签名**：minio/selfupdate 支持 Verifier，go-selfupdate 支持 SHA256/ECDSA/PGP
4. **私有仓库认证**：使用 GitHub Token / GitLab Token，不要硬编码
5. **权限最小化**：更新过程中不要以 root 运行，除非必要
6. **防止降级攻击**：版本比较应拒绝旧版本（除非显式允许降级）
7. **临时文件安全**：临时文件应创建在受保护目录，权限设置为 0600
8. **考虑使用 TUF**：对于高安全需求，使用 The Update Framework (go-tuf)
