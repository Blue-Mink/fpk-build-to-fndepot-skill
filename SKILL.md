---
name: fpk-build-to-fndepot
description: "从构建 FPK 到发布 FnDepot 开源商店的全流水线：V2 商店协议（fnpack.json/schema_version/releases 多版本结构）、合格源机制、GitHub Release。触发：发布到应用商店、fndepot 发布、fnpack.json、fnOS 应用上架、FPK 商店发布。构建/打包/审计/部署阶段请用 fn-fpk-builder-skill 工具链。"
---

# 从构建 FPK 到发布至 FnDepot 开源商店 — 全链路 Skill

> 整合来源：官方 FPK 开发规范（开源 skill 参考）+ EWEDLCM/FnDepot V2 协议（`fnpack.json` + `schema_version:"2"`，V1 已废弃倒计时） + kci-lnk/fn-fpk-builder-skill 工具链 + 4 个实战应用全过程经验。
> **构建/打包/审计/远程部署的权威工具链与规范在 `fn-fpk-builder` 技能**（其 `scripts/fpk.py`、`references/official-contract.md` 等）；本技能聚焦 **商店发布流水线 + 高频迭代部署**。

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
⑤ 发布 FnDepot（Public 仓库→fnpack.json（V2）→README/图标/预览 URL）
         ↓
⑥ GitHub Release（gh CLI / Python API）
         ↓
⑦ CI 自动化（可选）
```

---


## 按需阅读（路由）

| 场景 | 读哪里 |
|------|--------|
| V2 协议字段（schema_version/source_info/apps/releases/packages）/ 固定分类 / i18n / 拆分模式 | [references/fndepot-protocol.md](references/fndepot-protocol.md) |
| 源地址与索引文件（fnpack.json）/ 发布流程 / V2 检查清单 / V1→V2 迁移 / 合格源机制 | [references/fndepot-protocol.md](references/fndepot-protocol.md#十二发布流程) |
| manifest 字段 / 环境变量 / cmd 脚本 / wizard / 桌面 UI 细节 | [references/fpk-build-spec.md](references/fpk-build-spec.md) |
| 构建→安装测试→Release 逐步操作 | [references/workflow.md](references/workflow.md) |
| 出错排查（构建/安装/部署/运行时/商店五阶段表） | [references/troubleshooting.md](references/troubleshooting.md) |
| Go 原生服务（前端嵌入）构建链 | 下方「3.17」节 + 姊妹技能 fn-fpk-builder-skill 的 `references/go-native-service-flow.md` |
| 装机/换版标准四步（stop→uninstall→install→start）、装后五维验证、错误码表 | 姊妹技能 fn-fpk-builder-skill 的 `references/install-runbook.md`（可执行版 `assets/install-runbook/reinstall.sh`） |
| 三类应用（Docker Web / Native 端口 / 虚拟机承载 / Go 原生服务）完整交付案例 | 姊妹技能 fn-fpk-builder-skill 的 `references/case-studies.md` |
| 入口图标设计（~52px 可读、槽位与墨量） | 姊妹技能 fn-fpk-builder-skill 的 `references/entry-icon-design.md` |
| 已装应用换版（热替换/就地升级/卸载重装） | 下方「4.9」节 |

## 核心坑点速览

### 3.15 ⚠️ 实战五大坑点

| # | 坑点 | 现象 | 解决方案 |
|---|------|------|----------|
| **1** | wizard 配置不生效 | 安装时端口/参数被忽略 | `install-fpk` 必须传 `-e env_file` |
| **2** | checksum 算法错误 | fnpack 构建失败 | 用 `fnpack crc32 app.tgz`（CRC32，非 MD5/SHA） |
| **3** | Docker volume 变量不存在 | 容器启动 volume 挂载失败 | 用 `$TRIM_PKGVAR`，勿用 `$TRIM_DATA_SHARE_PATHS` |
| **4** | HTTP Cookie 被丢弃 | 登录后跳回登录页 | 设置 `COOKIE_SECURE=false` |
| **5** | 容器重建密码丢失 | 重装后数据库重置 | description 提示看日志；或预置 `ADMIN_EMAIL`/`ADMIN_PASSWORD` |


---

### 3.17 Go 原生服务应用构建链（build.sh，高频迭代版）

适用：Go 后端 + SPA 前端（Vite/Next 构建产物嵌入二进制 `go:embed` 或包内 `web/` 目录）的常驻服务 FPK。此类应用发版频率高，构建必须流水线化。**版本号单一事实源 = manifest**，build.sh/ldflags/产物名全部从它读取。

```bash
#!/bin/bash
set -e
export COPYFILE_DISABLE=1
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
BUILD_DIR="$SCRIPT_DIR/build"
FNOS_DIR="$SCRIPT_DIR/fnos"
FNPACK="$BUILD_DIR/fnpack"

