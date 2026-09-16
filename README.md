# Spec Kit - PingCode Integration Extension

[![Spec Kit](https://img.shields.io/badge/spec--kit-extension-blue?logo=github)](https://github.com/github/spec-kit)
[![Version](https://img.shields.io/badge/version-2.0.0-green)](https://github.com/HJXArthurAtlas/Spec-Kit-PingCode/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

将 spec-kit 的规格产物(SPEC.md + TASKS.md)转换为 PingCode 工作项层级,并把本地任务完成状态回写到 PingCode。基于 [PingCode CLI](https://github.com/metaphor/pingcode-cli) 命令行工具,扩展本身零代码。

## 功能

- **层级转换**:任务行 → 任务工作项,原生 `--parent` 挂接;Phase 是实施阶段而非交付物,不作映射,仅作为描述内任务清单的分组 checklist
- **单卡模式(默认)**:整个 SPEC.md → 一张用户故事卡(`spec_artifact`),迭代挂这一层;User Story 章节仅作为描述文本嵌入
- **按章节模式(可选)**:spec.md 的每个 `### User Story N` 章节各建一张卡(`story_artifact`),任务按 `[US#]` 标记挂到所属卡,优先级按章节尾注 `P1/P2/P3` 经 `priority_mapping` 映射
- **极简模式(可选)**:与上两种正交(`task_artifact: ""`),不建任务工作项,任务以 checklist 存在于卡描述中
- **原生父子挂接**:PingCode `--parent` 参数,无需链接字段配置
- **Epic/Feature 关联**:建卡前交互选择史诗与特性(`--epic`/`--feature` 可跳过交互),卡 `--parent` 挂特性之下,呈现 史诗 → 特性 → 用户故事 完整层级;可选择直接挂史诗或跳过
- **上下文发现**:探查项目的类型/状态/迭代字典,生成配置片段
- **状态同步**:本地 `[x]`/`[~]`/`[ ]` 标记 → PingCode 状态流转,各卡关联任务全完成后逐卡收尾
- **灵活迭代**:迭代不写死,运行时按解析链确定(参数 > 配置 > 上下文 > 自动发现/询问)
- **幂等防护**:已有映射文件时提供补建/重建/中止选项

## 前置条件

1. **Spec Kit** ≥ 0.1.0
2. **pingcode CLI** 已安装并在 PATH 中(<https://github.com/metaphor/pingcode-cli>),推荐同时把它的 `pingcode` skill 装到你的 agent
3. **已认证**:`pingcode auth login`(或环境变量 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET`)
4. **工作区上下文**:目标项目已解析(`pingcode context set-current-project <项目名>`;命令执行中也会自动处理)

## 安装

```bash
# 在 spec-kit 项目内,从 Release 归档安装
specify extension add pingcode \
  --from https://github.com/HJXArthurAtlas/Spec-Kit-PingCode/releases/download/v2.0.0/pingcode-2.0.0.zip

# 或本地开发安装
specify extension add --dev /path/to/spec-kit-pingcode
```

> spec-kit 社区目录为 discovery-only(仅搜索发现、不支持按名安装),故始终通过 `--from` 指定 Release 归档安装。

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
史诗(Epic) → 特性(Feature)  ← 运行时交互选择(或 --epic/--feature 指定),卡 --parent 挂特性下

单卡模式(默认,story_artifact: ""):
SPEC.md (# 标题 + 正文)   →  用户故事(Story)      ← 迭代挂在这一层
  │
  ├─ ## Phase N 标题      →  任务清单的分组 checklist;SPEC 全文(含 US 章节)嵌入描述
  │
  └─ - [ ] T001 任务行    →  任务(Task),--parent 挂到 story
                             T 编号保留在标题中,便于溯源

按章节模式(story_artifact 非空):
### User Story N 章节    →  各建一张卡(story_artifact 类型),--parent 挂特性
  │                         (spec_artifact 设类型名时先建 spec 父卡,章节卡挂其下)
  │                         优先级:章节尾注 P1/P2/P3 → priority_mapping → --priority
  └─ - [ ] T001 [US1]     →  任务,--parent 挂所属章节卡;无 [US#] 标记的
                             跨故事任务挂 spec 父卡/特性
```

User Story 是可独立交付验证的切片,按章节模式下每个章节对应一张卡;Phase 是实施阶段而非交付物,不作映射,仅作描述分组。单卡模式则把整个 spec 视为一个可交付闭环,对应一张 story 卡。

## 命令

### `/speckit.pingcode.specstoissues`

从 spec 与 tasks 创建完整 PingCode 层级,写映射文件 `specs/<spec-name>/pingcode-mapping.json`。

```bash
/speckit.pingcode.specstoissues                          # 自动检测 spec
/speckit.pingcode.specstoissues --spec 001-user-auth     # 指定 spec
/speckit.pingcode.specstoissues --sprint "Sprint 22"     # 指定迭代
/speckit.pingcode.specstoissues --epic "平台基建" --feature "认证体系"  # 指定关联,跳过交互
/speckit.pingcode.specstoissues --dry-run                # 只看创建计划
```

spec 自动检测优先级:`--spec` 参数 > git 分支名 > 当前目录 > 唯一 spec。

**迭代解析链**:`--sprint` 参数 > config 的 `sprint` > context 当前迭代 > `sprint list --status in_progress`(唯一命中自动选,多个列出询问,零个询问是否挂迭代)。解析结果写入映射文件,sync 不再重复询问。

**Epic/Feature 关联**:创建 story 前交互选择史诗与特性,story 以 `--parent` 挂到特性下(Epic 经父子链隐式关联;无特性时可直接挂史诗,或选择不关联)。`--epic`/`--feature` 按名称/identifier/id 精确匹配,可跳过交互。映射文件结构不变。

### `/speckit.pingcode.discover-context`

探查项目的类型/状态/优先级/迭代字典,输出可粘贴的配置片段,存 `discovered-context.json`。首次接入或类型/状态名对不上时运行。

### `/speckit.pingcode.sync-status`

将 `tasks.md` 的完成标记同步为 PingCode 状态流转,各卡任务全完成后逐卡收尾 story,写日志 `pingcode-sync-log.json`。

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
  spec_artifact: "用户故事"    # SPEC.md 整体的映射类型;按章节模式下可选作父卡
  story_artifact: ""           # 空=单卡模式;设为类型名=按 User Story 章节建卡
  task_artifact: "任务"        # 空=极简模式

defaults:
  spec:
    priority: ""                 # 如 "中" / "高",留空不设置
  story:
    priority: ""                 # 按章节模式章节卡的兜底优先级
  task:
    priority: ""

status_mapping:
  completed: "已完成"          # - [x]
  pending: "未开始"            # - [ ]
  in_progress: "进行中"        # - [~]

priority_mapping:
  p1: "高"                     # 章节 P1/P2/P3 → 优先级名(按章节模式章节卡)
  p2: "中"
  p3: "低"

sync:
  complete_story_when_tasks_done: true
```

环境变量覆盖(优先级高于配置文件):`SPECKIT_PINGCODE_PROJECT`、`SPECKIT_PINGCODE_SPRINT`、`SPECKIT_PINGCODE_SPEC_ARTIFACT`、`SPECKIT_PINGCODE_STORY_ARTIFACT`、`SPECKIT_PINGCODE_TASK_ARTIFACT`、`SPECKIT_PINGCODE_STATUS_COMPLETED/PENDING/IN_PROGRESS`、`SPECKIT_PINGCODE_PRIORITY_P1/P2/P3`。

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
| `command not found: pingcode` | 安装 [PingCode CLI](https://github.com/metaphor/pingcode-cli) 并确认在 PATH |
| `authenticated: false` | `pingcode auth login`,或配置 client 凭证环境变量 |
| workspace context 报错 | `pingcode context set-current-project "<项目名>"` |
| `No cached sprint matched` / 创建要求 current_user_id、current_sprint_id | 迭代字典未填充:管道驱动一次 `pingcode context init`(项目→迭代→用户),再 `context set-current-sprint <ID>`;用户用 `pingcode directory me` 取 ID 后 `set-current-user` |
| 类型名/状态名不识别 | 运行 `/speckit.pingcode.discover-context`,按实际名称修正配置 |
| HTTP 429 | CLI 返回 `x-pc-retry-after` 响应头,按其指示等待后重试 |
| 重复工作项 | 检查 `pingcode-mapping.json`;重跑时选"补建"而非"重建" |

## 参考

[Spec Kit Jira](https://github.com/mbachorik/spec-kit-jira)

## 许可

MIT - 见 [LICENSE](LICENSE)
