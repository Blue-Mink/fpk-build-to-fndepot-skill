---
name: fpk-build-to-fndepot
description: Full-pipeline skill for fnOS FPK development → FnDepot publishing. Merges jianyun8023's official FPK development spec (manifest/lifecycle/privilege/resource/wizard/desktop-UI/environment-variables/error-handling) with FnDepot v1.1.1 store protocol (fndepot.json/multi-arch/ICON/Preview/Release) and real-world battle-tested pitfalls from TREK/global-radio/fn-knock deployments. Covers: FPK project scaffolding → packaging (fnpack/fpk.py) → local install verification → FnDepot repo publishing → GitHub Release → CI automation + troubleshooting. Trigger when users say: build fpk, publish to FnDepot, 打包 fpk, 发布到应用商店, fndepot 发布, fnOS 应用打包, fpk 构建发布, fpk 开发, 飞牛应用开发.
---

# 从构建 FPK 到发布至 FnDepot 开源商店 — 全链路 Skill

> 整合四大来源：
> 1. **jianyun8023/fnos-fpk-dev** — 官方 FPK 开发规范
> 2. **EWEDLCM/FnDepot v1.1.1** — 开源商店协议
> 3. **kci-lnk/fn-fpk-builder-skill** — 自动化构建/审计/部署工具链
> 4. **实战踩坑** — TREK/global-radio/fn-knock/kms-activator 全过程经验

---

## 一、全链路总览

```
① 分析项目（类型/架构/端口/权限/桌面UI）
         ↓
② 创建 FPK 骨架（manifest + cmd/ + config/ + wizard/ + app/ + ui/ + 图标）
         ↓
③ 构建包（准备产物 → app.tgz → CRC32 → fnpack pack / fpk.py build）
         ↓
④ 本地安装测试（install-fpk -e → status → HTTP → 日志）
         ↓
⑤ 发布 FnDepot（Public 仓库→目录→fndepot.json→README→Preview）
         ↓
⑥ GitHub Release（gh CLI / Python API）
         ↓
⑦ CI 自动化（可选）
```

---

## 二、FnDepot 开源商店协议（v1.1.1）

> 来源：EWEDLCM/FnDepot

### 2.1 仓库规范

| 项目 | 要求 |
|------|------|
| 托管平台 | **GitHub** |
| 可见性 | **Public**（私有无法索引） |
| 仓库名 | **`FnDepot`**（[强制] 必须） |
| 分支 | **main** |

### 2.2 目录结构

```
FnDepot/
├── fndepot.json                  # 全局元数据索引（核心）
├── fnpack.json                   # [可选] 兼容旧版索引（部分仓库同时保留）
├── README.md                     # 仓库说明（列所有应用）
│
├── {app_name}/
│   ├── ICON.PNG                  # [必填] 全大写！64x64
│   ├── ICON_256.PNG              # 高清图标
│   ├── {app_name}_{arch}.fpk     # 多架构安装包
│   ├── {app_name}.fpk            # 兼容模式回退
│   ├── README.md                 # 应用详情（含首次使用提示）
│   └── Preview/                  # 预览截图≥2张
│       ├── 01.png
│       └── 02.jpg
```

### 2.3 fndepot.json 规范

```json
{
  "my-app": {
    "display_name": "我的应用",
    "platform": ["x86", "arm"],
    "version": "1.0.0",
    "desc": "应用描述\\n首次使用请查看容器日志",
    "labels": "工具,效率",
    "distributor": "发布者",
    "distributor_url": "https://github.com/...",
    "bug_report_url": "https://github.com/.../issues",
    "install_type": "系统空间",
    "size": "15",
    "isdocker": "true",
    "download_url": "",
    "changelog": "更新内容",
    "arch_diff": {
      "arm": {
        "version": "1.0.1",
        "size": "25",
        "desc": "ARM 优化版",
        "download_url": "./my-app/my-app_arm.fpk"
      }
    }
  }
}
```

**必填字段**：`display_name` / `version` / `desc` / `labels` / `distributor` / `distributor_url` / `bug_report_url` / `install_type` / `size` / `changelog`

**架构匹配规则**：
1. 检测客户端架构 → 2. 筛选 platform 匹配的应用（不匹配不显示） → 3. `arch_diff[arch]` > 通用字段 → 4. 下载：`download_url` > 目录拼接 > 无文件不入库

