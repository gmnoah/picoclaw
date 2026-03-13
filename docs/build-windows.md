# Windows 手动编译说明

在 Windows 下不依赖 `make` 或 `build.ps1` 时，可按以下步骤手动编译 picoclaw。

## 环境要求

- 已安装 [Go](https://go.dev/dl/)（建议 1.21+）
- 编译 **picoclaw-launcher** 时还需安装 [Node.js](https://nodejs.org/) 与 pnpm（见下方「四、编译 picoclaw-launcher」）
- 在项目根目录打开 PowerShell 或 CMD 执行以下命令

## 一、准备嵌入的 workspace

`cmd/picoclaw/internal/onboard` 通过 `//go:embed workspace` 嵌入根目录的 `workspace`，编译前必须先把 `workspace` 拷到该包目录下。**在项目根目录**执行：

```powershell
$onboard = "cmd\picoclaw\internal\onboard"
if (Test-Path "$onboard\workspace") { Remove-Item -Recurse -Force "$onboard\workspace" }
Copy-Item -Recurse -Force workspace "$onboard\workspace"
```

## 二、编译

### 方式 A：不执行 go generate（推荐）

已用上面的命令复制好 `workspace` 后，**无需**执行 `go generate`，直接编译即可：

```powershell
mkdir build -Force
go build -v -tags stdjson -ldflags "-s -w" -o build\picoclaw.exe .\cmd\picoclaw
```

### 方式 B：需要执行 go generate 时

若项目其他包也有 `go:generate` 且需要执行，可先让系统能找到 `cp`（来自 Git for Windows），再执行 generate：

```powershell
# 按实际安装路径修改，常见为 C:\Program Files\Git\usr\bin
$env:PATH = "C:\Program Files\Git\usr\bin;$env:PATH"
$env:CGO_ENABLED = "0"
go generate ./...
```

然后再执行上面的 `go build` 命令。

### 可选：写入版本信息

若希望二进制中包含版本、Commit、构建时间等信息：

```powershell
$v = git describe --tags --always --dirty 2>$null; if (-not $v) { $v = "dev" }
$c = git rev-parse --short=8 HEAD 2>$null; if (-not $c) { $c = "dev" }
$t = Get-Date -Format "yyyy-MM-ddTHH:mm:sszzz"
$g = (go version) -replace 'go version ', ''
$pkg = "github.com/sipeed/picoclaw/pkg/config"
go build -v -tags stdjson -ldflags "-X ${pkg}.Version=$v -X ${pkg}.GitCommit=$c -X ${pkg}.BuildTime=$t -X ${pkg}.GoVersion=$g -s -w" -o build\picoclaw.exe .\cmd\picoclaw
```

## 三、运行

```powershell
.\build\picoclaw.exe
```

## 四、编译 picoclaw-launcher（Web 控制台）

picoclaw-launcher 为带 Web 前端的控制台，需先构建前端再编译 Go 后端。

### 1. 安装 pnpm（若未安装）

```powershell
npm install -g pnpm
```

### 2. 构建前端并输出到 backend

在项目根目录下进入前端目录并构建：

```powershell
cd web\frontend
pnpm install
pnpm run build:backend
```

完成后会在 `web/backend/dist` 下生成前端静态资源。

### 3. 编译 Launcher 可执行文件

回到项目根目录后执行：

```powershell
go build -o build\picoclaw-launcher.exe .\web\backend
```

（若当前在 `web\frontend`，需先执行 `cd ..\..` 回到项目根目录。）

运行：

```powershell
.\build\picoclaw-launcher.exe
```

## 常见问题

| 现象 | 原因 | 处理 |
|------|------|------|
| `pattern workspace: no matching files found` | `cmd\picoclaw\internal\onboard\workspace` 不存在 | 先执行「一、准备嵌入的 workspace」中的复制命令 |
| `exec: "cp": executable file not found in %PATH%` | 执行了 `go generate ./...` 且系统无 `cp` | 采用方式 A 不执行 generate，或按方式 B 将 Git 的 `usr\bin` 加入 PATH |

## 简要流程汇总

```powershell
# 1. 复制 workspace
Copy-Item -Recurse -Force workspace cmd\picoclaw\internal\onboard\workspace

# 2. 编译
go build -v -tags stdjson -ldflags "-s -w" -o build\picoclaw.exe .\cmd\picoclaw

# 3. 运行
.\build\picoclaw.exe
```
