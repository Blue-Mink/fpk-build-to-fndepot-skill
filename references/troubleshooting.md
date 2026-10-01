# 故障排查（Troubleshooting）

> 按阶段索引：构建 / 安装 / 部署换版 / 运行时 / FnDepot 商店。

## 五、故障排查（Troubleshooting）

> 用户遇到问题时的系统化排查指南

### 5.1 构建阶段

| 错误 | 原因 | 解决 |
|------|------|------|
| `fnpack pack` 报错 | manifest 中 checksum/tar_size 未更新 | 重新 `fnpack crc32 app.tgz` + `stat -c%s app.tgz`，更新 manifest |
| `no such file or directory` | `ui/` 放在了根目录而非 `app/ui/` | 把 `ui/` 移到 `app/ui/` 下 |
| `manifest: key=value format` | manifest 中等号前后有空格 | 去掉等号前后空格 |
| `platform` 被忽略 | 使用了已废弃的 `arch` 字段 | 改为 `platform=x86` |
| 构建中断「工具链消失」 | fnOS `/tmp` 是 tmpfs 且被持续回收（工具链/暂存放 /tmp） | Go 工具链、fnpack、构建产物、部署暂存一律放持久卷路径 |
| fnpack 下载 404 | 官方 URL 间歇不可达（非 x86 主机尤甚） | 流水线保留「已验证本地副本」兜底（版本+哈希须与登记一致） |

### 5.2 安装阶段

| 错误 | 原因 | 解决 |
|------|------|------|
| `install-fpk` 后应用不运行 | cmd/main 脚本缺执行权限 | `chmod +x cmd/*` 后重新打包 |
| wizard 配置不生效 | `install-fpk` 没传 `-e` | `install-fpk app.fpk -e /var/apps/.../wizard.env` |
| `appcenter-cli status` 返回 3 | 容器/进程未启动 | 检查 `docker ps -a` 或手动执行启动命令 |
| Docker 容器启动后立即退出 | 端口冲突或 volume 路径错误 | 检查端口占用 + 确认 `$TRIM_PKGVAR` 存在 |
| 安装时提示端口被占用 | 其他应用已占用该端口 | 修改 manifest `service_port` + wizard 默认值后重打包 |

### 5.3 部署/换版阶段（已装应用）

| 错误 | 原因 | 解决 |
|------|------|------|
| `install-fpk` 成功但版本没变 | 对已装应用是 no-op（走「已安装」跳过分支） | 换版必须 停→换→拉起（热替换）或 daemon RPC 就地升级 |
| `text file busy`（cp 二进制失败） | 目标二进制正被运行中进程占用 | 先停进程再 copy，或 `mv` 原子替换 |
| `APP_AUTO_START_FAILED_PORT_USAGE`（点启动没反应） | 手动 kill 后残留进程占端口，平台 start 被拒 | `ss -tlnp` 找端口持有者精确清理，确认端口释放后再 start |
| 平台 status 显示 stopped 但进程还在（或反之） | daemon 状态失准（旧进程 PPID=1 无监管） | 判活看真实进程与端口持有者，不信 status |
| 升级「完成」但 `/api/version` 仍旧版 | daemon 换包后未重启进程（磁盘新版、进程旧版） | 精确 pid 直杀旧进程 + 平台 start 拉起 |
| 就地升级被拒（同版本） | 目标版本 = 已装版本 | 先 bump manifest 版本再走升级链 |
| 面板版本显示与实跑不一致 | 面板元数据滞后（已知现象） | 版本判定只信运行进程 `/api/version` 回显 |
| 升级后 `@appdata` 数据没了 | 走了 CLI「升级」（=卸载+重装） | 正式升级只用 daemon RPC；误删时只能从备份恢复 |

### 5.4 运行时

| 错误 | 原因 | 解决 |
|------|------|------|
| HTTP 登录后跳回登录页 | Cookie Secure 标志 | 设 `COOKIE_SECURE=false` 环境变量后重建容器 |
| 页面打开空白/只显示验证码 | ESA WAF 拦截 | 调低 WAF 安全等级，或内网 HTTP 直连 |
| 桌面图标点了不弹出页面 | `app/ui/config` 配置错 | 检查 key 是否以 appname 为前缀 |
| 重装后密码不对 | 容器重建数据库重置 | `docker logs appname \| grep -iE "email\|password\|admin"` |

### 5.5 FnDepot 商店（V2 为主，V1 旧规则标出）

| 错误 | 原因 | 解决 |
|------|------|------|
| 整源同步失败（所有应用消失/不变） | 源级错误：JSON 语法错、`schema_version` 非字符串 `"2"`、`source_info` 缺失 | `jq empty` 修语法；`schema_version` 必须是字符串 `"2"`（`2`/`"2.0"` 都不行） |
| 单个应用不显示 | 应用级错误：键名 ≠ FPK `appname`、`categories` 用了自创分类、缺必填字段、详情 `app_name` 不匹配 | 键名与 manifest 完全一致（大小写）；categories 只用 9 个固定值（≤2）；补必填字段 |
| 版本/包被跳过 | 版本级：`packages` 架构键不在 all/x86/arm、`sha256` 格式错、`size` 是文本、URL 不可达 | 架构键白名单；sha256 64 位 hex；size 整数字节；curl 验证 download_url |
| 下载后校验失败 | JSON `sha256` 与实际 FPK 不一致（静默替换过同版本文件） | 重新 `sha256sum` 同步 JSON；已发布版本+架构文件视为不可变，改内容发新版本 |
| 商店显示"不适用于此设备" | `platform` 不含当前架构，或 `packages` 无对应架构且无 `all` 回退 | 检查 platform 声明 + packages 分支（x86 设备：`packages.x86` → `packages.all`） |
| 应用不更新版本 | `releases` 下没加新版本键（V2） | 新增可比较版本键 + 更新 updated_at/地址/size/sha256 后 git push |
| 移除应用不生效 | 用空文件/格式错误/临时故障移除 → 触发缓存保留 | 从 `apps` 删键（应用）或 `releases` 删键（版本）后 git push |
| GitHub 仓库添加源失败 | 根目录没有名称完全匹配的 `fnpack.json`（大小写严格） | 放根目录 `fnpack.json`；其他文件名请给 JSON 直链 |
| 旧规则（仅 V1 仓库适用） | — | V1 时代的「目录名=键名」「`ICON.PNG` 全大写」「`fndepot.json` version 字段」在 V2 已失效（客户端不扫描目录、图标走 `icon_url`） |

---