**安装包查找优先级**：`{app_name}_{arch}.fpk` > `{app_name}_all.fpk` > `{app_name}.fpk`

### 2.4 fnpack.json 说明（兼容层）

部分 FnDepot 仓库（如 Blue-Mink/FnDepot）同时保留 `fnpack.json` 作为旧版兼容索引。它的结构与 fndepot.json 类似但字段更少。**新开发建议只维护 `fndepot.json`**，如有旧版客户端则保留 `fnpack.json` 同步更新。

---

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
| **Docker 多架构镜像**（如 `mauriceboe/trek`） | `"all"` 或 `["x86","arm"]` | 镜像自身支持多架构，fpk 仅编排 |
| **Docker 单架构镜像** | `"x86"` 或 `"arm"` | 镜像仅支持一种架构 |
| **原生 Go/Rust 编译** | 具体架构 `"x86"` 或 `"arm"` | 必须匹配二进制 ELF 架构，不能混用 |
| **纯脚本 / 静态资源** | `"all"` | 无架构依赖 |

> **关键判断**：如果包内有原生二进制（ELF），必须声明具体架构，不能用 `all`。Docker 应用的架构由镜像决定，fpk 只负责编排，可以用 `all` 或列举多架构。

### 3.15 ⚠️ 实战五大坑点

| # | 坑点 | 现象 | 解决方案 |
|---|------|------|----------|
| **1** | wizard 配置不生效 | 安装时端口/参数被忽略 | `install-fpk` 必须传 `-e env_file` |
| **2** | checksum 算法错误 | fnpack 构建失败 | 用 `fnpack crc32 app.tgz`（CRC32，非 MD5/SHA） |
| **3** | Docker volume 变量不存在 | 容器启动 volume 挂载失败 | 用 `$TRIM_PKGVAR`，勿用 `$TRIM_DATA_SHARE_PATHS` |
| **4** | HTTP Cookie 被丢弃 | 登录后跳回登录页 | 设置 `COOKIE_SECURE=false` |
| **5** | 容器重建密码丢失 | 重装后数据库重置 | description 提示看日志；或预置 `ADMIN_EMAIL`/`ADMIN_PASSWORD` |

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

---

## 四、操作流程

### 4.1 分析项目 → 确定参数

检查：语言类型 / 是否Docker / 端口 / 持久化 / 构建方式 / 架构（Docker多架构镜像？还是原生ELF？）/ 桌面UI / 向导需求

### 4.2 版本号规范

```
主版本.次版本.修订号[-预发布]
1.0.0          # 初版
1.0.1          # 小修复
1.1.0          # 新功能（向后兼容）
2.0.0          # 不兼容变更
3.4.1-beta     # 预发布版
```

**FnDepot 客户端**：自动检测 `fndepot.json` 中 `version` 字段变化，版本号变更即提示更新。

### 4.3 自动化构建（fn-fpk-builder-skill）

```bash
# SKILL_DIR 常见位置：
#   QwenPaw: skills/fn-fpk-builder/
#   Codex:   $HOME/.codex/skills/fn-fpk-builder-skill/
#   Claud:   $HOME/.claude/skills/fn-fpk-builder-skill/
SKILL_DIR=/path/to/skills/fn-fpk-builder

# 安装 fnpack 工具链
python3 "$SKILL_DIR/scripts/fpk.py" toolchain --install --json

# 诊断项目
python3 "$SKILL_DIR/scripts/fpk.py" doctor --project ./pkg --json

# 构建
python3 "$SKILL_DIR/scripts/fpk.py" build --project ./pkg --out ./dist --arch x86 --json

# 独立审计
python3 "$SKILL_DIR/scripts/fpk.py" inspect ./dist/app.fpk --json
```

### 4.4 手动构建（fnpack）

```bash
# 1. 准备 app 目录
# 2. 打包 app.tgz
cd app && tar -czf ../app.tgz . && cd ..

# 3. CRC32 checksum（不是 MD5 也不是 SHA256！）
CRC32=$(fnpack crc32 app.tgz)
TAR_SIZE=$(stat -c%s app.tgz)
echo "CRC32=$CRC32  TAR_SIZE=$TAR_SIZE"
# → 更新 manifest 中的 checksum 和 tar_size

# 4. 执行构建
fnpack pack .                    # 打包当前目录
# 或 fnpack pack ./my-app-fpk    # 打包指定目录
# 产物：当前目录下的 .fpk 文件
```

