# Spec Kit - PingCode Integration Extension

[![Spec Kit](https://img.shields.io/badge/spec--kit-extension-blue?logo=github)](https://github.com/github/spec-kit)
[![Version](https://img.shields.io/badge/version-2.5.4-green)](https://github.com/HJXArthurAtlas/Spec-Kit-PingCode/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

将 spec-kit 的规格产物映射为 PingCode 需求树——只有 SPEC.md 整体与 `### User Story N` 章节两级建卡,映射层级可配置(相邻不跳级);建卡时自顶向下确认 spec 上方的未映射祖先层(需求必问,中间工作项层只选已有);本地任务全部勾选后自动收尾卡片。基于 [PingCode CLI](https://github.com/metaphor/pingcode-cli) 命令行工具,扩展本身零代码。

## 功能

- **最多三级映射**:SPEC.md 整体、`### User Story N` 章节、任务行(`task_artifact`,可选)建工作项卡;Phase 永不建卡、不进卡描述
- **任务卡**:`task_artifact` 配置后每个任务行建卡挂所属 story/spec 卡下,由 `after_tasks` hook 在 `/speckit.tasks` 后补建(condition 门控:未配置 `task_artifact` 时 hook 不触发);本地勾选后由 sync-status 按配置流转
- **单卡模式**:只配 `spec_artifact`,整个 spec(含章节)折叠进一张卡
- **按章节模式(默认)**:`story_artifact` 配置后每个章节一张卡、内容各自携带,spec 卡只保留非章节正文
- **章节优先级**:章节尾注 `P1/P2/P3` 经 `priority_mapping` 映射为项目优先级
- **原生父子挂接**:PingCode `--parent` 参数,无需链接字段配置
- **层级映射**:spec/story 映射到需求树相邻层级(史诗=1/特性=2/用户故事=3,不跳级);spec 上方的未映射祖先层自顶向下确认——需求(idea)必问,史诗层询问新建同名或选已有,其余中间层只选已有(`--idea`/`--epic`/`--product` 可跳过交互)
- **上下文发现**:探查项目的类型/状态/迭代字典,生成配置片段
- **一键初始化**:`/speckit.pingcode.init` 交互选择项目/产品/迭代/映射类型/状态/优先级,直接生成配置文件
- **状态同步**:本地任务全部 `[x]` 后自动收尾所属卡(单向,不回退)
- **灵活迭代**:迭代不写死,运行时按解析链确定(参数 > 配置 > 上下文 > 自动发现/询问)
- **幂等防护**:登记中已有条目时提供补建/重建/中止选项,已创建的卡片不重复建
- **统一映射登记**:所有 spec 的制品生成状态、卡片 id 与卡片状态集中在 `specs/pingcode-mapping.json`

## 前置条件

1. **Spec Kit** ≥ 0.1.0
2. **pingcode CLI** 已安装并在 PATH 中(<https://github.com/metaphor/pingcode-cli>),推荐同时把它的 `pingcode` skill 装到你的 agent
3. **已认证**:`pingcode auth login`(或环境变量 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET`)
4. **工作区上下文**:目标项目已解析(`pingcode context set-current-project <项目名>`;命令执行中也会自动处理)

## 安装

```bash
# 在 spec-kit 项目内,从 Release 归档安装
specify extension add pingcode \
  --from https://github.com/HJXArthurAtlas/Spec-Kit-PingCode/releases/download/v2.5.4/pingcode-2.5.4.zip

# 或本地开发安装
specify extension add --dev /path/to/spec-kit-pingcode
```

> spec-kit 社区目录为 discovery-only(仅搜索发现、不支持按名安装),故始终通过 `--from` 指定 Release 归档安装。

## 快速开始

```bash
# 0. 交互式初始化:选项目/产品/迭代/映射类型,直接生成配置文件
/speckit.pingcode.init
#    手动替代:复制 pingcode-config.template.yml 为 pingcode-config.yml 填 project;
#    名称校准用 /speckit.pingcode.discover-context

# 1. 生成 spec 与方案;/speckit.plan 完成后默认提示创建 PingCode 卡片
/speckit.specify 做一个用户认证模块
/speckit.plan
#    → 确认后选择需求与挂靠史诗,创建 特性 → 用户故事 卡片层级
#    手动补跑:/speckit.pingcode.specstoissues

# 2. 生成本地任务清单
/speckit.tasks

# 3. 本地实施,勾选 tasks.md 中的任务

# 4. 任务全部勾选后,收尾 PingCode 卡片
/speckit.pingcode.sync-status
```

## 层级映射

```text
PingCode 需求树:  需求(idea,产品级) → 史诗(1) → 特性(2) → 用户故事(3) → 任务(4)
spec-kit 制品:    SPEC.md 整体 → User Story 章节 → 任务行(task_artifact 可选建卡)

spec_artifact + story_artifact(+ task_artifact)决定制品栈落点(必须相邻,不跳级):

特性 + 用户故事(默认):
需求(询问) → 史诗(询问:新建同名/选已有) → 特性(spec 卡) → 用户故事(story 卡)

史诗 + 特性:
需求(询问) → 史诗(spec 卡) → 特性(story 卡) → 故事/任务(永不问)

用户故事 + 留空(单卡):
需求 → 史诗 → 特性(询问:只选已有) → 用户故事(spec 卡)

## Phase N  →  永不建工作项,也不进卡描述
## - [ ] T001 任务行  →  task_artifact 配置时建任务卡挂所属卡下;未配置时仅本地 checklist
```

规则:设 spec 位于第 n 层,第 1..n-1 层未映射祖先自顶向下确认——需求(idea)必问,记录式关联(idea 是产品域实体,不作工作项父级,登记于统一映射文件并在卡描述注记);史诗层未映射时询问**新建同名**或**选已有**,其余中间层只选已有;第 n 层以下永不询问。迭代挂在 spec/story 卡层级。

## 命令

### `/speckit.pingcode.init`

交互式初始化:认证自检 → 选项目 → 选产品(需求关联用) → 按项目实际字典选映射类型(含需求树层级归类与不跳级校验)/收尾状态/优先级名 → 选迭代(可留空走运行时解析链)并初始化 CLI 上下文 → 生成 `pingcode-config.yml`。已存在的配置逐键展示差异,确认覆盖(旧文件备份 `.bak`)。

```bash
/speckit.pingcode.init                  # 全交互
/speckit.pingcode.init --project WYT    # 跳过项目选择
/speckit.pingcode.init --dry-run        # 只预览生成的配置
```

### `/speckit.pingcode.specstoissues`

从 spec 创建 PingCode 卡片层级(需求 → Epic → 特性 → 用户故事),写映射文件 `specs/<spec-name>/pingcode-mapping.json`。不读取 `tasks.md`(任务进度由 sync-status 从本地解析)。

```bash
/speckit.pingcode.specstoissues                          # 自动检测 spec
/speckit.pingcode.specstoissues --spec 001-user-auth     # 指定 spec
/speckit.pingcode.specstoissues --sprint "Sprint 22"     # 指定迭代
/speckit.pingcode.specstoissues --product "网运通" --idea "SLC-12"  # 指定产品与需求,跳过交互
/speckit.pingcode.specstoissues --idea "SLC-12" --epic "WYT-100"    # 需求与史诗都指定,零交互
/speckit.pingcode.specstoissues --dry-run                # 只看创建计划
```

spec 自动检测优先级:`--spec` 参数 > git 分支名 > 当前目录 > 唯一 spec。

**迭代解析链**:`--sprint` 参数 > config 的 `sprint` > context 当前迭代 > `sprint list --status in_progress`(唯一命中自动选,多个列出询问,零个询问是否挂迭代)。解析结果写入映射文件,sync 不再重复询问。

**动态祖先关联**:设 spec 映射在第 n 层,自顶向下确认第 1..n-1 层——需求(idea)必问,记录式关联(idea 是产品域实体,不作工作项父级;登记于统一映射文件并在卡描述注记 `> 来源需求:`);史诗层未映射时询问**新建同名史诗**或**选择已有史诗**(新建的描述内记来源需求),其余中间层只选已有。候选清单**全量列出、只剔终态**(已完成/已关闭不列),不做归属预判或父级预筛。`--idea`/`--epic`/`--product` 按名称/identifier/id 精确匹配跳过对应交互。

### `/speckit.pingcode.discover-context`

探查项目的类型/状态/优先级/迭代字典,输出可粘贴的配置片段,存 `discovered-context.json`。类型/状态名对不上时运行;首次接入优先用 `/speckit.pingcode.init`。

### `/speckit.pingcode.sync-status`

本地任务勾选后同步状态到 PingCode(均受 `sync.complete_story_when_tasks_done` 控制):配置 `task_artifact` 时逐任务卡流转 `- [x]` 的任务;story/spec 卡在关联任务全部勾选后收尾。写日志 `pingcode-sync-log.json`。

```bash
/speckit.pingcode.sync-status              # 自动检测
/speckit.pingcode.sync-status --dry-run    # 只看流转计划
```

**方向性规则**:本地 → PingCode 单向执行。本地勾选而远端未完成 → 流转;远端已推进而本地未勾选 → 不回退,仅记入差异报告由人工确认。

## 配置

`.specify/extensions/pingcode/pingcode-config.yml` —— 推荐用 `/speckit.pingcode.init` 交互生成;手动方式复制模板 `pingcode-config.template.yml` 填写:

```yaml
project: "网运通项目组"        # 必填,项目名或标识符
product: ""                    # 可选,需求关联用;留空 = 每次运行交互选择
sprint: ""                    # 可选,留空走运行时解析链

mapping:
  spec_artifact: "特性"        # SPEC.md 整体的映射类型(需求树第 2 层,挂所选史诗下)
  story_artifact: "用户故事"   # 必须为 spec 的紧邻下一层;空=章节并入 spec 卡描述
  # task_artifact: "任务"       # 可选:配置后任务行建卡(after_tasks hook 随之生效);不配置=仅本地 checklist

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

环境变量覆盖(优先级高于配置文件):`SPECKIT_PINGCODE_PROJECT`、`SPECKIT_PINGCODE_PRODUCT`、`SPECKIT_PINGCODE_SPRINT`、`SPECKIT_PINGCODE_SPEC_ARTIFACT`、`SPECKIT_PINGCODE_STORY_ARTIFACT`、`SPECKIT_PINGCODE_PRIORITY_P1/P2/P3`、`SPECKIT_PINGCODE_STATUS_COMPLETED`。

## 任务完成标记

任务不建工作项、也不进卡描述,勾选标记只用于本地 `tasks.md` 与收尾判断:

| 标记 | 含义 |
|---|---|
| `- [x]` | 完成(所属卡全部 `[x]` 时可收尾) |
| `- [~]` | 进行中 |
| `- [ ]` | 未开始 |

收尾目标状态按项目可配置(`status_mapping.completed`);执行时名称不匹配会按 `state_type=completed` 兜底解析。

## 产物文件

| 文件 | 作用 |
|---|---|
| `specs/pingcode-mapping.json` | 统一映射登记:所有 spec 的制品生成状态、卡片 id 与卡片状态;specstoissues 写入、sync-status 消费/回写 |
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
| 史诗(中间层)列表为空 | 先在 PingCode 创建史诗后重跑,或跳过关联(spec 卡上浮为顶层) |
| 层级校验失败(story ≠ spec 下一层) | 映射跳级:运行 `/speckit.pingcode.init` 重选映射类型,保证相邻 |
| HTTP 429 | CLI 返回 `x-pc-retry-after` 响应头,按其指示等待后重试 |
| 重复卡片 | 检查 `pingcode-mapping.json`;重跑时选"补建"而非"重建" |

## 参考

[Spec Kit Jira](https://github.com/mbachorik/spec-kit-jira)

## 许可

MIT - 见 [LICENSE](LICENSE)
