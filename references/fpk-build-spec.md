# FPK 构建规范（目录结构 / manifest / 环境变量 / 生命周期脚本）

> manifest 与生命周期的官方权威契约以 fn-fpk-builder/references/official-contract.md 为准（若该技能可用）；本页为独立可用的速查版。

## 三、FPK 构建规范（完整版）

### 3.1 开发目录结构

```
my-app-fpk/
├── manifest                          # [必填] key=value（等号前后无空格）
├── cmd/                              # [必填] 生命周期脚本（chmod +x）
│   ├── main                          #   [必填] start/stop/status
│   ├── install_init / install_callback      # [可选] 安装前/后
│   ├── uninstall_init / uninstall_callback  # [可选] 卸载前/后
│   ├── upgrade_init / upgrade_callback      # [可选] 更新前/后
│   └── config_init / config_callback        # [可选] 配置前/后
├── config/                           # [可选] 权限与资源
│   ├── privilege                     #       权限声明（JSON）
│   └── resource                      #       资源声明（JSON）
├── wizard/                           # [可选] 向导 JSON
│   ├── install / uninstall / upgrade / config
├── app/                              # [必填] 应用文件 → target/
│   ├── docker-compose.yml
│   ├── ui/                           # [可选] 桌面图标 Web UI
│   │   ├── config
│   │   └── images/ (icon-64.png, icon-256.png)
│   └── ...
├── ICON.PNG                          # [必填] 64x64
├── ICON_256.PNG                      # [推荐] 256x256
└── .prerun.sh                        # [可选] 构建前脚本
```

### 3.2 开发目录 → 安装目录映射

| 开发时 | 安装后 | 环境变量 | 说明 |
|--------|--------|---------|------|
| `app/` | `target/` | `TRIM_APPDEST` | 可执行文件、UI 资源 |
| `cmd/` | `cmd/` | — | 生命周期脚本 |
| `config/` | `config/` | — | privilege + resource |
| `wizard/` | `wizard/` | — | 向导定义 |
| `manifest` | `manifest` | — | 应用元信息 |
| — | `var/` | `TRIM_PKGVAR` | 运行时数据（系统创建） |
| — | `etc/` | `TRIM_PKGETC` | 静态配置（系统创建） |
| — | `home/` | `TRIM_PKGHOME` | 用户数据（系统创建） |
| — | `tmp/` | `TRIM_PKGTMP` | 临时文件（系统创建） |
| — | `meta/` | `TRIM_PKGMETA` | 元数据（系统创建） |

> ⚠️ **关键**：`app/ → target/`，`ui/` 必须放 `app/ui/` 下！放在根目录会导致 fnpack 打包报 "no such file or directory"。

### 3.3 manifest 完整字段

```ini
# === 必填 ===
appname=com.example.myapp                # 唯一标识（全局唯一，必填）
version=1.0.0                            # 版本号 x[.y[.z]][-build]
display_name=我的应用                      # 显示名
desc=应用描述                              # 支持HTML，建议加"首次使用看日志"
source=thirdparty                        # [固定值]
maintainer=开发者                         # 维护者名
distributor=发布者                        # 发布者名

# === 架构 ===
platform=x86                             # x86 | arm | all（替代废弃的 arch）

# === 端口与服务 ===
service_port=8080                        # 服务端口
checkport=true                           # 启动前检查端口占用

# === 安装 ===
install_type=                            # 空=用户选位置；root=系统分区
install_dep_apps=mariaDB:redis           # 依赖应用（冒号分隔）
os_min_version=0.9.0                     # 最低系统版本

# === 桌面图标 ===
desktop_uidir=ui                         # UI目录（相对target）
desktop_applaunchname=com.example.myapp.main  # 入口ID

# === 控制 ===
ctl_stop=true                            # 显示启停按钮

# === 包构建 ===
checksum=xxxxxxxx                        # app.tgz的CRC32
tar_size=12345                           # app.tgz字节数
changelog=修复已知问题                      # 更新日志
```

### 3.4 完整环境变量表

**路径变量**（cmd/ 脚本中可用）：

