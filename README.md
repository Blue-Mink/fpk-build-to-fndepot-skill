# fpk-build-to-fndepot-skill 🚀

> **全链路 Skill：从构建 FPK 到发布至 FnDepot 开源商店**

FPK（飞牛应用包）是 [fnOS](https://www.fnnas.com/) 的第三方应用打包格式。本项目提供一份完整的、经过实战验证的 Skill 文档，帮助你从零开始完成 FPK 应用开发 → 本地安装测试 → 发布到 FnDepot 开源商店 → GitHub Release 的完整流水线。

---

## 📖 内容概要

| 章节 | 内容 |
|------|------|
| **一、全链路总览** | 7 步流程图：分析→骨架→构建→测试→发布→Release→CI |
| **按需路由** | 场景 → 对应 reference 文件（协议/构建规范/操作流程/排查） |
| **核心坑点速览** | 实战五大坑点（wizard/-e、CRC32、volume 变量、Cookie、容器重建） |
| **3.17 Go 原生构建链** | build.sh 流水线（版本单一源/前端嵌入/交叉编译/staging 平铺）+ 6 条硬约束 |
| **4.9 高频迭代部署** | 热替换 / daemon RPC 就地升级 / 卸载重装三态 + 发版约定 |
| **命令速查** | 构建/安装/状态/Release 常用命令表 |
| **references/fndepot-protocol.md** | FnDepot **V2 协议**：fnpack.json 结构、releases 多版本、固定分类、i18n、拆分模式、V2 检查清单、V1→V2 迁移、合格源机制 |
| **references/fpk-build-spec.md** | 开发目录结构、manifest 字段、环境变量表、cmd 脚本模板、权限/资源、wizard、桌面 UI、docker-compose 模板 |
| **references/workflow.md** | 分析→版本号→构建→安装测试→FnDepot 发布→GitHub Release→版本更新逐步操作 |
| **references/troubleshooting.md** | 构建/安装/部署换版/运行时/FnDepot 商店五阶段排查表 |

---

## 🎯 适用人群

- **fnOS 第三方开发者** — 想把自己的应用打包成 FPK 发布到商店
- **自建应用维护者** — 已通过 FnDepot 发布，需要规范的更新流程
- **NAS 爱好者** — 想了解 FPK 打包原理和 FnDepot 商店协议

---

## 📦 整合来源

本 Skill 融合了四大来源的精华：

1. **官方 FPK 开发规范**（开源 skill 参考，仓库名已脱敏）
2. **[EWEDLCM/FnDepot](https://github.com/EWEDLCM/FnDepot)** v1.1.1 — 开源商店协议
3. **[kci-lnk/fn-fpk-builder-skill](https://github.com/kci-lnk/fn-fpk-builder-skill)** — 自动化构建/审计/部署工具链（MIT）
4. **实战踩坑** — 某 4 个实战应用 的完整打包发布经验

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
| 某社区 FnDepot 商店仓库 | 实战案例：4 个实战应用打包发布全过程经验来源 |
| [kci-lnk/fn-fpk-builder-skill](https://github.com/kci-lnk/fn-fpk-builder-skill) | 自动化 FPK 构建工具（fpk.py） |
| [EWEDLCM/FnDepot](https://github.com/EWEDLCM/FnDepot) | FnDepot 开源商店协议与客户端 |
| 官方 FPK 开发规范 skill（已脱敏） | 官方 FPK 开发规范 |

---

## 📄 许可

MIT — 随意使用、修改、分享。

---

> **提示**：完整内容请查看同目录下的 [`SKILL.md`](./SKILL.md) 文件。