# FnDepot 开源商店协议（V2）与发布

> 来源：EWEDLCM/FnDepot 官方规范（V2，2026-09-30 核实）+ 实战沉淀。
> **客户端要求 >v0.0.7**（飞牛已 httpOnly，旧客户端失效）。V1 暂兼容但官方公告后续废弃，新源直接写 V2。
> 外部源定位：由用户自行添加，仅本地生效；FnDepot 官方不对外部源的应用安全性/稳定性背书。

## 一、V2 顶层结构

```json
{
  "schema_version": "2",
  "source_info": { "name": "源显示名", "author": "源维护者" },
  "apps": { "<appname>": { "…应用字段": "…" } }
}
```

| 字段 | 要求 |
|------|------|
| `schema_version` | 必须**字符串** `"2"`；`2`、`"2.0"`、`"2.1"` 均不合法 |
| `source_info.name` / `author` | 必填（源显示名 / 源维护者） |
| `source_info.homepage` / `description` / `i18n` | 可选 |
| `apps` | 对象，键名 = FPK manifest `appname`（大小写敏感，字母/数字开头，只允许字母数字点下划线短横线） |

必须严格 JSON：UTF-8、无注释、无尾逗号。

## 二、源地址与索引文件

- **JSON 直链**：任意文件名的 HTTP(S) 直链均可，内容符合 V1/V2 即识别。
- **GitHub 仓库地址**：根目录**必须存在**名称完全匹配的 `fnpack.json`（大小写严格）；客户端不探测 `fndepot.json` 或其他文件名。其他布局请给 JSON 直链而非仓库地址。
- 客户端按 JSON **内容**识别 V1/V2，不按文件名。
- **目录结构无强制要求**：客户端只按 JSON 内 URL 解析，不扫描目录。FPK/图标/预览图可分目录、可集中存放，只要 JSON 相对 URL 与仓库实际布局一致。

两种常见布局（示例，任选或自定）：

```text
# 按应用分目录
FnDepot/
├── fnpack.json
└── app1/{fpk/,icons/,previews/}

# 集中式（便于批量校验脚本）
FnDepot/
├── fnpack.json
├── apps/sample.app.json        # 拆分详情
├── assets/{icons/,previews/}
└── packages/sample.app-1.0.0-x86.fpk
```

## 三、应用字段

### 3.1 常用字段

| 字段 | 类型 | 要求 |
|------|------|------|
| `display_name` | 字符串 | 合并后必填 |
| `desc` | 字符串 | 合并后必填（支持 HTML，客户端清理危险内容） |
| `platform` | 字符串/数组 | 合并后必填，只允许 `all`/`x86`/`arm` |
| `categories` | 字符串数组 | 合并后必填，**固定分类**（见 3.2），≤2 个，第一项为主分类 |
| `icon_url` | URL | 合并后必填（建议 PNG/WebP <500KB） |
| `preview_urls` | URL 数组 | 可选，最多读前 8 张（单张尽量 <2MB） |
| `readme_url` | URL | 可选（README <1MB） |
| `bug_report_url` | URL | 可选 |
| `details_url` | URL | 可选，拆分模式详情 JSON（见第六节） |
| `details_updated_at` | 字符串 | 可选，**只能放应用节点** |
| `maintainer` / `maintainer_url` | 字符串 | 可选，开发者 |
| `distributor` / `distributor_url` | 字符串 | 可选，发布者（与开发者相同则省略） |
| `run_as` | 字符串 | 合并后必填，`package` 或 `root` |
| `install_type` | 字符串 | 合并后必填，`""`=存储空间 / `root`=系统空间 |
| `is_docker` | **布尔** | 合并后必填 |
| `service_port` | 字符串 | 可选，无端口填空串 |
| `releases` | 对象 | 单文件模式必填（版本集合，见第四节） |
| `i18n` | 对象 | 可选（见第五节） |

### 3.2 固定分类（9 个，不可自创）

`影音娱乐、系统工具、编程开发、AI赋能、生活服务、智能智控、教育学习、游戏地带、硬件驱动`

「全部」是界面筛选项，不要写入 `categories`。

### 3.3 开发者 / 发布者 / 源作者（三角色勿混）

- `source_info.author` = 维护这个**源**的人（≠ 应用开发者，除非确是同一主体）
- `maintainer` = 开发**应用**的人
- `distributor` = 发布/打包/分发的人

## 四、版本与安装包（V2 核心：多版本共存）

```text
releases.<version>.packages.<arch>
```

### 4.1 版本节点 `releases.<version>`

