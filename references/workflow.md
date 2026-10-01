# 操作流程（构建 → 安装测试 → Release）

> 高频迭代部署（热替换/就地升级）见 SKILL.md「4.9」节。

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

**FnDepot 客户端**：自动检测源中版本变化即提示更新（V2：`releases` 下出现新的可比较版本键；V1：`version` 字段变化）。

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

### 4.6 发布到 FnDepot（V2）

协议细节见 [fndepot-protocol.md](fndepot-protocol.md)。要点：**V2 客户端不扫描目录，只认 `fnpack.json` 内的 URL**——目录布局自定，文件与 JSON 相对 URL 一致即可。

```bash
# 1. 进入商店仓库（GitHub 仓库模式：根目录必须有 fnpack.json；
#    社区惯例仓库名用 FnDepot，但协议不再强制仓库名）
cd FnDepot

# 2. 上传 FPK 与资源（布局自定，示例为集中式）
mkdir -p packages assets/icons assets/previews
cp /path/to/my-app-1.0.0-x86.fpk packages/
cp /path/to/icon.png    assets/icons/my-app.png
cp /path/to/preview-*.png assets/previews/

# 3. 编辑根目录 fnpack.json（V2 结构）：
#    apps.my-app 增加 display_name/desc/platform/categories(9 固定值≤2)/
#    icon_url/run_as/install_type/is_docker + releases."1.0.0".packages.x86
#    （download_url + size 整数 + sha256）

# 4. 校验 + 提交推送
jq empty fnpack.json
sha256sum packages/my-app-1.0.0-x86.fpk    # 与 JSON sha256 一致
stat -c '%s' packages/my-app-1.0.0-x86.fpk # 与 JSON size 一致
git add . && git commit -m "add my-app v1.0.0" && git push origin main
```

存量 V1 仓库（`fndepot.json` + `ICON.PNG` 目录习惯）仍可工作（客户端暂兼容），但官方公告 V1 后续废弃——迁移七步见 [fndepot-protocol.md 第十一节](fndepot-protocol.md#十一v1-兼容与迁移存量源参考)。

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
2. 更新 `fnpack.json`（V2）：`releases` 新增版本键 + `changelog` / `size` / `sha256`（已发布版本+架构文件不可变，勿静默替换）
3. `git push` 到 `main` → 客户端自动检测更新