| 变量 | 说明 | 典型值 |
|------|------|--------|
| `TRIM_APPDEST` | 应用可执行文件目录 | `/vol[x]/@appcenter/[appname]` |
| `TRIM_PKGVAR` | 运行时数据目录 | `/vol[x]/@appdata/[appname]` |
| `TRIM_PKGETC` | 静态配置目录 | `/vol[x]/@appconf/[appname]` |
| `TRIM_PKGHOME` | 用户数据目录 | `/vol[x]/@apphome/[appname]` |
| `TRIM_PKGTMP` | 临时文件目录 | `/vol[x]/@apptemp/[appname]` |
| `TRIM_PKGMETA` | 应用元数据目录 | `/vol[x]/@appmeta/[appname]` |

**信息变量**：

| 变量 | 说明 |
|------|------|
| `TRIM_APPNAME` | 应用名称（来自 manifest appname） |
| `TRIM_APPVER` | 版本号 |
| `TRIM_OLD_APPVER` | 升级前版本号（仅 upgrade 脚本） |
| `TRIM_SERVICE_PORT` | manifest 声明的服务端口 |
| `TRIM_USERNAME` / `TRIM_GROUPNAME` | 应用专用用户/组 |
| `TRIM_RUN_USERNAME` | 实际运行用户 |
| `TRIM_TEMP_LOGFILE` | 错误日志路径（写入后系统以弹窗展示） |
| `TRIM_PKG_ARCH` | 当前安装架构 |
| `wizard_*` | 向导表单值（仅 callback 脚本） |

### 3.5 fnpack 下载与安装

```bash
# 方式一：手动下载
# 官方地址：https://developer.fnnas.com/docs/cli/fnpack
# 下载后可执行 chmod +x fnpack && sudo mv fnpack /usr/local/bin/

# 方式二：通过 fn-fpk-builder-skill 自动安装（推荐）
SKILL_DIR=/path/to/skills/fn-fpk-builder
python3 "$SKILL_DIR/scripts/fpk.py" toolchain --install --json
```

> **SKILL_DIR 常见位置**：
> - QwenPaw: `skills/fn-fpk-builder/`（在当前 workspace 下）
> - Codex: `$HOME/.codex/skills/fn-fpk-builder-skill/`
> - Claud: `$HOME/.claude/skills/fn-fpk-builder-skill/`

### 3.6 cmd/main 脚本模板

```bash
#!/bin/sh

install() {
    cd "${TRIM_APPDEST}/app" && docker compose up -d
}

uninstall() {
    cd "${TRIM_APPDEST}/app" && docker compose down -v
}

start() {
    cd "${TRIM_APPDEST}/app" && docker compose start
}

stop() {
    cd "${TRIM_APPDEST}/app" && docker compose stop
}

status() {
    PID_FILE="${TRIM_PKGVAR}/app.pid"
    if [ -f "$PID_FILE" ]; then
        pid=$(cat "$PID_FILE" | tr -d '[:space:]')
        if [ -n "$pid" ] && kill -0 "$pid" 2>/dev/null; then
            exit 0           # 运行中
        fi
        rm -f "$PID_FILE"
    fi
    exit 3                   # 未运行
}

case "$1" in install|uninstall|start|stop|status) "$1" ;; *) exit 1 ;; esac
```

**退出码规则**：`status` → 0=运行中 / 3=未运行；错误写 `$TRIM_TEMP_LOGFILE` + exit 1

### 3.7 回调流程

```
安装：install_init → 系统解压文件 → install_callback（wizard_* 在此可用）
卸载：uninstall_init → 系统删文件(保留var/shares) → uninstall_callback
更新：upgrade_init   → 系统替换文件 → upgrade_callback（TRIM_OLD_APPVER 在此可用）
配置：config_init    → 系统更新环境变量 → config_callback
```

### 3.8 config/privilege 权限声明

```json
{
    "defaults": { "run-as": "package" },
    "username": "myapp",
    "groupname": "myapp"
}
```

`run-as`: `"package"`（推荐安全）或 `"root"`（仅官方合作开发者可发布）

