---
description: "将 spec-kit 的 spec 和 tasks 转换为 PingCode 工作项层级"
---

# 从 Spec 和 Tasks 创建 PingCode 工作项

本命令把 spec-kit 产物转换为 PingCode 工作项层级(通过 `pingcode` CLI,不依赖 MCP):

- **SPEC.md** → 单卡模式(默认):整个 spec 一张卡(`mapping.spec_artifact`,默认用户故事);按章节模式:`### User Story N` 章节各自建卡(`mapping.story_artifact`)
- **TASKS.md 的 `## Phase N:` 标题** → 不建工作项,作为分组 checklist 嵌入所属卡描述(Phase 是实施阶段而非交付物,不作映射)
- **任务行 `- [ ] T001 ...`** → Task 工作项,`--parent` 挂到所属卡;按章节模式按任务行的 `[US#]` 标记路由到对应章节卡
- **可选需求树关联**:建卡前交互选择 Epic/Feature 并 `--parent` 挂接,呈现 史诗 → 特性 → 用户故事 层级

所有操作通过 bash 执行 `pingcode` CLI 完成。**严格遵守:禁止猜测任何 ID;所有名称先解析为 ID 再执行写操作。**

## 前置条件

1. `pingcode` CLI 已安装且在 PATH 中(安装:https://github.com/metaphor/pingcode-cli )
2. 已完成认证:`pingcode auth status` 显示 `authenticated: true` 且 `token_valid: true`
3. 扩展配置文件存在:`.specify/extensions/pingcode/pingcode-config.yml`
4. spec 目录存在且包含 `spec.md` 和 `tasks.md`

任何一项不满足 → 停止并按"故障排除"一节给出指引,不要继续。

## 用户输入

$ARGUMENTS

支持的可选参数:
- `--spec <name>`:指定 spec 目录名;缺省时自动检测
- `--sprint <名称或ID>`:指定迭代;缺省时走迭代解析链
- `--epic <名称/identifier/ID>`:指定关联的史诗;缺省时交互选择
- `--feature <名称/identifier/ID>`:指定关联的特性;缺省时在所选史诗下交互选择
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

读取 `.specify/extensions/pingcode/pingcode-config.yml`(不存在则提示从 `pingcode-config.template.yml` 复制并填写 `project`,终止)。

按以下优先级合并(高 → 低):环境变量 > config 文件 > 内置默认。

| 配置项 | 环境变量 | 默认 |
|---|---|---|
| `project` | `SPECKIT_PINGCODE_PROJECT` | (必填) |
| `sprint` | `SPECKIT_PINGCODE_SPRINT` | `""` |
| `mapping.spec_artifact` | `SPECKIT_PINGCODE_SPEC_ARTIFACT` | `用户故事` |
| `mapping.story_artifact` | `SPECKIT_PINGCODE_STORY_ARTIFACT` | `""` |
| `mapping.task_artifact` | `SPECKIT_PINGCODE_TASK_ARTIFACT` | `任务` |
| `status_mapping.*` | `SPECKIT_PINGCODE_STATUS_{COMPLETED,PENDING,IN_PROGRESS}` | 已完成/未开始/进行中 |
| `priority_mapping.p1/p2/p3` | `SPECKIT_PINGCODE_PRIORITY_{P1,P2,P3}` | 高/中/低 |

确定模式:

```
按章节模式 = story_artifact 非空:spec.md 每个 `### User Story N` 章节建一张卡,任务按 [US#] 标记路由
单卡模式   = story_artifact 为 ""(默认):整个 spec 一张卡(spec_artifact),章节仅嵌入描述
极简模式   = task_artifact 为 "":不建任务工作项,任务以 checklist 存在描述里(与上两种正交)
spec 父卡  = 仅按章节模式且 spec_artifact 非空:先建父卡挂 Feature/Epic,章节卡挂父卡下;留空则章节卡直接挂 Feature/Epic
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

- `mapping.spec_artifact` → spec_type_id(单卡模式必用;按章节模式作父卡时用)
- `mapping.story_artifact` → story_type_id(按章节模式)
- `mapping.task_artifact` → task_type_id(非极简模式)

任一名称匹配失败 → 终止,引导运行 `/speckit.pingcode.discover-context` 查看该项目实际类型名,修正配置后重试。**禁止用不存在的类型名调用创建接口。**

### 6. 解析 SPEC.md

单卡模式(以及按章节模式的 spec 父卡):

- 标题:第一个 `# ` H1(无则用 spec 目录名)
- 描述:全文 markdown

按章节模式,另提取 User Story 章节:

- 章节:`### User Story N - ...` 起,到下一个 `###`/`##` 之前
- `us_no`:`User Story (\d+)` 取编号为 `US<N>`;无编号的 User Story 章节按出现顺序补 `US<k>`
- 优先级:从章节标题尾注捕获 `P\d+`(匹配 `(Priority: P1)` 或 `(P1)`,大小写不敏感)为 `p_no`;捕获后连同 `🎯 MVP` 从标题剥除
- 卡标题:`US<N> - <章节标题>`
- 卡描述:章节全文(含验收场景),附来源注记与该 US 的任务清单(见第 10/11 步)
- 一个章节都解析不到 → 终止,提示清空 `story_artifact` 改走单卡模式

### 7. 解析 TASKS.md

提取 Phase 与任务:

- Phase:`## ` 标题(如 `## Phase 1: Setup`)
- 任务:Phase 下的列表项 `- [x] T001 描述` / `- [ ] T002 ...` / `- [~] T003 ...`
- 状态:`[x]` → completed,`[ ]` → pending,`[~]` → in_progress
- 编号:`T\d+` 正则提取;无编号的任务行也要创建,编号留空
- 章节:`[US\d+]` 标记提取为 `us_no`(如 `- [ ] T001 [US1] ...`);无标记则 `us_no` 留空(Setup/Foundational/Polish 等跨故事任务)

### 8. 幂等检查

若 `specs/<name>/pingcode-mapping.json` 已存在:展示现有映射摘要(各 story 卡 identifier + 任务状态),询问用户:

1. **补建**——跳过已有 identifier 的条目,只创建缺失的
2. **重建**——忽略旧映射全部重建(会在 PingCode 产生重复,需用户明确确认)
3. **中止**——退出,让用户人工处理

默认建议补建。不存在映射文件则直接继续。

### 9. 选择 Epic 与 Feature 关联(交互)

story 卡通过 `--parent` 挂到 Feature(或 Epic)下,PingCode 即呈现完整需求层级:史诗 → 特性 → 用户故事。映射文件记录各卡信息,关联关系在每次创建时确定。

**前置**:第 8 步选择了"补建"且各卡均已存在(有 identifier)→ 跳过本步,沿用远端现有挂接。

1. 确定类型名:默认按 `史诗` / `特性`;若缓存 `work_item_types["<project_id>"].values[]` 中无此名,列出 `group=requirement` 的实际类型让用户指认,或选择跳过关联。
2. 选择史诗:

```bash
pingcode workitem list --type <epic 类型名> --project <project_id> --limit 100
```

   展示 `identifier + title` 清单让用户选择,取所选项 `values[].id` 为 epic_id。列表为空 → 告知并询问:跳过关联或中止。
3. 选择特性:

```bash
pingcode workitem list --type <feature 类型名> --project <project_id> --limit 100
```

   在输出 `values[]` 中筛 `parent_id === <epic_id>` 的条目展示;筛完为空 → 展示项目全部特性并注明"无直接挂在所选史诗下的特性";项目无任何特性 → 询问:直接挂 Epic 或跳过。
4. 确定挂接目标:
   - 选定特性 → `parent_ref = 特性 id`(Epic 经父子链隐式关联)
   - 直接挂史诗(仅无特性可用时提供) → `parent_ref = epic_id`
   - 不关联 → 建卡时省略 `--parent`

`--epic` / `--feature` 参数:在上述 list 输出中按名称、identifier 或 id 精确匹配,命中即跳过对应交互;`--feature` 命中但其 `parent_id` 与所选史诗不一致时回显两者,让用户确认。**`--parent` 只接受 list 输出的 `id`**(CLI 不解析 identifier)。

`--dry-run`:在创建计划中回显 `关联: <epic identifier> - <标题> / <feature identifier> - <标题>`,不执行创建。

### 10. 创建 spec 卡(单卡模式;按章节模式且 spec_artifact 非空时为父卡)

先组装描述(任务清单按 Phase 分组;单卡模式嵌全部任务,按章节模式的父卡只嵌 `us_no` 为空的跨故事任务):

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
  --parent <parent_ref,第 9 步解析结果> \
  --description "$desc"
```

(`--priority`/`--sprint`/`--parent` 按解析结果省略可选项;"不关联"时省略 `--parent`。按章节模式且 `spec_artifact` 为空 → 跳过本步,章节卡直接挂 `parent_ref`。)

从 JSON 输出提取 `id`、`identifier`、`html_url` 并回显:

```
✅ 已创建卡: <identifier> - <标题>
   <html_url>
```

### 11. 创建章节卡(仅按章节模式)

对每个 `### User Story N` 章节,挂到 spec 父卡(第 10 步,若有)或 `parent_ref` 下:

```bash
pingcode workitem create \
  --title "US<N> - <章节标题>" \
  --type <story_type_id> \
  --project <project_id> \
  --sprint <sprint_id> \
  --priority "<章节优先级,解析链见下>" \
  --parent <spec_card_id 或 parent_ref> \
  --description "Story from spec: <name>
US: US<N> - <章节标题>

<章节全文,含验收场景>"
```

章节卡优先级解析链(高 → 低):`priority_mapping[<p_no 小写>]`(如 P1 → p1)→ `defaults.story.priority` → 省略 `--priority`。映射名不在缓存 `work_item_priorities` 字典 → 输出提示并按下一级回落,不猜测相近名称。

章节卡描述末尾嵌入该 US 的任务清单(`us_no` 匹配的任务,按 Phase 分组)。单卡模式跳过本步。

### 12. 创建任务工作项(极简模式跳过)

**逐个任务**执行,父级按模式取:

- 单卡模式:story 卡 id(第 10 步)
- 按章节模式:任务 `us_no` 非空 → 对应章节卡 id;为空 → spec 父卡 id,无父卡 → `parent_ref`,均无 → 省略 `--parent`

```bash
pingcode workitem create \
  --title "T001 初始化项目骨架" \
  --type <task_type_id> \
  --project <project_id> \
  --sprint <sprint_id> \
  --parent <parent_id> \
  --description "Task from spec: <name>
Phase: <phase 名>
User Story: <us_no 或 ->
Local status: pending"
```

规则:
- **必须为 TASKS.md 中每一行任务创建工作项**,不得合并、省略或只放进描述
- 单个任务创建失败:记录错误到映射(`"error": "<原因>"`),继续后续任务,最后汇总报告
- 遇到 HTTP 429:读取 `x-pc-retry-after` 等待后重试同一任务
- 每创建 10 个输出一次进度

回显格式:

```
为 Story 卡 <identifier> 创建任务:
  ├── WYT-1001 - T001 初始化项目骨架
  ├── WYT-1002 - T002 配置 lint
  └── ...
```

### 13. 写入映射文件

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
  "mode": "single | by-story",
  "minimal": false,
  "spec_card": {
    "id": "<id>",
    "identifier": "<identifier>",
    "title": "<标题>",
    "url": "<html_url>",
    "state": "<当前状态名>"
  },
  "stories": [
    {
      "us_no": "US1",
      "title": "US1 - <章节标题>",
      "id": "<id>",
      "identifier": "<identifier>",
      "url": "<html_url>",
      "state": "<当前状态名>",
      "priority": "<设置的优先级名 或 null>"
    }
  ],
  "tasks": [
    {
      "task_id": "T001",
      "title": "初始化项目骨架",
      "local_status": "pending",
      "workitem_id": "<id>",
      "identifier": "<identifier>",
      "url": "<html_url>",
      "state": "<状态名>",
      "us_no": "US1"
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

- 单卡模式:`stories` 恰一个条目且 `us_no: null`;`spec_card` 为 `null`
- 按章节模式:每个 US 章节一个 `stories` 条目;`spec_card` 仅在 `spec_artifact` 非空时非 `null`
- 极简模式:`tasks` 为 `[]`,任务仅存在于卡描述的 checklist 中

### 14. 输出总结

```
═══════════════════════════════════════════
✅ PingCode 层级创建完成(<单卡模式/按章节模式>)
═══════════════════════════════════════════
项目: <项目名>    迭代: <迭代名/未挂>
Story: <identifier> - <标题>            ← 单卡模式
  <url>
Story 卡: <n> 张                        ← 按章节模式
  • US1 <identifier> - <标题>
  • US2 <identifier> - <标题>
关联: <epic identifier> - <标题> → <feature identifier> - <标题>(未关联时省略本行)
任务: 创建 <n> 个,失败 <m> 个
映射: specs/<name>/pingcode-mapping.json

后续:
  • 本地完成任务的勾选后运行 /speckit.pingcode.sync-status 同步状态
═══════════════════════════════════════════
```

## 故障排除

| 症状 | 处理 |
|---|---|
| `command not found: pingcode` | 按 https://github.com/metaphor/pingcode-cli 安装 CLI |
| `authenticated: false` | 运行 `pingcode auth login`,或设置 `PINGCODE_CLIENT_ID`/`PINGCODE_CLIENT_SECRET` |
| workspace context 报错 | 运行 `pingcode context set-current-project "<项目名>"` |
| 仓库子目录里执行命令后上下文丢失 | CLI 已将相对缓存路径锚定到 git 仓库根(metaphor/pingcode-cli bd4977f 起);旧版 CLI 需在仓库根执行,或将 `PINGCODE_WORKSPACE_CACHE` 设为绝对路径 |
| 类型名匹配失败 | 运行 `/speckit.pingcode.discover-context` 查看实际类型名 |
| 特性与史诗挂接不符 | list 输出的特性 `parent_id` 未指向所选史诗时,展示全部特性由用户确认;挂错可用 `pingcode workitem update <id> --parent <id>` 调整 |
| 状态名不识别 | 用缓存字典里的真实状态名;或按 `state_type` 兜底 |
| HTTP 429 | 等待 `x-pc-retry-after` 秒后重试 |
| 项目下无进行中迭代 | 询问用户指定迭代或不挂迭代 |