### 4.5 本地安装测试

```bash
# 安装（wizard 应用必须加 -e！）
install-fpk ./app.fpk -e /var/apps/com.example.myapp/app/wizard.env

# 验证状态
appcenter-cli status com.example.myapp
# exit 0 = running | exit 3 = not running

# 测试端口
curl -I http://localhost:32679

# 查看日志
docker logs com.example.myapp 2>&1 | grep -iE "error|warn|fail"

# 查看初始密码
docker logs com.example.myapp 2>&1 | grep -iE "email|password|admin"
```

### 4.6 发布到 FnDepot

```bash
# 1. 克隆/进入 FnDepot 仓库（名称必须为 FnDepot）
cd FnDepot

# 2. 创建应用目录
mkdir -p my-app/Preview

# 3. 复制文件
cp /path/to/ICON.PNG my-app/
cp /path/to/ICON_256.PNG my-app/
cp /path/to/README.md my-app/
cp /path/to/my-app.fpk my-app/
cp /path/to/preview/*.png my-app/Preview/

# 4. 编辑 fndepot.json + 根目录 README.md

# 5. 提交推送
git add . && git commit -m "add my-app v1.0.0" && git push origin main
```

### 4.7 GitHub Release

```bash
# 方式一：gh CLI（推荐）
gh release create v1.0.0 \
  --title "my-app v1.0.0" \
  --notes "初始版本" \
  ./my-app.fpk

# 方式二：Python API（网络差/走NAS中转时用）
python3 -c "
import requests
t = 'token YOUR_TOKEN'
h = {'Authorization': t, 'Accept': 'application/vnd.github+json'}
r = requests.post('https://api.github.com/repos/YOURNAME/FnDepot/releases',
    json={'tag_name':'v1.0.0','name':'v1.0.0'}, headers=h)
u = r.json()['upload_url'].split('{')[0]
with open('app.fpk','rb') as f:
    requests.post(u+'?name=app-v1.0.0.fpk',
        headers={'Authorization':t,'Content-Type':'application/octet-stream'}, data=f)
print(r.json()['html_url'])
"
```

### 4.8 版本更新

FnDepot 客户端自动同步，只需：
1. 替换 `my-app/` 下的 `.fpk` 文件
2. 更新 `fndepot.json`：`version` / `changelog` / `size`
3. `git push` 到 `main` → 客户端自动检测更新

---

## 五、故障排查（Troubleshooting）

> 用户遇到问题时的系统化排查指南

### 5.1 构建阶段

| 错误 | 原因 | 解决 |
|------|------|------|
| `fnpack pack` 报错 | manifest 中 checksum/tar_size 未更新 | 重新 `fnpack crc32 app.tgz` + `stat -c%s app.tgz`，更新 manifest |
| `no such file or directory` | `ui/` 放在了根目录而非 `app/ui/` | 把 `ui/` 移到 `app/ui/` 下 |
| `manifest: key=value format` | manifest 中等号前后有空格 | 去掉等号前后空格 |
| `platform` 被忽略 | 使用了已废弃的 `arch` 字段 | 改为 `platform=x86` |

### 5.2 安装阶段

| 错误 | 原因 | 解决 |
|------|------|------|
| `install-fpk` 后应用不运行 | cmd/main 脚本缺执行权限 | `chmod +x cmd/*` 后重新打包 |
| wizard 配置不生效 | `install-fpk` 没传 `-e` | `install-fpk app.fpk -e /var/apps/.../wizard.env` |
| `appcenter-cli status` 返回 3 | 容器/进程未启动 | 检查 `docker ps -a` 或手动执行启动命令 |
| Docker 容器启动后立即退出 | 端口冲突或 volume 路径错误 | 检查端口占用 + 确认 `$TRIM_PKGVAR` 存在 |
| 安装时提示端口被占用 | 其他应用已占用该端口 | 修改 manifest `service_port` + wizard 默认值后重打包 |

### 5.3 运行时

| 错误 | 原因 | 解决 |
|------|------|------|
| HTTP 登录后跳回登录页 | Cookie Secure 标志 | 设 `COOKIE_SECURE=false` 环境变量后重建容器 |
| 页面打开空白/只显示验证码 | ESA WAF 拦截 | 调低 WAF 安全等级，或内网 HTTP 直连 |
| 桌面图标点了不弹出页面 | `app/ui/config` 配置错 | 检查 key 是否以 appname 为前缀 |
| 重装后密码不对 | 容器重建数据库重置 | `docker logs appname \| grep -iE "email\|password\|admin"` |

