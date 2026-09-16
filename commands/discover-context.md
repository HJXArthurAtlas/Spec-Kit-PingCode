---
description: "发现 PingCode 项目的类型/状态/迭代字典,生成配置片段"
---

# 发现 PingCode 项目上下文

本命令探查当前 PingCode 项目的实际字典(工作项类型、状态、优先级、迭代),并生成可直接粘贴进 `pingcode-config.yml` 的配置片段。PingCode 没有自定义字段发现的概念,对应 spec-kit-jira 的 discover-fields,这里发现的是**类型与状态字典**。

## 前置条件

1. `pingcode` CLI 已安装且已认证
2. 配置文件 `.specify/extensions/pingcode/pingcode-config.yml` 存在,且 `project` 已填写(或通过环境变量 `SPECKIT_PINGCODE_PROJECT` 提供)

## 用户输入

$ARGUMENTS

可选 `--project <名称或ID>` 临时覆盖配置中的项目。

## 步骤

### 1. 环境自检

```bash
command -v pingcode
pingcode auth status
```

### 2. 定位项目并构建字典

```bash
pingcode context set-current-project "<项目名或ID>"
```

该命令输出 `preferences.current_project_id`,并填充 `.pingcode/cache.json` 的字典。记住这个 `project_id`。

### 3. 读取字典

从 `.pingcode/cache.json` 读取(键结构不同,注意区分):

- `work_item_types["<project_id>"].values[]` → `id`、`name`、`group`(requirement/task/bug)
- `work_item_states["<project_id>::<type_id>"].values[]` → 状态**按工作项类型分组**,每种类型有独立状态集;取 `name` 与 `type`(pending/in_progress/completed/closed)。对 spec/story 两种映射类型都要读取
- `work_item_priorities` → 优先级字典
- `projects.values[]` → 项目清单(名称/标识符)

### 4. 查询迭代

```bash
pingcode sprint list <project_id> --status pending
pingcode sprint list <project_id> --status in_progress
pingcode sprint list <project_id> --status completed
```

汇总为迭代表:名称、状态、起止时间。

### 5. 展示结果

按以下格式输出:

```
📋 项目: <项目名> (<identifier>)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
工作项类型:
  • 史诗   (epic)    group=requirement
  • 特性   (feature) group=requirement
  • 用户故事 (story)  group=requirement
  • 任务   (task)    group=task
  • 缺陷   (bug)     group=bug

状态(按 state_type 分组):
  pending:    未开始、待评审
  started:    进行中、开发中
  completed:  已完成、已发布

迭代:
  • Sprint 21  in_progress  2026-08-26 ~ 2026-09-08
  • Sprint 22  pending      2026-09-09 ~ 2026-09-22
```

### 6. 生成配置片段

基于发现的真实名称,输出推荐配置(状态取每个 state_type 的第一个,并提示可替换):

```yaml
# 建议粘贴到 .specify/extensions/pingcode/pingcode-config.yml
project: "<项目名>"

mapping:
  spec_artifact: "<特性 或 用户故事,取实际存在的最接近类型>"
  story_artifact: "<用户故事类型名;章节不单独建卡则留空>"

status_mapping:
  completed: "<state_type=completed 的实际状态名>"

priority_mapping:
  p1: "<实际优先级名,如 高/紧急>"
  p2: "<如 中>"
  p3: "<如 低>"
```

`spec_artifact` 优先推荐 `特性`(章节卡挂其下,构成完整需求树);项目无特性类型时推荐 `用户故事`。优先级名取 `work_item_priorities` 字典实际值;项目无多级优先级概念时全部留空。

### 7. 保存探查结果

写 `.specify/extensions/pingcode/discovered-context.json`:

```json
{
  "discovered_at": "<ISO8601>",
  "project_id": "<id>",
  "project_name": "<名>",
  "identifier": "<项目标识>",
  "types": [{"id": "story", "name": "用户故事", "group": "requirement"}],
  "states": [{"name": "进行中", "type": "started"}],
  "priorities": ["低", "中", "高"],
  "sprints": [{"id": "...", "name": "Sprint 21", "status": "in_progress"}]
}
```

结尾提示:把片段合入 `pingcode-config.yml`(或改用 `/speckit.pingcode.init` 交互生成)后,即可运行 `/speckit.pingcode.specstoissues`。
