# fpk-build-to-fndepot-skill 🚀

> **全链路 Skill：从构建 FPK 到发布至 FnDepot 开源商店**

FPK（飞牛应用包）是 [fnOS](https://www.fnnas.com/) 的第三方应用打包格式。本项目提供一份完整的、经过实战验证的 Skill 文档，帮助你从零开始完成 FPK 应用开发 → 本地安装测试 → 发布到 FnDepot 开源商店 → GitHub Release 的完整流水线。

---

## 📖 内容概要

| 章节 | 内容 |
|------|------|
| **一、全链路总览** | 7 步流程图：分析→骨架→构建→测试→发布→Release→CI |
| **二、FnDepot 开源商店协议** | 仓库规范、目录结构、fndepot.json 字段详解、架构匹配规则 |
| **三、FPK 构建规范（完整版）** | 开发目录结构、manifest 字段、环境变量表、cmd/main 脚本模板、权限/资源声明、wizard 向导、桌面 UI、实战五大坑点 |
| **四、操作流程** | 分析→版本号→自动化构建→手动构建→安装测试→FnDepot 发布→GitHub Release→版本更新 |
| **五、故障排查** | 构建/安装/运行时/FnDepot 商店四类常见问题系统化排查 |
| **六、快捷查询** | 命令速查表、发布前检查清单、README 模板 |

---

## 🎯 适用人群

- **fnOS 第三方开发者** — 想把自己的应用打包成 FPK 发布到商店
- **自建应用维护者** — 已通过 FnDepot 发布，需要规范的更新流程
- **NAS 爱好者** — 想了解 FPK 打包原理和 FnDepot 商店协议

---

## 📦 整合来源

本 Skill 融合了四大来源的精华：

1. **[jianyun8023/fnos-fpk-dev](https://github.com/jianyun8023/my-skills/tree/main/skills/tooling/fnos-fpk-dev)** — 官方 FPK 开发规范
2. **[EWEDLCM/FnDepot](https://github.com/EWEDLCM/FnDepot)** v1.1.1 — 开源商店协议
3. **[kci-lnk/fn-fpk-builder-skill](https://github.com/kci-lnk/fn-fpk-builder-skill)** — 自动化构建/审计工具链
4. **实战踩坑** — TREK / global-radio / fn-knock / kms-activator 的完整打包发布经验

---

## 🚀 快速开始

### 前提条件

- 一台运行 **fnOS 0.9.0+** 的飞牛 NAS
- 安装 **fnpack** CLI 工具（详见 Skill 3.5 节）
- 一个 **GitHub** 账号（用于 FnDepot 发布）
- 你的应用已确定：类型 / 端口 / 架构 / 持久化需求

### 最短路径

```bash
# 1. 阅读完整 Skill
cat SKILL.md

# 2. 创建 FPK 骨架
mkdir -p my-app-fpk/{cmd,config,wizard,app/ui/images}

# 3. 按规范编写 manifest + cmd/main + app/docker-compose.yml
# 4. 构建：打包 app.tgz → CRC32 → 更新 manifest → fnpack pack
# 5. 测试：install-fpk ./app.fpk
# 6. 发布：推送到 FnDepot 仓库 + GitHub Release
```

---

## 🔗 相关仓库

| 仓库 | 说明 |
|------|------|
| [Blue-Mink/FnDepot](https://github.com/Blue-Mink/FnDepot) | 实战案例：TREK / global-radio / fn-knock / kms-activator 已入库 |
| [kci-lnk/fn-fpk-builder-skill](https://github.com/kci-lnk/fn-fpk-builder-skill) | 自动化 FPK 构建工具（fpk.py） |
| [EWEDLCM/FnDepot](https://github.com/EWEDLCM/FnDepot) | FnDepot 开源商店协议与客户端 |
| [jianyun8023/my-skills](https://github.com/jianyun8023/my-skills) | 官方 FPK 开发规范 |

---

## 📄 许可

MIT — 随意使用、修改、分享。

---

> **提示**：完整内容请查看同目录下的 [`SKILL.md`](./SKILL.md) 文件。