### 5.4 FnDepot 商店

| 错误 | 原因 | 解决 |
|------|------|------|
| FnDepot 不显示应用 | 目录名 ≠ fndepot.json 键名 | 确保三者一致：目录名=键名=app_name |
| 应用不更新版本 | `fndepot.json` 中 version 没更新 | 更新 version 后 git push |
| ICON 不显示 | `ICON.PNG` 文件名未全大写 | 改为 `ICON.PNG`（全大写） |
| 商店显示"不适用于此设备" | `platform` 不匹配设备架构 | 检查 `platform` 声明是否包含用户架构 |

---

## 六、快捷查询

### 6.1 命令速查

| 操作 | 命令 |
|------|------|
| 安装 fnpack | `python3 fpk.py toolchain --install --json` |
| CRC32 | `fnpack crc32 app.tgz` |
| 打包 app | `tar -czf app.tgz -C app .` |
| 构建 FPK | `fnpack pack .` 或 `fpk.py build --project ...` |
| 审计 FPK | `fpk.py inspect ./app.fpk --json` |
| 安装 | `install-fpk ./app.fpk` |
| 安装(wizard) | `install-fpk ./app.fpk -e /path/to/env` |
| 状态 | `appcenter-cli status appname` |
| 卸载 | `appcenter-cli uninstall appname` |
| 查看日志 | `docker logs appname` |
| 查初始密码 | `docker logs appname \| grep -iE "email\|password\|admin"` |
| Release | `gh release create v1.0.0 --title "v1.0.0" ./app.fpk` |

### 6.2 发布前检查清单

- [ ] FnDepot 仓库 **Public**
- [ ] `fndepot.json` 完整（无缺失必填字段）
- [ ] `ICON.PNG` **全大写**
- [ ] 目录名 = fndepot.json 键名
- [ ] .fpk 安装包就位且架构命名正确
- [ ] README.md 含首次使用提示（查看日志获取密码）
- [ ] Preview/ 截图 ≥2 张
- [ ] 根目录 README 已更新应用列表
- [ ] git push 到 main 成功
- [ ] GitHub Release 已创建 + fpk asset 已上传
- [ ] 🧪 本地安装验证通过（running + HTTP 200 + 日志无报错）
- [ ] manifest 字段完整（含 checksum + tar_size + desktop 配置）
- [ ] `cmd/*` 脚本有执行权限（`chmod +x`）
- [ ] `ui/` 在 `app/ui/` 下（不是根目录）
- [ ] manifest 中 platform 与架构匹配

### 6.3 README 模板（应用目录）

```markdown
# 应用名称

## 简介
应用描述

## 截图
![截图1](Preview/01.png) ![截图2](Preview/02.png)

## 版本信息
- 当前版本：1.0.0 | 最后更新：2026-07-25

## 安装
通过 FnDepot 应用源安装，或下载 fpk 手动：
```bash
install-fpk ./app_name.fpk
```

## 首次使用
首次启动后查看容器日志获取默认管理员邮箱和密码：
```bash
docker logs app_name 2>&1 | grep -iE "email|password|admin"
```

## 反馈
[Issues](https://github.com/YOURNAME/FnDepot/issues)
```

---

## 📜 引用与致谢

- **jianyun8023/fnos-fpk-dev** — [GitHub](https://github.com/jianyun8023/my-skills/tree/main/skills/tooling/fnos-fpk-dev) / [SkillsMP](https://skillsmp.com/creators/jianyun8023/my-skills/skills-tooling-fnos-fpk-dev)
- **EWEDLCM/FnDepot** — [GitHub](https://github.com/EWEDLCM/FnDepot)，协议 v1.1.1
- **kci-lnk/fn-fpk-builder-skill** — [GitHub](https://github.com/kci-lnk/fn-fpk-builder-skill)，MIT
- 实战案例：TREK / global-radio / fn-knock / kms-activator → [Blue-Mink/FnDepot](https://github.com/Blue-Mink/FnDepot)
- 官方文档：[fnOS Developer Center](https://dev.fnnas.net/)