# Spec Kit - PingCode Integration Extension

[![Spec Kit](https://img.shields.io/badge/spec--kit-extension-blue?logo=github)](https://github.com/github/spec-kit)
[![Version](https://img.shields.io/badge/version-1.0.0-green)](https://github.com/HJXArthurAtlas/Spec-Kit-PingCode-CLI/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

将 spec-kit 的规格产物(SPEC.md + TASKS.md)转换为 PingCode 工作项层级,并把本地任务完成状态回写到 PingCode。基于 [pingcode-cli](https://github.com/metaphor/pingcode-cli) 命令行工具,**不含 MCP 依赖,扩展本身零代码**。

## 功能

- **层级转换**:SPEC.md → 用户故事,Phase → story 描述中的分组 checklist(2 层模式,默认),任务行 → 任务工作项
- **3 层模式(可选)**:Phase 建独立工作项(SPEC → Phase → 任务)
- **极简模式(可选)**:只建 story,任务以 checklist 存在于描述中
- **原生父子挂接**:PingCode `--parent` 参数,无需链接字段配置
- **上下文发现**:探查项目的类型/状态/迭代字典,生成配置片段
- **状态同步**:本地 `[x]`/`[~]`/`[ ]` 标记 → PingCode 状态流转,任务全完成自动收尾 story
- **灵活迭代**:迭代不写死,运行时按解析链确定(参数 > 配置 > 上下文 > 自动发现/询问)
- **幂等防护**:已有映射文件时提供补建/重建/中止选项

## 前置条件

1. **Spec Kit** ≥ 0.1.0
2. **pingcode CLI** 已安装并在 PATH 中(<https://github.com/metaphor/pingcode-cli>),推荐同时把它的 `pingcode` skill 装到你的 agent
3. **已认证**:`pingcode auth login`(或环境变量 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET`)
4. **工作区上下文**:目标项目已解析(`pingcode context set-current-project <项目名>`;命令执行中也会自动处理)

## 安装

```bash
# 在 spec-kit 项目内
specify extension add pingcode

# 或本地开发安装
specify extension add --dev /path/to/spec-kit-pingcode
```

## 快速开始

```bash
# 0. 生成 spec 与任务
/speckit.specify 做一个用户认证模块
/speckit.plan
/speckit.tasks

# 1. 配置(复制模板,填 project)
cp .specify/extensions/pingcode/pingcode-config.template.yml \
   .specify/extensions/pingcode/pingcode-config.yml

# 2. (可选)探查项目字典,校准类型/状态名
/speckit.pingcode.discover-context

# 3. 创建 PingCode 工作项层级
/speckit.pingcode.specstoissues

# 4. 本地实施,勾选 tasks.md 中的任务

# 5. 同步完成状态到 PingCode
/speckit.pingcode.sync-status
```

## 层级映射

```text
SPEC.md (# 标题 + 正文)   →  用户故事(Story)      ← 迭代挂在这一层
  │
  ├─ ## Phase N 标题      →  (默认)嵌入 story 描述的分组 checklist
  │                          可选:建为独立工作项(3 层模式)
  │
  └─ - [ ] T001 任务行    →  任务(Task),--parent 挂到 story
                             T 编号保留在标题中,便于溯源
```

一个 spec 是一个可交付闭环,对应一张 story 卡;Phase 是实施阶段而非交付物,因此默认不建独立工作项。

## 命令

### `/speckit.pingcode.specstoissues`

从 spec 与 tasks 创建完整 PingCode 层级,写映射文件 `specs/<spec-name>/pingcode-mapping.json`。

```bash
/speckit.pingcode.specstoissues                          # 自动检测 spec
/speckit.pingcode.specstoissues --spec 001-user-auth     # 指定 spec
/speckit.pingcode.specstoissues --sprint "Sprint 22"     # 指定迭代
/speckit.pingcode.specstoissues --dry-run                # 只看创建计划
```

spec 自动检测优先级:`--spec` 参数 > git 分支名 > 当前目录 > 唯一 spec。

**迭代解析链**:`--sprint` 参数 > config 的 `sprint` > context 当前迭代 > `sprint list --status in_progress`(唯一命中自动选,多个列出询问,零个询问是否挂迭代)。解析结果写入映射文件,sync 不再重复询问。

### `/speckit.pingcode.discover-context`

探查项目的类型/状态/优先级/迭代字典,输出可粘贴的配置片段,存 `discovered-context.json`。首次接入或类型/状态名对不上时运行。

### `/speckit.pingcode.sync-status`

将 `tasks.md` 的完成标记同步为 PingCode 状态流转,全完成后收尾 story,写日志 `pingcode-sync-log.json`。

```bash
/speckit.pingcode.sync-status              # 自动检测
/speckit.pingcode.sync-status --dry-run    # 只看流转计划
```

**方向性规则**:本地 → PingCode 单向执行。本地勾选而远端未完成 → 流转;远端已推进而本地未勾选 → 不回退,仅记入差异报告由人工确认。

## 配置

`.specify/extensions/pingcode/pingcode-config.yml`(模板:`pingcode-config.template.yml`):

```yaml
project: "网运通项目组"        # 必填,项目名或标识符
sprint: ""                    # 可选,留空走运行时解析链

mapping:
  spec_artifact: "用户故事"    # SPEC.md 的映射类型
  phase_artifact: ""           # 空=嵌入描述;设为特性类型名启用 3 层
  task_artifact: "任务"        # 空=极简模式

status_mapping:
  completed: "已完成"          # - [x]
  pending: "未开始"            # - [ ]
  in_progress: "进行中"        # - [~]

sync:
  complete_story_when_tasks_done: true
```

环境变量覆盖(优先级高于配置文件):`SPECKIT_PINGCODE_PROJECT`、`SPECKIT_PINGCODE_SPRINT`、`SPECKIT_PINGCODE_SPEC_ARTIFACT`、`SPECKIT_PINGCODE_PHASE_ARTIFACT`、`SPECKIT_PINGCODE_TASK_ARTIFACT`、`SPECKIT_PINGCODE_STATUS_COMPLETED/PENDING/IN_PROGRESS`。

## 任务完成标记

| 标记 | 状态 | 默认 PingCode 状态 |
|---|---|---|
| `- [x]` | completed | 已完成 |
| `- [ ]` | pending | 未开始 |
| `- [~]` | in_progress | 进行中 |

状态名按项目可配置;执行时名称不匹配会按 `state_type`(pending/started/completed)兜底解析。

## 产物文件

| 文件 | 作用 |
|---|---|
| `specs/<name>/pingcode-mapping.json` | spec/任务 ↔ 工作项的映射,specstoissues 写入、sync-status 消费 |
| `specs/<name>/pingcode-sync-log.json` | 每次同步的流转/差异/进度记录 |
| `.specify/extensions/pingcode/discovered-context.json` | 项目字典探查结果 |

## 故障排除

| 症状 | 处理 |
|---|---|
| `command not found: pingcode` | 安装 [pingcode-cli](https://github.com/metaphor/pingcode-cli) 并确认在 PATH |
| `authenticated: false` | `pingcode auth login`,或配置 client 凭证环境变量 |
| workspace context 报错 | `pingcode context set-current-project "<项目名>"` |
| `No cached sprint matched` / 创建要求 current_user_id、current_sprint_id | 迭代字典未填充:管道驱动一次 `pingcode context init`(项目→迭代→用户),再 `context set-current-sprint <ID>`;用户用 `pingcode directory me` 取 ID 后 `set-current-user` |
| 类型名/状态名不识别 | 运行 `/speckit.pingcode.discover-context`,按实际名称修正配置 |
| HTTP 429 | CLI 有 `x-pingcode` 限流提示,按提示等待后重试 |
| 重复工作项 | 检查 `pingcode-mapping.json`;重跑时选"补建"而非"重建" |

## 与 spec-kit-jira 的差异

| | spec-kit-jira | spec-kit-pingcode |
|---|---|---|
| 交互层 | Jira MCP server 工具调用 | `pingcode` CLI bash 命令 |
| 默认层级 | Epic → Story → Task(3 层) | Story → Task(2 层),Phase 嵌入描述 |
| 父子挂接 | Epic Link / Parent / 关系链接 | `--parent` 参数 |
| 字段发现 | discover-fields(自定义字段) | discover-context(类型/状态/迭代字典) |
| 状态流转 | transition API | `workitem update --state` |


## 许可

MIT - 见 [LICENSE](LICENSE)
