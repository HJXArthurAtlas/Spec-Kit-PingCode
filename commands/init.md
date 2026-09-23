---
description: "交互式初始化 PingCode 集成:选择项目/产品/迭代/映射类型,生成 pingcode-config.yml"
---

# 初始化 PingCode 集成配置

交互式完成扩展配置并直接生成 `.specify/extensions/pingcode/pingcode-config.yml`。所有类型/状态/优先级名都从项目实际字典探查后写入,不猜测;产品与迭代可固定到配置,也可留空走运行时解析链。

## 前置条件

1. `pingcode` CLI 已安装且在 PATH 中
2. 已认证:`pingcode auth status` 显示 `authenticated: true` 且 `token_valid: true`

任何一项不满足 → 停止并给出指引(`pingcode auth login`),不要继续。

## 用户输入

$ARGUMENTS

可选参数:
- `--project <名称或ID>`:跳过项目选择,直接解析并锁定
- `--force`:已存在配置文件时不询问,直接覆盖(旧文件备份为 `.bak`)
- `--dry-run`:只输出将生成的配置内容,不写文件

## 步骤

### 1. 环境自检

```bash
command -v pingcode              # 缺失 → 提示安装 CLI,终止
pingcode auth status             # 未认证 → 引导 pingcode auth login,终止
pingcode context list
```

### 2. 选择并锁定项目

- `--project` 提供时直接使用
- 否则看 `context list` 的 `dictionaries.projects`:≥ 1 → 从缓存 `projects.values[]` 列出(名称/标识符)供用户选择;0 → 请用户直接给出项目名称或标识符

确定后执行:

```bash
pingcode context set-current-project "<项目名称或ID>"
```

从输出 `preferences.current_project_id` 记下 project_id,后续字典键都基于它。名称解析失败 → 列出缓存 `projects.values[]` 供重选。

该命令会同时把类型/状态/优先级/迭代字典写入 `.pingcode/cache.json`;确认 `.pingcode/` 已被项目 `.gitignore` 忽略,没有则追加。

### 3. 选择产品(需求关联用)

`specstoissues` 创建卡片时要关联产品下的「需求」(idea)。查询产品:

```bash
pingcode product list --compact
```

- 恰好 1 个 → 询问是否固定到配置
- 多个 → 列出(名称/标识符)让用户选择其一或跳过
- 0 个或查询失败 → 写空字符串(运行时交互选择,或跳过需求关联)

选定 → `product: "<产品名称>"`;跳过 → `product: ""`。名称/标识符与 ID 的对应以 `product list` 输出为准,不猜测。

### 4. 选择映射类型(含需求树层级校验)

从缓存 `work_item_types["<project_id>"].values[]` 列出全部类型(`id`/`name`/`group`)让用户选择:

- `spec_artifact`:推荐 `group=requirement` 中名称含「特性/feature」的类型
- `story_artifact`:推荐「用户故事/story」类;用户可选择"不按章节建卡"(写空字符串)
- `task_artifact`:可选,推荐 `group=task` 中「任务/task」类;留空 = 任务不建卡(仅本地 checklist)。
  配置后任务行建任务卡挂所属 story/spec 卡下,勾选流转受 `sync.complete_story_when_tasks_done` 控制
- 两者皆空 → 提示至少配置一项,回到本步重选

**需求树层级归类**(specstoissues 据此确定要确认的祖先层,层级:史诗=1/特性=2/用户故事=3/任务=4):

- 规范名(含「史诗/epic」「特性/feature」「用户故事/story」「任务/task」)自动归类
- 非规范名 → 询问用户该类型归属第几层,并把 `<类型名>: <层>` 写入配置 `mapping.type_levels`(运行时识别用)
- 各映射必须落在相邻层级(不跳级):`story_artifact` 非空时其层级必须 = spec 层级 + 1;
  `task_artifact` 非空时其层级必须 = story(或 spec,story 留空时)层级 + 1;
  违例(如 spec=史诗 + story=用户故事,中间跳过特性)→ 说明父子链会断裂,回到本步重选
- spec=用户故事(第 3 层)时无 requirement 下一层可用 → `story_artifact` 必须留空
  (`task_artifact` 仍可配第 4 层类型)

### 5. 选择收尾状态名

取收尾对象类型的状态集(键 `work_item_states["<project_id>::<type_id>"]`;spec-story 模式用 story 类型,spec-only 模式用 spec 类型):

- `status_mapping.completed`:优先取 `type=completed` 的第一个状态名,并列出全部可选项让用户确认

### 6. 选择优先级名

列出 `work_item_priorities` 字典的全部名称:

- `priority_mapping.p1/p2/p3`:名称含「高/紧急」「中」「低」时自动建议对应档位,否则让用户逐档指定
- 项目无多级优先级概念 → 三档全部留空