# fnpack：官方地址下载，失败用已验证本地副本兜底（版本+哈希须一致）
[ -x "$FNPACK" ] || { curl -fsSL "<官方 fnpack 地址>" -o "$FNPACK" \
  || cp <已验证的本地 fnpack 副本> "$FNPACK"; chmod +x "$FNPACK"; }

VERSION=$(awk -F'= *' '/^version[ \t]*=/ {gsub(/ /,"",$2); print $2; exit}' "$FNOS_DIR/manifest")
[ -n "$VERSION" ] || { echo "manifest 缺 version" >&2; exit 1; }

# 前端（outDir 与包内 web/ 对齐；只有 Go 改动时可整段跳过走快速路径）
cd "$SCRIPT_DIR/frontend" && npm ci --silent && npm run build && cd -

# Go 交叉编译（静态链接 + 版本注入 + 去符号）
export PATH=<Go 工具链持久路径>/bin:$PATH
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -ldflags "-s -w -X main.Version=$VERSION" \
  -o "$BUILD_DIR/bin/app-server" ./cmd/server

# staging 平铺：二进制 + web/（app.tgz 在内容目录内部打包，无外层前缀！）
STAGE="$BUILD_DIR/stage-app"; rm -rf "$STAGE"; mkdir -p "$STAGE"
cp "$BUILD_DIR/bin/app-server" "$STAGE/"
cp -r "$SCRIPT_DIR/web/dist" "$STAGE/web"
( cd "$STAGE" && tar -czf "$BUILD_DIR/app.tgz" . )

# 按 manifest checksum 打包 + 产物命名 + sha256
# ...（fpk.py build 或 fnpack pack，见 4.3/4.4）
cp <final.fpk> "$SCRIPT_DIR/app_${VERSION}_x86.fpk"
sha256sum "$SCRIPT_DIR/app_${VERSION}_x86.fpk"
```

硬约束与坑（实测沉淀，详见姊妹技能 fn-fpk-builder 的 `references/go-native-service-flow.md`）：

| # | 约束/坑 | 说明 |
|---|---------|------|
| 1 | `app.tgz` 无外层前缀 | 成员必须 `./app-server`、`./web/…`（在内容目录内 `tar -czf`）；带 `app/` 前缀 → 安装后文件落错位置（典型错误码 10266 / 入口 404） |
| 2 | go:embed 前端 | 部署后改磁盘 `web/` 对已嵌入内容**无效**；前端有 diff 必须整包重建，只改 Go 可跳过 `npm ci` |
| 3 | x86 only 约定 | manifest `platform=x86`、只挂 x86 资产、fnpack.json 应用级 `platform:["x86"]`（V2）；顺手补打未验证 arm 包是事故源 |
| 4 | /tmp 是 tmpfs 会被回收 | Go 工具链/fnpack/构建产物/部署暂存放持久卷；构建-部署合成单条原子命令 |
| 5 | fnpack 下载间歇 404 | 保留「已验证本地副本」兜底分支，不静默降级到未知版本 |
| 6 | scp 静默失败 | 传输后两端 sha256 比对 |
---

### 4.9 高频迭代部署（开发机上的 fnOS 设备）

同一应用名内高频迭代时，三种部署路径语义完全不同，**选错会丢数据或假成功**：

| 路径 | 适用 | 数据 | 平台 DB |
|------|------|------|---------|
| **① 二进制热替换** | 开发迭代（分钟级） | `@appdata` 不动 | 不更新（版本/端口/入口不变时可用） |
| **② daemon RPC 就地升级** | 正式 FPK 换版 | **保** | 更新 |
| **③ 完整卸载重装** | 改注册信息（端口/入口/manifest 字段） | **可能清空，先备份** | 更新 |

**① 热替换标准链**：

```text
1. ss -tlnp 按应用端口找真实 pid → SIGTERM → 等进程真正退出
   （pid 文件与平台 status 都不可信：旧进程 PPID=1 无监管可能还活着）