键 = 可比较版本号（推荐 SemVer；禁用 `latest`/日期文字）。字段：

| 字段 | 说明 |
|------|------|
| `changelog` | 更新说明 |
| `updated_at` | 建议 ISO 8601 带时区（`2026-09-08T00:00:00+08:00`） |
| `os_min_version` / `os_max_version` | 系统版本范围（空/缺失=不限） |
| `run_as` / `install_type` / `is_docker` / `service_port` | 覆盖应用级 |
| `packages` | **必填**，按架构提供安装包 |

### 4.2 安装包分支 `packages.<arch>`

架构键只允许 `all`/`x86`/`arm`（其他键使版本校验失败）：

| 字段 | 要求 |
|------|------|
| `download_url` | **必填**（必须存在于最终选中分支，不可跨分支继承） |
| `sha256` | 强烈建议（64 位 hex；**存在即客户端强制校验**） |
| `size` | 建议，**整数字节**（禁止 `"20 MB"` 文本） |
| `changelog` / `i18n` / `updated_at` / `os_*` / `run_as` 等 | 覆盖上层 |

**最稳妥：每个含 `download_url` 的分支都自带 `sha256` + `size`。**

### 4.3 选择与合并顺序

- 包选择：`packages.<当前架构>` → 缺失/无效回退 `packages.all` → 都没有则该版本不可安装。
- 字段合并：`应用级 → 版本级 → packages.all → packages.<当前架构>`（all 是通用默认层，架构分支是最后覆盖层）。
- `platform: ["all"]` 的多架构合一包直接命名 `packages.all`；不要自创 `universal` 等键。

## 五、国际化 i18n

可出现层级：`source_info`（name/description）、应用节点（display_name/desc）、版本节点（changelog）、安装包分支（changelog）。

回退顺序（每字段独立）：精确语言（zh-TW）→ 同语系（zh-CN）→ en-US → zh-CN → 基础字段。**基础字段必须填写**，不能只填 i18n。

运行时语言以 FnOS 环境变量 `TRIM_SYS_LANGUAGE` 为准（不是浏览器 Cookie/Accept-Language）。

## 六、details_url 拆分模式

源应用多时，主 `fnpack.json` 只存索引：

```json
"apps": { "sample.app": { "details_url": "apps/sample.app.json", "details_updated_at": "2026-08-05T12:00:00+08:00" } }
```

详情文件：首字段 `app_name` 必须与主索引键名**完全一致**；详情字段覆盖主索引同名字段后合并校验；相对 URL 相对**详情文件所在目录**解析；失败/超时/校验错 → 跳过该应用并保留本地有效缓存。

## 七、URL、缓存与资源限制

- URL 只允许 http/https；主源资源相对主 JSON 地址解析；禁止 file/ftp/data、URL 内嵌凭据/长期令牌。
- 大小上限：主源 ~2MB；单详情 ~512KB；单源 ≤5000 应用；单应用 ≤100 版本；预览图 ≤8 张。
- 客户端用短时缓存 + ETag/Last-Modified 条件请求 → 源服务器建议稳定 URL、`Content-Type: application/json; charset=utf-8`、提供 ETag/Last-Modified。
- FPK 建议：准确 `Content-Length`、`size`、`sha256`，服务支持 `Accept-Ranges: bytes`（断点续传）。
- 外部源应用**不统计/上报**下载量、评分、评论。

## 八、校验失败与缓存行为（四级粒度）

| 粒度 | 触发 | 行为 |
|------|------|------|
| 源级 | JSON 语法错、`schema_version` 非法、`source_info` 缺失、`apps` 类型错 | 整源同步失败，**不清空已有缓存** |
| 应用级 | 字段类型错、分类非法、缺必填、`app_name` 不匹配、无可用版本 | 只跳过该应用 |
| 版本/包级 | 无有效 packages、URL 非法、哈希格式错、架构不可用 | 只跳过该版本/分支；架构分支失败有合法 all 则回退 |
| 缓存保留 | 单应用同步失败 | 保留上次有效缓存（防网络抖动致应用消失）；只有源同步成功且应用键被明确移除才清理 |

**不要用空文件/格式错误/临时故障来移除应用**（会触发缓存保留，移除不生效）。

## 九、GitHub 来源、Fork 与重复源

- `source_info.name/author/homepage` 不是唯一身份；同名/同 CDN 不构成同源证明。
- 客户端用 GitHub API 真实血缘（`full_name`/`fork`/`parent`/`source`）+ `appname|version` 重叠度识别高风险重复源，**可能停用后添加的重复 Fork**。
- 不要加 `source_info.id` 试图建立/规避身份关系（该字段无身份语义）。