### 7. 选择迭代并初始化 CLI 运行时上下文

```bash
pingcode sprint list <project_id> --status in_progress
```

- 恰好 1 个进行中 → 询问:固定到配置,还是留空(每次运行动态解析/询问)
- 多个 → 列出(名称/编号/起止日期)让用户选择其一或留空
- 0 个 → 写空字符串(运行时再解析,或届时选择"不挂迭代")

选定 → `sprint: "<迭代名称>"`;留空 → `sprint: ""`

**CLI 运行时上下文初始化——本步总是执行:已有偏好时做校验与修复,不静默跳过**(偏好存在不代表
新鲜或完整;`workitem create` 硬性要求 context 含当前用户与当前迭代,缺失会拒绝执行):

1. 取当前用户 id:`pingcode directory me`
2. 写入偏好:
   - 迭代已定 → 一次管道喂齐三项(顺序固定:项目 → 迭代 → 用户,均可传编号或 ID),顺带保证
     sprints/users 字典完整:

```bash
printf '<项目>\n<迭代ID>\n<用户ID>\n' | pingcode context init
```

   - 迭代留空 → 项目已由第 2 步设置,只补用户:`pingcode context set-current-user <用户ID>`
3. 迭代留空时的残留校验(必做):检查 `context list` 的 `preferences.current_sprint_id`——
   - 无残留 → 回显"迭代将每次动态解析"
   - 有残留 → 对照缓存 sprints 字典查其状态:仍 `in_progress` → 回显并确认沿用;
     非进行中或查不到 → **不得静默保留**:`pingcode context set-current-sprint <最新进行中迭代ID>`
     更新,或经用户明确接受"context 残留会优先于动态解析"的风险
4. 字典校验:`context list` 中类型/状态/优先级/sprints 字典非空;为空 → 类型/状态/优先级重跑第 2 步
   `set-current-project`,sprints/users 重跑上面的管道 `context init`
5. 终态校验:`preferences` 应含 `current_project_id`、`current_user_id`(迭代已定则还有 `current_sprint_id`)

完成后回显 context 终态;`/speckit.pingcode.specstoissues` 运行时不再需要补建上下文。

### 8. 生成配置

按以下结构组装(`defaults` 两档优先级询问用户,可留空):

```yaml
project: "<选定项目>"
product: "<选定产品 或 空>"
sprint: "<选定迭代 或 空>"

mapping:
  spec_artifact: "<选定>"
  story_artifact: "<选定 或 空>"
  task_artifact: "<选定 或 空>"
  # 仅当存在非规范名类型时由 init 写入层级归类,如:
  # type_levels:
  #   模块: 2

priority_mapping:
  p1: "<...>"
  p2: "<...>"
  p3: "<...>"

defaults:
  spec:
    priority: ""
  story:
    priority: ""

status_mapping:
  completed: "<选定>"

sync:
  complete_story_when_tasks_done: true
```

已存在配置文件:逐键展示新旧差异,未提供 `--force` 时询问覆盖或中止;覆盖前把旧文件复制为 `pingcode-config.yml.bak`。

`--dry-run`:输出配置内容后结束,不写文件。

写入 `.specify/extensions/pingcode/pingcode-config.yml` 并回显:

```
✅ 配置已生成: .specify/extensions/pingcode/pingcode-config.yml
   项目: <名称>(<id>)
   产品: <名称 或 运行时选择>
   spec 卡: <spec_artifact>   story 卡: <story_artifact 或 不建>
   收尾状态: <completed 名>   迭代: <名 或 运行时解析>
   上下文: 用户 <名> ✓   当前迭代: <名 或 动态解析>   字典: ✓
```

### 9. 下一步

- 创建卡片:`/speckit.pingcode.specstoissues`(可先加 `--dry-run` 预览创建计划)
- 字典变化后重探/校准:`/speckit.pingcode.discover-context`

## 故障排除

| 症状 | 处理 |
|---|---|
| `command not found: pingcode` | 按 https://github.com/metaphor/pingcode-cli 安装 CLI |
| `authenticated: false` | 运行 `pingcode auth login`,或设置 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET` |
| 项目名解析失败 | 用 `pingcode context list` 查看缓存项目清单;确认名称/标识符后重试 |
| 产品列表为空/解析失败 | `pingcode product list --compact` 核对;企业未开通产品域时写空,运行时跳过需求关联 |
| 字典为空 | 重跑 `pingcode context set-current-project`;仍为空则检查 CLI 版本 |
| workspace context 报错 | 在 git 仓库根执行,或将 `PINGCODE_WORKSPACE_CACHE` 设为绝对路径 |