2. 换二进制（运行中二进制不能 cp 覆盖：text file busy → 先停进程再 copy
   或 mv 原子替换）+ 同步 web/ 目录
3. 用应用自己的 cmd/main 带全套 TRIM_* 环境拉起
   （TRIM_APPNAME / TRIM_APPDEST / TRIM_PKGVAR / TRIM_SERVICE_PORT）
4. 验证：/api/version（或等价端点）返回目标版本 + 核心 API 冒烟
```

**② daemon RPC 就地升级标准链**（官方 Web UI「升级」走的同一条路）：

```text
1. 升级前记数据指纹：config 文件 sha256 + 数据目录项数
2. FPK 暂存到持久目录 → 升级前检查 canUpgrade
   （同版本升级被拒：目标版本=已装版本时先 bump）
3. 向 /var/run/com.trim.app.center.sock 提交 /rpc/v1/update/task
4. daemon 执行 停应用→换包→自动恢复
5. 验证：DB 登记版本 + 数据指纹一致 + 进程 RUNNING + 运行进程 /api/version
```

**绝对红线与坑**：

- **`install-fpk` 对已安装应用是 no-op**：rc=0 但二进制没换、版本不升、DB 不更新——「覆盖安装报成功」是假成功，换版永远走 停→换→拉起 或 daemon RPC。
- **CLI「升级」= 卸载+重装，毁 `@appdata`**——正式升级禁用 CLI。
- **daemon 换包后不一定重启进程**：DB 已新版、进程仍旧版（磁盘二进制已换、进程未加载）→ 精确 pid 直杀 + 平台 start 拉起。
- 手动 kill 后平台**不 respawn**；残留进程占端口 → 平台 start 报 `APP_AUTO_START_FAILED_PORT_USAGE`（用户看到「点启动没反应」）——杀进程后先确认端口释放。
- **版本判定只信运行进程**：`curl /api/version` 回显 = 构建注入值。平台面板版本显示可滞后于实际二进制（已知现象）。
- `/rpc/v1` 是非契约接口，fnOS 大版本可能变，用前探测。

**发版约定（GitHub 托管）**：

- 轻量 tag `vX.Y.Z` 触发 CI 自动建 Release 挂裸二进制；FPK 手动补挂。
- FPK 资产上传必须走 release 的 `upload_url`（指向 uploads.github.com）——旧 `api.github.com/.../assets` POST 路径已迁移恒 404；资产名带 `?name=app_X.Y.Z_x86.fpk`。
- 新资产 ~60s CDN 传播延迟，期间 404 属正常；持续 404 才查仓库可见性/token 权限。
- **完整性核对免下载**：GitHub 侧 asset digest 与本地 sha256 比对一致即通过。
- 交付附 sha256 尾行；changelog 紧凑格式（单行加粗亮点+一句说明）。
- 推送必须等用户明确指令；多版本积压时一次指令下推齐，tag 对齐各版本 FPK 实际构建时的代码树。
---

## 命令速查

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

## 📜 引用与致谢

- **官方 FPK 开发规范** — 开源 skill 参考（仓库名已脱敏）
- **EWEDLCM/FnDepot** — [GitHub](https://github.com/EWEDLCM/FnDepot)，协议 **V2**（`fnpack.json` + `schema_version:"2"`）
- **kci-lnk/fn-fpk-builder-skill** — [GitHub](https://github.com/kci-lnk/fn-fpk-builder-skill)，MIT
- 实战案例：4 个实战应用的打包发布全过程（已入库某社区 FnDepot 商店，仓库名已脱敏）
- 官方文档：[fnOS Developer Center](https://dev.fnnas.net/)

---

## 发布卫生（脱敏红线）

Skill 文档与仓库要公开，**发布前必须过一遍**：

- 设备地址一律写 `<NAS_IP>`，应用名/端口/版本写 `<app>` / `<port>`；不要出现真实内网地址或公网地址（示例用 `localhost`）。
- 不出现账号口令、token、私钥块、SSH 目标、私有工作区路径。
- **不出现未发布构件的 sha256**（未公开构建的产物哈希同样属敏感信息）；引用工具/官方文档的固定哈希可以保留。
- 样例应用按类别描述（"一个路由器镜像 FPK"），不点真实应用名。
- 本技能自带 `tests/test_docs.py` 做机器校验：`python3 -m unittest discover -s tests`（覆盖路由、锚点、设备信息/凭据/哈希泄漏）。
