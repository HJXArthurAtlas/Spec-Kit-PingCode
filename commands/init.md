---
description: "交互式初始化 PingCode 集成:选择项目/迭代/映射类型,生成 pingcode-config.yml"
---

# 初始化 PingCode 集成配置

交互式完成扩展配置并直接生成 `.specify/extensions/pingcode/pingcode-config.yml`。所有类型/状态/优先级名都从项目实际字典探查后写入,不猜测;迭代可固定到配置,也可留空走运行时解析链。

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

### 3. 选择映射类型

从缓存 `work_item_types["<project_id>"].values[]` 列出全部类型(`id`/`name`/`group`)让用户选择:

- `spec_artifact`:推荐 `group=requirement` 中名称含「特性/feature」或「用户故事/story」的类型
- `story_artifact`:推荐「用户故事/story」类;用户可选择"不按章节建卡"(写空字符串)
- 两者皆空 → 提示至少配置一项,回到本步重选

### 4. 选择收尾状态名

取收尾对象类型的状态集(键 `work_item_states["<project_id>::<type_id>"]`;spec-story 模式用 story 类型,spec-only 模式用 spec 类型):

- `status_mapping.completed`:优先取 `type=completed` 的第一个状态名,并列出全部可选项让用户确认

### 5. 选择优先级名

列出 `work_item_priorities` 字典的全部名称:

- `priority_mapping.p1/p2/p3`:名称含「高/紧急」「中」「低」时自动建议对应档位,否则让用户逐档指定
- 项目无多级优先级概念 → 三档全部留空

### 6. 选择迭代并补全运行上下文

```bash
pingcode sprint list <project_id> --status in_progress
```

- 恰好 1 个进行中 → 询问:固定到配置,还是留空(每次运行动态解析/询问)
- 多个 → 列出(名称/编号/起止日期)让用户选择其一或留空
- 0 个 → 写空字符串(运行时再解析,或届时选择"不挂迭代")

选定 → `sprint: "<迭代名称>"`;留空 → `sprint: ""`

随后补全运行上下文——`workitem create` 硬性要求 context 含当前用户与当前迭代,缺失会拒绝执行:

1. 取当前用户 id:`pingcode directory me`
2. 迭代已定 → 一次管道喂齐三项(顺序:项目 → 迭代 → 用户,均可传编号或 ID):

```bash
printf '<项目>\n<迭代ID>\n<用户ID>\n' | pingcode context init
```

   迭代留空 → 项目已由第 2 步设置,只补用户:`pingcode context set-current-user <用户ID>`
3. 校验:`pingcode context list` 的 `preferences` 应含 `current_project_id`、`current_user_id`(迭代已定则还有 `current_sprint_id`)

完成后,`/speckit.pingcode.specstoissues` 运行时不再需要补建上下文。

### 7. 生成配置

按以下结构组装(`defaults` 两档优先级询问用户,可留空):

```yaml
project: "<选定项目>"
sprint: "<选定迭代 或 空>"

mapping:
  spec_artifact: "<选定>"
  story_artifact: "<选定 或 空>"

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
   spec 卡: <spec_artifact>   story 卡: <story_artifact 或 不建>
   收尾状态: <completed 名>   迭代: <名 或 运行时解析>
```

### 8. 下一步

- 创建卡片:`/speckit.pingcode.specstoissues`(可先加 `--dry-run` 预览创建计划)
- 字典变化后重探/校准:`/speckit.pingcode.discover-context`

## 故障排除

| 症状 | 处理 |
|---|---|
| `command not found: pingcode` | 按 https://github.com/metaphor/pingcode-cli 安装 CLI |
| `authenticated: false` | 运行 `pingcode auth login`,或设置 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET` |
| 项目名解析失败 | 用 `pingcode context list` 查看缓存项目清单;确认名称/标识符后重试 |
| 字典为空 | 重跑 `pingcode context set-current-project`;仍为空则检查 CLI 版本 |
| workspace context 报错 | 在 git 仓库根执行,或将 `PINGCODE_WORKSPACE_CACHE` 设为绝对路径 |
