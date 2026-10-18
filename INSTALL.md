# 安装：fpk-build-to-fndepot（跨 Agent）

本技能只依赖 Markdown + 标准工具；发布阶段会用到 GitHub API/Release（网络），
构建/打包/部署依赖姊妹技能 **`fn-fpk-builder`**。

## 一键安装

```bash
S=<本技能目录>/scripts/install.sh
sh $S --list                                  # 查看目标
sh $S --target claude-code --apply            # ~/.claude/skills/
sh $S --target codex --apply                  # ~/.codex/prompts/
sh $S --target cursor --apply                 # .cursor/rules/
sh $S --target agents-md --apply              # AGENTS.md
sh $S --target skills-dir --dest <目录> --apply --with-deps   # 连 fn-fpk-builder 一起装
```

| 参数 | 作用 |
|---|---|
| `--target` | 目标平台（见 `--list`） |
| `--dest` | 自定义落地目录 |
| `--link` | 符号链接（升级本目录即生效） |
| `--with-deps` | 同时复制同级 `fn-fpk-builder`（等价 `--dep fn-fpk-builder`） |
| `--dep <名字,…>` | 指定要一并复制的同级依赖技能（可用于任意技能） |
| `--apply` | 真正写入（默认 dry-run） |

## 支持的目标

- **技能目录**：`qwenpaw` `claude-code` `qwen-code` `agents-skills` `skills-dir`
- **提示词/命令**：`codex` `gemini-cli` `hermes` `generic`
- **项目规则**：`agents-md` `cursor` `windsurf` `cline` `copilot` `kiro`
- **组合**：`all-project`

> 复制时自动排除 `.git/`、`.pushpack*`、`__pycache__/`、`*.pyc`、`node_modules/`，
> 本技能目录自带 git 仓库也能安全分发。

## 依赖与边界

- **`fn-fpk-builder`**：构建/打包/审计/远程部署的权威工具链；未装时用 `--with-deps`，
  或把两个技能目录放在同一 `skills/` 下。
- **网络**：发布阶段访问 `api.github.com`（Release 上传/校验）；无遥测。
- **凭据**：GitHub token 由调用方以环境变量传入，技能文档只写占位符，不落盘到技能目录。
- **写操作**：上传 release / 改仓库属于写操作，执行前必须向用户确认。

## 自检与卸载

```bash
python3 -m unittest discover -s tests    # 文档自检（结构/链接/脱敏）
```

| 形态 | 卸载 |
|---|---|
| 技能目录 | 删 `<落地>/fpk-build-to-fndepot`（`--with-deps` 时连同 `fn-fpk-builder`） |
| 提示词/命令 | 删对应 `.md`/`.toml` |
| 规则文件 | 删对应规则文件；单文件用 `begin/end` 标记整段删除 |

## Windows（PowerShell 5.1+）

```powershell
powershell -ExecutionPolicy Bypass -File scripts\install.ps1 -List
powershell -ExecutionPolicy Bypass -File scripts\install.ps1 -Target claude-code -Apply
powershell -ExecutionPolicy Bypass -File scripts\install.ps1 -Target skills-dir -Dest D:\skills -Apply
```

参数与 `install.sh` 一致：`-Target` `-Dest` `-Dep` `-Link` `-Apply` `-List`；默认 dry-run。
复制时同样排除 `.git/`、`.pushpack*`、`__pycache__/`、`*.pyc`、`node_modules/`。
