---
name: agent_creator
description: "创建新的 CoPaw 智能体（workspace）并注册到前端可见列表；按需求自动装配技能：先查本地 skill_pool，再用 clawhub 在线检索，若仍无则生成新技能。所有写入必须先征得用户同意并做 JSON 校验与同目录备份。"
metadata: { "copaw": { "emoji": "🧩" }, "skill_version": "0.2.1" }
---

# 创建 CoPaw 智能体（含技能自动装配）

## 什么时候用

当用户需要在 CoPaw 里新增一个独立智能体（独立 workspace、独立 skills/记忆/配置）时使用，例如：

- “帮我创建一个负责 X 的新智能体”
- “我想要一个专门写报告/做数据分析/运营增长的 agent”
- “创建 agent 并确保在 CoPaw 前端能看到（注册到列表）”

## Docker 环境适配说明

### 路径映射
- **工作区路径**：在 Docker 环境中，智能体工作区通常位于 `/app/working/workspaces/<agent_id>/`
- **技能池路径**：技能池位于 `/app/working/skill_pool/`
- **配置文件**：主配置文件位于 `/app/working/config.json`

### 权限处理
- Docker 容器内以 root 用户运行，无需额外权限配置
- 文件操作使用容器内路径，避免宿主机路径混淆

### 网络访问
- Docker 容器内可访问外部网络（用于在线技能检索）
- 确保容器有 DNS 解析能力（用于 npx clawhub 命令）

### 环境变量
- `HOME` 环境变量应指向 `/root`
- `PATH` 应包含 `/app/venv/bin` 以使用 copaw 命令

## 你必须遵守的写入协议（硬规则）

本技能涉及创建目录与修改 JSON 配置：

1) **默认只读（dry-run）**：先输出将执行的变更清单（会创建哪些目录、会修改哪些文件、会导入哪些技能）  
2) **每次写入前必须先询问用户并得到明确同意**（每次写都问）  
3) 一旦用户允许写入，你才可以使用 `--write` 执行落盘操作，并确保：
   - 写入前 **JSON 格式校验**（保证生成 JSON 合法可解析）
   - 写入前 **原文件同目录备份**：`<file>.bak.<timestamp>`
   - 建议“临时文件 + 原子替换”避免写一半损坏

脚本也有硬保护：未带 `--write` 时绝不会写入。

## 技能流程（必须包含）

### Step 0：理解需求 → 抽取能力关键词

从用户描述中提取关键词（任务、领域、工具、输出物、渠道等），用于技能匹配与在线检索。

### Step 1：本地技能池匹配并导入

在本地 skill pool 中检索（通常位于 `~/.copaw/skill_pool`）：
- 优先读取 `skill_pool/skill.json`（技能清单索引）
- 辅助读取各技能 `SKILL.md`

若匹配到合适技能，则导入到新智能体 workspace（复制到 `<workspace>/skills/<skill_name>` 并在 `<workspace>/skill.json` 启用）。

### Step 2：在线技能库检索并导入

使用在线技能库检索（优先 clawhub）：
- 若机器可用 `npx`：先运行 `npx clawhub search "<keyword>"`
- 若不可用：改用 WebSearch / 手动指定候选

导入规则：
- 与 Step 1 已导入的技能（同名或高度相似）不重复导入
- 下载后必须先读取候选技能的 `SKILL.md` 做二次确认（功能匹配 + 风险提示）

### Step 3：若 1 和 2 都没导入任何技能 → 为智能体生成新技能

如果没有任何技能可用，则为该智能体生成一个最小可用新技能（Skill-Creator 风格）：
- 仅包含必要的 `SKILL.md`（保持简洁，避免冗长）
- 如需确定性执行，可生成 `scripts/` 中的脚本骨架

### Step 4：使用 find-skills 技能查找更多合适技能

调用 find-skills 技能为新的智能体查找合适职能的技能并添加进新的智能体的技能表里：
- 使用 `npx clawhub search "<keyword>"` 搜索相关技能
- 评估技能匹配度，选择最合适的 2-3 个技能
- 下载并导入到智能体工作区的 skills 目录
- 在 skill.json 中启用这些技能

### Step 5：为智能体设置模型配置

创建智能体后，为其设置默认的模型配置（与 default agent 一致）：
- 在 `agent.json` 中添加 `active_model` 字段
- 设置 `provider_id: "minimax-custom"`
- 设置 `model: "MiniMax-M2.7"`（默认模型）

### Step 6：添加多智能体协作技能

为智能体添加多智能体协作技能，确保智能体可以与其他智能体通信：
- 检查是否已存在 multi_agent_collaboration 技能
- 如不存在，从技能池或在线搜索并导入
- 在 skill.json 中启用该技能

### Step 7：测试智能体通信（收尾操作）

使用多智能体协作技能尝试与新建的智能体沟通，以保证智能体创建正常：
- 从当前智能体向新建智能体发送测试消息
- 验证智能体可以正常接收和响应消息
- 记录测试结果，确保智能体功能正常

## CoPaw 的“智能体列表注册”位置

不同版本可能不同：

- 在本机 CoPaw v1.0.x 中，智能体注册表位于：`~/.copaw/config.json` 的 `agents.profiles` 与 `agents.agent_order`（前端 Console 也通常从这里读取）
- 如果你的版本存在独立 `agents.json`，脚本会优先使用它；否则回退到 `config.json`

## 使用方式（建议）

### 1) 先 dry-run（只读）

```bash
python scripts/create_agent.py --spec-md ./agent_spec.md --dry-run
```

你需要把 dry-run 的输出贴给用户审阅，并明确问：
> “是否允许我按该计划写入（创建 workspace、修改注册表、导入技能）？”

### 2) 用户允许后再写入

```bash
python scripts/create_agent.py --spec-md ./agent_spec.md --write
```

### 3) 完整流程（推荐）

1. **dry-run**：运行 `--dry-run` 查看计划
2. **用户确认**：询问用户是否同意
3. **写入**：运行 `--write` 执行创建
4. **技能查找**：自动调用 find-skills 查找更多技能
5. **模型配置**：自动设置模型配置
6. **协作技能**：添加多智能体协作技能
7. **测试通信**：发送测试消息验证智能体功能

## agent_spec.md 示例

建议用户用一个 markdown 来描述要创建的 agent，支持 YAML frontmatter：

```md
---
id: growth_agent
name: 增长运营智能体
description: 负责拉新、转化分析、内容增长与渠道策略
keywords: [增长, 运营, 小红书, SEO, 数据分析]
---

补充说明：希望优先导入本地已有的 news / browser / docx 等技能，如没有则去 clawhub 搜索。
```