## 十、合格源机制（2026-09 新增）

- EWEDLCM/FnDepot 仓库根目录 `valid_sources.txt` = **合格外部源名单**（每日聚合）。
- GitHub Action 每天**北京时间 00:00** 运行 `scripts/generate_sources.py`：扫描候选 FnDepot 仓库 → 校验协议符合性 → 重新生成名单并自动提交（commit 前缀「🤖 自动更新合格源列表」）。
- 含义：源不符合 V1/V2 规范会被**移出合格源名单**（可见性/信任度下降）。发布前用官方检查清单自测（下节）。

## 十一、V1 兼容与迁移（存量源参考）

新版客户端同时支持 V1/V2（按内容识别），**V1 后续取消兼容**。

V1 识别特征：无 `schema_version`，根节点直接是 `appname → 应用信息` 扁平映射（`version`/`download_url`/`arch_diff` 等）。

V1 兼容规则保留：无 `platform` 默认 x86；架构包优先匹配；未标架构包视为全架构（不强制 `_all` 文件名）；回退顺序 `显式 download_url → 架构命名包 → 通用命名包`（`{appname}.fpk` 可作通用包）。

**V1→V2 迁移七步**：

1. 仓库文件名保持 `fnpack.json`（GitHub 模式强制；原 `fndepot.json` 不再被探测）。
2. 加 `schema_version: "2"` + `source_info`（name/author）。
3. `labels` 自由文本 → 整理为 9 个固定 `categories`（≤2 个）。
4. 版本信息 → `releases.<version>.packages.<arch>` 三层结构。
5. 每个 FPK 填真实 `download_url` + `size`（整数）+ `sha256`；`ICON.PNG`/`Preview/` 目录习惯 → `icon_url`/`preview_urls` URL。
6. 用 FPK manifest 的 `appname`/`version`/`platform` 逐项核对 JSON。
7. x86 与 arm 设备分别实测包选择和安装；确认用户客户端 >v0.0.7。

V1 时代的目录约定（`{app}/ICON.PNG` 全大写 + `Preview/` ≥2 张 + 目录名=键名）**不再是 V2 强制要求**——客户端不扫描目录，只认 JSON 内 URL。

## 十二、发布流程

```text
1. 准备 fnpack.json（V2 结构，严格 JSON）
2. FPK/图标/预览图上传仓库（布局自定，URL 与 JSON 一致）
3. jq 校验 + sha256/size 核对（下节命令）
4. git push 到 main
5. 用户侧添加源（直链或仓库地址）→ 检查同步状态与被跳过应用的错误详情
6. 双架构 + 双语言实测展示与安装
```

## 十三、发布前检查清单（V2）

**JSON**：
- [ ] UTF-8 严格 JSON（无注释/尾逗号）；`jq empty fnpack.json` 通过
- [ ] `schema_version` 是字符串 `"2"`；`source_info.name/author` 非空
- [ ] 每个应用键名与 FPK `appname` 完全一致（保留大小写）
- [ ] `categories` 只用 9 个固定分类（≤2）；`platform` 只用 all/x86/arm
- [ ] `run_as`/`install_type`/`is_docker` 类型与值正确（is_docker 是布尔）
- [ ] `details_updated_at` 只在应用节点；i18n 字段名与层级匹配且基础字段仍在

**安装包**：
- [ ] 每个目标架构有 `packages.<arch>` 或 `packages.all`
- [ ] `download_url` 真实可访问，相对 URL 基准目录正确
- [ ] `size` = 实际字节数（`stat -c '%s'`）
- [ ] `sha256` 与 `sha256sum` 一致
- [ ] FPK 内 manifest 的 appname/version/platform 与 JSON 一致

**发布后**：
- [ ] 源同步状态正常，无意外跳过的应用
- [ ] x86/arm 分别验证展示与包选择
- [ ] 简介/图标/预览/README/开发者/发布者显示正确
- [ ] `TRIM_SYS_LANGUAGE=zh-CN` 与 `en-US` 下 i18n 回退正确

## 十四、发布新版本与移除应用

- 新版本 = `releases` 下**新增**可比较版本键 + 更新 `updated_at`/地址/size/sha256。
- **已发布的「版本号+架构」文件视为不可变**：内容变更必须发新版本或新地址，不要静默替换同版本 FPK。
- 移除版本 = 从 `releases` 删键；移除应用 = 从 `apps` 删键；下次成功同步后客户端清理未安装应用的旧缓存。
