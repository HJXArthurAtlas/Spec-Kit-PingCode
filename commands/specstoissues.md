---
description: "将 spec-kit 的 spec 和 tasks 转换为 PingCode 工作项层级"
---

# 从 Spec 和 Tasks 创建 PingCode 工作项

本命令把 spec-kit 产物转换为 PingCode 工作项层级(通过 `pingcode` CLI,不依赖 MCP):

- **SPEC.md** → Story(用户故事,或 config 指定的类型)
- **TASKS.md 的 `## Phase N:` 标题** → 默认不建工作项,作为分组 checklist 嵌入 story 描述(2 层模式);config 启用时建独立工作项(3 层模式)
- **任务行 `- [ ] T001 ...`** → Task 工作项,`--parent` 挂到 story

所有操作通过 bash 执行 `pingcode` CLI 完成。**严格遵守:禁止猜测任何 ID;所有名称先解析为 ID 再执行写操作。**

## 前置条件

1. `pingcode` CLI 已安装且在 PATH 中(安装:https://github.com/metaphor/pingcode-cli )
2. 已完成认证:`pingcode auth status` 显示 `authenticated: true` 且 `token_valid: true`
3. 扩展配置文件存在:`.specify/extensions/pingcode-cli/pingcode-config.yml`
4. spec 目录存在且包含 `spec.md` 和 `tasks.md`

任何一项不满足 → 停止并按"故障排除"一节给出指引,不要继续。

## 用户输入

$ARGUMENTS

支持的可选参数:
- `--spec <name>`:指定 spec 目录名;缺省时自动检测
- `--sprint <名称或ID>`:指定迭代;缺省时走迭代解析链
- `--dry-run`:只输出将要执行的创建计划,不实际调用写接口

## 步骤

### 1. 检测 spec 目录

按优先级确定 spec 目录:

1. `--spec <name>` 参数
2. git 分支名匹配 `specs/<分支名>/`(`git branch --show-current`,去掉 `feature/`、`spec/` 等前缀后尝试)
3. 当前目录位于 `specs/<name>/` 内
4. `specs/` 下只有一个目录时直接使用

验证目录内同时存在 `spec.md` 与 `tasks.md`,否则报错列出可用 spec。输出:`📂 使用 spec: <name>`。

### 2. 环境自检

依次执行:

```bash
command -v pingcode              # 缺失 → 提示安装 CLI,终止
pingcode auth status             # authenticated/token_valid 不为 true → 引导 pingcode auth login,终止
pingcode context list            # 查看字典与偏好
```

若 `context list` 中 `dictionaries.projects` 为 0,或 `preferences.current_project_name` 与配置的项目不符,执行:

```bash
pingcode context set-current-project "<配置的项目名>"
```

该命令会解析名称 → ID,并把项目/类型/状态/优先级字典写入 `.pingcode/cache.json`。若项目名无法解析,从缓存 `projects.values[]` 列出全部可用项目供用户选择,更新配置后继续。

**创建接口还要求上下文包含当前用户与当前迭代**,缺一不可(实测 `workitem create` 会拒绝执行):

1. 当前用户:`context set-current-user @me` 不可用(@me 仅展开已缓存 ID)。用 `pingcode directory me` 取当前用户 `id`,再执行 `pingcode context set-current-user <id>`。
2. 当前迭代:先完成第 4 步的迭代解析,拿到 sprint ID 后执行 `pingcode context set-current-sprint <sprint_id>`。
   ⚠️ 只能传 ID:迭代字典未填充时按名称设置会把原始名称当作 ID 存入(实测踩坑)。

**迭代字典填充**:`--sprint` 参数会对照缓存的迭代字典校验,报 `No cached sprint matched` 时说明字典为空。用管道驱动一次交互式初始化填充(顺序:项目 → 迭代 → 用户,均可传编号或 ID):

```bash
printf '<项目编号>\n<迭代编号>\n<用户ID>\n' | pingcode context init
```

完成后用 `pingcode context list` 确认 `preferences` 含 `current_project_id`/`current_sprint_id`/`current_user_id` 三项。

工作区缓存 `.pingcode/cache.json` 是机器本地文件,确认它已被项目 `.gitignore` 忽略;没有则追加 `.pingcode/`。

### 3. 加载配置

读取 `.specify/extensions/pingcode-cli/pingcode-config.yml`(不存在则提示从 `pingcode-config.template.yml` 复制并填写 `project`,终止)。

按以下优先级合并(高 → 低):环境变量 > config 文件 > 内置默认。

| 配置项 | 环境变量 | 默认 |
|---|---|---|
| `project` | `SPECKIT_PINGCODE_PROJECT` | (必填) |
| `sprint` | `SPECKIT_PINGCODE_SPRINT` | `""` |
| `mapping.spec_artifact` | `SPECKIT_PINGCODE_SPEC_ARTIFACT` | `用户故事` |
| `mapping.phase_artifact` | `SPECKIT_PINGCODE_PHASE_ARTIFACT` | `""` |
| `mapping.task_artifact` | `SPECKIT_PINGCODE_TASK_ARTIFACT` | `任务` |
| `status_mapping.*` | `SPECKIT_PINGCODE_STATUS_{COMPLETED,PENDING,IN_PROGRESS}` | 已完成/未开始/进行中 |

确定模式:

```
3 层模式 = phase_artifact 非空
2 层模式 = phase_artifact 为 ""(默认):Phase 嵌入 story 描述
极简模式 = task_artifact 为 "":只建 story,任务以 checklist 存在描述里
```

### 4. 解析项目与迭代

**项目**(必须):取自检步骤缓存的 `current_project_id`;若执行了 `context set-current-project`,从其 JSON 输出 `preferences.current_project_id` 获取。

**迭代**(解析链,高 → 低):

1. `--sprint` 参数(名称或 ID;名称用 `pingcode sprint list <project_id>` 按 name 匹配解析为 ID)
2. config 的 `sprint`
3. `pingcode context list` 的 `preferences.current_sprint_id`(若有)
4. 动态发现:`pingcode sprint list <project_id> --status in_progress`
   - 恰好 1 个进行中 → 自动选用并向用户回显
   - 多个 → 列出(名称/编号/日期)让用户选择
   - 0 个 → 询问用户:指定迭代,或不挂迭代直接创建

解析结果必须为 sprint ID(`sprint list` 输出 `values[].id`)。用户明确选择"不挂迭代"时,创建命令省略 `--sprint`。

### 5. 解析类型名 → 类型 ID

从 `.pingcode/cache.json` 的 `work_item_types["<project_id>"].values[]` 按名称精确匹配:

- `mapping.spec_artifact` → spec_type_id
- `mapping.phase_artifact` → phase_type_id(3 层模式)
- `mapping.task_artifact` → task_type_id(非极简模式)

任一名称匹配失败 → 终止,引导运行 `/speckit.pingcode-cli.discover-context` 查看该项目实际类型名,修正配置后重试。**禁止用不存在的类型名调用创建接口。**

### 6. 解析 SPEC.md

- 标题:第一个 `# ` H1(无则用 spec 目录名)
- 描述:全文 markdown

### 7. 解析 TASKS.md

提取 Phase 与任务:

- Phase:`## ` 标题(如 `## Phase 1: Setup`)
- 任务:Phase 下的列表项 `- [x] T001 描述` / `- [ ] T002 ...` / `- [~] T003 ...`
- 状态:`[x]` → completed,`[ ]` → pending,`[~]` → in_progress
- 编号:`T\d+` 正则提取;无编号的任务行也要创建,编号留空

### 8. 幂等检查

若 `specs/<name>/pingcode-mapping.json` 已存在:展示现有映射摘要(story identifier + 各任务状态),询问用户:

1. **补建**——跳过已有 identifier 的条目,只创建缺失的
2. **重建**——忽略旧映射全部重建(会在 PingCode 产生重复,需用户明确确认)
3. **中止**——退出,让用户人工处理

默认建议补建。不存在映射文件则直接继续。

### 9. 创建 Story

先组装描述(2 层模式在此嵌入任务清单,3 层模式只列 phase 清单):

```markdown
> 来源: specs/<name>/spec.md (spec-kit)

<SPEC.md 全文>

## 任务清单

### Phase 1: Setup
- [ ] T001 ...
- [ ] T002 ...

### Phase 2: Foundational
- [ ] T010 ...
```

长文本通过变量传递,避免 shell 转义问题:

```bash
desc=$(cat <<'DESC_EOF'
<上面组装的完整描述>
DESC_EOF
)
pingcode workitem create \
  --title "<spec 标题>" \
  --type <spec_type_id 或名称> \
  --project <project_id> \
  --sprint <sprint_id> \
  --priority "<config.defaults.spec.priority>" \
  --description "$desc"
```

(`--priority`/`--sprint` 按解析结果省略可选项。)

从 JSON 输出提取 `id`、`identifier`、`html_url` 并回显:

```
✅ 已创建 Story: <identifier> - <标题>
   <html_url>
```

### 10. 创建 Phase 工作项(仅 3 层模式)

对每个 Phase:

```bash
pingcode workitem create \
  --title "<phase 标题>" \
  --type <phase_type_id> \
  --project <project_id> \
  --sprint <sprint_id> \
  --parent <story_id> \
  --description "Phase from spec: <name>"
```

2 层模式跳过本步。

### 11. 创建任务工作项(极简模式跳过)

**逐个任务**执行,父级按模式取 story_id(2 层)或 phase_id(3 层):

```bash
pingcode workitem create \
  --title "T001 初始化项目骨架" \
  --type <task_type_id> \
  --project <project_id> \
  --sprint <sprint_id> \
  --parent <parent_id> \
  --description "Task from spec: <name>
Phase: <phase 名>
Local status: pending"
```

规则:
- **必须为 TASKS.md 中每一行任务创建工作项**,不得合并、省略或只放进描述
- 单个任务创建失败:记录错误到映射(`"error": "<原因>"`),继续后续任务,最后汇总报告
- 遇到 HTTP 429:读取 `x-pc-retry-after` 等待后重试同一任务
- 每创建 10 个输出一次进度

回显格式:

```
为 Story <identifier> 创建任务(2 层模式):
  ├── WYT-1001 - T001 初始化项目骨架
  ├── WYT-1002 - T002 配置 lint
  └── ...
```

### 12. 写入映射文件

写 `specs/<name>/pingcode-mapping.json`:

```json
{
  "created_at": "<ISO8601>",
  "updated_at": "<ISO8601>",
  "spec": "<name>",
  "project_id": "<id>",
  "project_name": "<名>",
  "sprint_id": "<id 或 null>",
  "sprint_name": "<名 或 null>",
  "mode": "2-level | 3-level | minimal",
  "story": {
    "id": "<id>",
    "identifier": "<identifier>",
    "title": "<标题>",
    "url": "<html_url>",
    "state": "<当前状态名>"
  },
  "phases": [
    {
      "name": "Phase 1: Setup",
      "workitem_id": "<3 层模式才有>",
      "tasks": [
        {
          "task_id": "T001",
          "title": "初始化项目骨架",
          "local_status": "pending",
          "workitem_id": "<id>",
          "identifier": "<identifier>",
          "url": "<html_url>",
          "state": "<状态名>"
        }
      ]
    }
  ],
  "errors": [],
  "summary": {
    "total_tasks": 42,
    "created": 42,
    "failed": 0
  }
}
```

### 13. 输出总结

```
═══════════════════════════════════════════
✅ PingCode 层级创建完成(<模式>)
═══════════════════════════════════════════
项目: <项目名>    迭代: <迭代名/未挂>
Story: <identifier> - <标题>
  <url>
任务: 创建 <n> 个,失败 <m> 个
映射: specs/<name>/pingcode-mapping.json

后续:
  • 本地完成任务的勾选后运行 /speckit.pingcode-cli.sync-status 同步状态
═══════════════════════════════════════════
```

## 故障排除

| 症状 | 处理 |
|---|---|
| `command not found: pingcode` | 按 https://github.com/metaphor/pingcode-cli 安装 CLI |
| `authenticated: false` | 运行 `pingcode auth login`,或设置 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET` |
| workspace context 报错 | 运行 `pingcode context set-current-project "<项目名>"` |
| 类型名匹配失败 | 运行 `/speckit.pingcode-cli.discover-context` 查看实际类型名 |
| 状态名不识别 | 用缓存字典里的真实状态名;或按 `state_type` 兜底 |
| HTTP 429 | 等待 `x-pc-retry-after` 秒后重试 |
| 项目下无进行中迭代 | 询问用户指定迭代或不挂迭代 |