### 3.9 config/resource 资源声明

```json
{
    "data-share": {
        "shares": [
            { "name": "myapp-data", "permission": { "rw": ["myapp"] } }
        ]
    }
}
```

### 3.10 wizard 用户向导

```json
[
    {
        "stepTitle": "基本配置",
        "items": [
            { "type": "text", "field": "port", "label": "端口", "initValue": "32679",
              "rules": [{ "required": true, "message": "必填" }] },
            { "type": "switch", "field": "enable_ssl", "label": "启用SSL", "initValue": "false" },
            { "type": "tips", "helpText": "安装完成后生效" }
        ]
    }
]
```

**类型**：`text` `password` `radio` `checkbox` `select` `switch` `tips`

**验证规则**：`required` / `min` / `max` / `len` / `pattern`

**callback 读取**：`PORT="${wizard_port:-32679}"`

### 3.11 桌面图标 Desktop UI

```
app/ui/config                   # 入口配置文件（JSON）
app/ui/images/icon-64.png       # 64x64
app/ui/images/icon-256.png      # 256x256
```

```json
{
    ".url": {
        "com.example.myapp.main": {
            "title": "我的应用",
            "icon": "images/icon-{0}.png",
            "type": "url", "protocol": "http", "port": "8080", "url": "/", "allUsers": true
        }
    }
}
```

**关键规则**：
- `.url` 下 key **必须以 `appname` 为前缀**
- `{0}` = 尺寸占位符，自动替换 64/256，必须准备两个尺寸文件
- `port` 是**字符串类型**，与 manifest `service_port` 一致
- HTTPS 页面勿配 `protocol: "http"`（协议降级+端口不可达）

### 3.12 错误处理

```bash
echo "错误信息" > "${TRIM_TEMP_LOGFILE}"
exit 1
```

系统以 Dialog 对话框展示给用户。

### 3.13 Bash 脚本黄金 5 条

1. 🔒 **变量加双引号**：`"$TRIM_PKGVAR"`（防路径含空格）
2. 📝 **错误写日志文件**：`"${TRIM_TEMP_LOGFILE}"`（不 echo stdout）
3. 🔢 **明确退出码**：成功 0 / 失败 1 / status 未运行 3
4. 🏗️ **用环境变量构建路径**：`/var/apps/${TRIM_APPNAME}`（不硬编码）
5. ⚠️ **heredoc 注意转义**：用户输入特殊字符可能破坏 JSON

### 3.14 架构选择（Docker vs 原生）

| 应用类型 | manifest `platform` | 说明 |
|---------|---------------------|------|
| **Docker 多架构镜像**（如 `exampleuser/multiarch-app`（示意）） | `"all"` 或 `["x86","arm"]` | 镜像自身支持多架构，fpk 仅编排 |
| **Docker 单架构镜像** | `"x86"` 或 `"arm"` | 镜像仅支持一种架构 |
| **原生 Go/Rust 编译** | 具体架构 `"x86"` 或 `"arm"` | 必须匹配二进制 ELF 架构，不能混用 |
| **纯脚本 / 静态资源** | `"all"` | 无架构依赖 |

> **关键判断**：如果包内有原生二进制（ELF），必须声明具体架构，不能用 `all`。Docker 应用的架构由镜像决定，fpk 只负责编排，可以用 `all` 或列举多架构。

### 3.16 docker-compose.yml 模板

```yaml
version: '3'
services:
  my-app:
    image: username/my-app:latest
    container_name: com.example.myapp
    ports:
      - "${PORT:-32679}:3000"
    volumes:
      - ${TRIM_PKGVAR:-/var/apps/com.example.myapp/data}:/app/data
    environment:
      - PORT=${PORT:-3000}
      - COOKIE_SECURE=false          # HTTP 下必须！
    restart: always
    healthcheck:                     # ✅ fnOS 依赖此判断运行状态
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
```

> **端口说明**：`${PORT:-32679}` 是**宿主机端口**（用户通过此端口访问），`:3000` 是**容器内端口**（应用监听端口）。wizard 让用户配置的是宿主机端口。

