# Spec Kit - PingCode Integration Extension

[![Spec Kit](https://img.shields.io/badge/spec--kit-extension-blue?logo=github)](https://github.com/github/spec-kit)
[![Version](https://img.shields.io/badge/version-2.2.1-green)](https://github.com/HJXArthurAtlas/Spec-Kit-PingCode/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

将 spec-kit 的规格产物映射为 PingCode 卡片——只有 SPEC.md 整体与 `### User Story N` 章节两级建卡,Phase 与任务以 checklist 折叠进卡描述;本地任务全部勾选后自动收尾卡片。基于 [PingCode CLI](https://github.com/metaphor/pingcode-cli) 命令行工具,扩展本身零代码。

## 功能

- **两级映射**:仅 SPEC.md 整体与 `### User Story N` 章节建卡(`spec_artifact`/`story_artifact`);Phase 与任务永不建工作项,以 checklist 折叠进所属卡描述
- **单卡模式(默认)**:只配 `spec_artifact`,整个 spec(章节 + 任务清单)折叠进一张卡
- **按章节模式**:`story_artifact` 配置后每个章节一张卡、内容各自携带,spec 卡只保留非章节正文
- **章节优先级**:章节尾注 `P1/P2/P3` 经 `priority_mapping` 映射为项目优先级
- **原生父子挂接**:PingCode `--parent` 参数,无需链接字段配置
- **父卡关联**:按顶层卡类型交互选择上一级——故事类卡挂特性下、特性类卡挂史诗下(`--epic`/`--feature` 可跳过交互),呈现 史诗 → 特性 → 用户故事 层级
- **上下文发现**:探查项目的类型/状态/迭代字典,生成配置片段
- **一键初始化**:`/speckit.pingcode.init` 交互选择项目/迭代/映射类型/状态/优先级,直接生成配置文件
- **状态同步**:本地任务全部 `[x]` 后自动收尾所属卡(单向,不回退)
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
  --from https://github.com/HJXArthurAtlas/Spec-Kit-PingCode/releases/download/v2.2.1/pingcode-2.2.1.zip

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

# 1. 交互式初始化:选项目/迭代/映射类型,直接生成配置文件
/speckit.pingcode.init
#    手动替代:复制 pingcode-config.template.yml 为 pingcode-config.yml 填 project;
#    名称校准用 /speckit.pingcode.discover-context

# 2. 创建 PingCode 卡片层级
/speckit.pingcode.specstoissues

# 3. 本地实施,勾选 tasks.md 中的任务

# 4. 任务全部勾选后,收尾 PingCode 卡片
/speckit.pingcode.sync-status
```

## 层级映射

```text
史诗(Epic) → 特性(Feature) → 用户故事(Story)    ← PingCode 需求树

只配置 spec_artifact(默认 "用户故事",也可配 "特性" 等):
SPEC.md 整体(含 US 章节、任务清单)  →  一张卡,内容全部折叠进描述

spec_artifact + story_artifact 都配置:
SPEC.md 非 User Story 正文 + 跨故事任务清单 → spec 卡(如 特性)
### User Story N 章节 + 自己的任务清单      → 章节卡(如 用户故事),--parent 挂 spec 卡下

## Phase N / - [ ] T001 任务行  →  永不建卡,以 checklist 文本折叠进所属卡描述
```

spec 与 User Story 是需求树上真实的交付单元;Phase 是实施分组、任务是执行项,只作为卡内 checklist 存在。顶层卡按类型挂上一级:故事类挂所选特性下,特性类挂所选史诗下(均可跳过);迭代挂在建卡层级。

## 命令

### `/speckit.pingcode.init`

交互式初始化:认证自检 → 选项目 → 按项目实际字典选映射类型/收尾状态/优先级名 → 选迭代(可留空走运行时解析链) → 生成 `pingcode-config.yml`。已存在的配置逐键展示差异,确认覆盖(旧文件备份 `.bak`)。

```bash
/speckit.pingcode.init                  # 全交互
/speckit.pingcode.init --project WYT    # 跳过项目选择
/speckit.pingcode.init --dry-run        # 只预览生成的配置
```

### `/speckit.pingcode.specstoissues`

从 spec 与 tasks 创建 PingCode 卡片层级,写映射文件 `specs/<spec-name>/pingcode-mapping.json`。

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

探查项目的类型/状态/优先级/迭代字典,输出可粘贴的配置片段,存 `discovered-context.json`。类型/状态名对不上时运行;首次接入优先用 `/speckit.pingcode.init`。

### `/speckit.pingcode.sync-status`

本地任务全部勾选后,自动将所属卡流转到完成态(spec-story 模式逐 story 卡判断,spec-only 模式判断 spec 卡),写日志 `pingcode-sync-log.json`。

```bash
/speckit.pingcode.sync-status              # 自动检测
/speckit.pingcode.sync-status --dry-run    # 只看流转计划
```

**方向性规则**:本地 → PingCode 单向执行。本地勾选而远端未完成 → 流转;远端已推进而本地未勾选 → 不回退,仅记入差异报告由人工确认。

## 配置

`.specify/extensions/pingcode/pingcode-config.yml` —— 推荐用 `/speckit.pingcode.init` 交互生成;手动方式复制模板 `pingcode-config.template.yml` 填写:

```yaml
project: "网运通项目组"        # 必填,项目名或标识符
sprint: ""                    # 可选,留空走运行时解析链

mapping:
  spec_artifact: "用户故事"    # SPEC.md 整体的映射类型(也可配 "特性" 等)
  story_artifact: ""           # 空=章节并入 spec 卡描述;设类型名=每章节一张卡

priority_mapping:
  p1: "高"                     # 章节 P1/P2/P3 → 优先级名
  p2: "中"
  p3: "低"

defaults:
  spec:
    priority: ""                 # 如 "中" / "高",留空不设置
  story:
    priority: ""                 # 章节卡兜底优先级(章节无 P 编号时)

status_mapping:
  completed: "已完成"          # 卡的任务全部 [x] 后流转到该状态

sync:
  complete_story_when_tasks_done: true
```

环境变量覆盖(优先级高于配置文件):`SPECKIT_PINGCODE_PROJECT`、`SPECKIT_PINGCODE_SPRINT`、`SPECKIT_PINGCODE_SPEC_ARTIFACT`、`SPECKIT_PINGCODE_STORY_ARTIFACT`、`SPECKIT_PINGCODE_PRIORITY_P1/P2/P3`、`SPECKIT_PINGCODE_STATUS_COMPLETED`。

## 任务完成标记

任务不建工作项,勾选标记只用于卡描述的 checklist 文本与收尾判断:

| 标记 | 含义 |
|---|---|
| `- [x]` | 完成(所属卡全部 `[x]` 时可收尾) |
| `- [~]` | 进行中 |
| `- [ ]` | 未开始 |

收尾目标状态按项目可配置(`status_mapping.completed`);执行时名称不匹配会按 `state_type=completed` 兜底解析。

## 产物文件

| 文件 | 作用 |
|---|---|
| `specs/<name>/pingcode-mapping.json` | spec/story 卡的映射,specstoissues 写入、sync-status 消费 |
| `specs/<name>/pingcode-sync-log.json` | 每次同步的收尾记录与进度 |
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
| 重复卡片 | 检查 `pingcode-mapping.json`;重跑时选"补建"而非"重建" |

## 参考

[Spec Kit Jira](https://github.com/mbachorik/spec-kit-jira)

## 许可

MIT - 见 [LICENSE](LICENSE)
