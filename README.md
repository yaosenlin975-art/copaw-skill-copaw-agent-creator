# CoPaw Skill：创建智能体（含技能自动装配）

[English](README_en.md) | 中文

本仓库提供一个面向 **CoPaw（AgentScope）** 的自定义技能：**copaw_agent_creator**。  
它用于**创建新的 CoPaw 智能体（workspace）**，并将其注册到 CoPaw 前端可见的智能体列表中；同时按你的要求加入“技能自动装配”流程：

1. 先从本地技能池（`~/.copaw/skill_pool`）按需求匹配并导入技能到新智能体  
2. 再通过在线技能库检索（优先 `npx clawhub search`）导入合适技能（与步骤 1 已导入/高度相似的不重复导入）  
3. 若 1 和 2 没导入任何技能，则为新智能体生成一个最小可用的新技能（Skill Creator 风格）

## 重要：写入安全协议

本技能涉及创建目录与修改 JSON 配置，因此**默认只读（dry-run）**，只有在你明确允许写入且执行时带 `--write` 才会落盘。

每次写入必须遵守：

1) **先询问你并得到同意**（每次写入都要问）  
2) 写入前必须：
- **格式校验**（JSON 必须合法可解析）
- **原文件备份（同目录）**：`<file>.bak.<timestamp>`（用于回滚）
- 建议采用“临时文件 + 原子替换”避免写一半损坏

脚本内部也做了硬保护：不带 `--write` 时不会写任何文件。

## 快速开始

> 先运行 dry-run 看看会改哪些文件：

```bash
python scripts/create_agent.py --spec-md ./agent_spec.md --dry-run
```

> 你确认无误并允许写入后再执行：

```bash
python scripts/create_agent.py --spec-md ./agent_spec.md --write
```

## 仓库内容

- `SKILL.md`：给 CoPaw agent 使用的技能说明书（流程、边界、如何提问获取写入授权）
- `scripts/create_agent.py`：核心脚本（创建 workspace + 注册 + 技能装配）
- `template/`：内置智能体模板（包含 AGENTS.md、SOUL.md、RULES.md 等）

## 模板机制

创建新 workspace 时，模板优先级：
1. `default` workspace（用户自定义模板）
2. `template/` 目录（内置模板，v0.2.2+ 新增）
3. 硬编码最小文件集

**内置模板特点：**
- 包含完整的 AGENTS.md / SOUL.md / PROFILE.md / MEMORY.md / RULES.md / HEARTBEAT.md / BOOTSTRAP.md
- `RULES.md`（Agent 死规定）自动注册到全局 `system_prompt_files`，创建后即生效
- 无需维护 default workspace 即可获得标准模板

## Docker 环境适配说明

在 Docker 容器中运行 CoPaw 时，路径映射与宿主机不同：

| 项目 | 宿主机路径 | Docker 容器内路径 |
|------|-----------|-----------------|
| 工作区根目录 | `~/.copaw/workspaces/` | `/app/working/workspaces/` |
| 技能池目录 | `~/.copaw/skill_pool/` | `/app/working/skill_pool/` |
| 主配置文件 | `~/.copaw/config.json` | `/app/working/config.json` |

权限处理：容器内以 root 用户运行，无需额外权限配置。  
网络访问：容器内可访问外部网络（用于 `npx clawhub` 在线检索）。  
环境变量：`HOME` 应指向 `/root`，`PATH` 应包含 `/app/venv/bin`。

## License

Apache-2.0，见 [LICENSE](LICENSE)。

"# copaw-skill-copaw-agent-creator" 
