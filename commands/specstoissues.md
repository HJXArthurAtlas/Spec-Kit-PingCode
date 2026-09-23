---
description: "将 spec-kit 的 spec 转换为 PingCode 需求树卡片(层级映射可配置,向上只补未映射祖先层)"
---

# 从 Spec 创建 PingCode 卡片层级

本命令把 spec-kit 产物转换为 PingCode 需求树卡片(通过 `pingcode` CLI,不依赖 MCP):

- **最多三级建卡**:SPEC.md 整体(`mapping.spec_artifact`)、`### User Story N` 章节(`mapping.story_artifact`)、任务行(`mapping.task_artifact`,可选)映射为工作项卡;各映射必须落在需求树**相邻层级**(不跳级),Phase 永不建卡
- **动态祖先关联**:设 spec 卡位于需求树第 n 层(史诗=1/特性=2/用户故事=3),第 1..n-1 层为未映射祖先——需求(idea)必问;中间工作项层**只从已有项中选择**,不代建;第 n 层以下永不询问
- **任务双形态**:`task_artifact` 为空(默认)时任务仅是本地 checklist,不建卡也不进卡描述;配置后每个任务行建一张任务卡挂所属 story/spec 卡下,`tasks.md` 未生成时(after_plan 触发的常态)由 `after_tasks` hook 或手动重跑补建,勾选状态由 `/speckit.pingcode.sync-status` 按 `sync` 配置流转
- **统一映射登记**:所有 spec 的映射集中登记在 `specs/pingcode-mapping.json`(制品生成状态、卡片 id、卡片状态),不再按 spec 分散存文件

所有操作通过 bash 执行 `pingcode` CLI 完成。**严格遵守:禁止猜测任何 ID;所有名称先解析为 ID 再执行写操作。**

## 前置条件

1. `pingcode` CLI 已安装且在 PATH 中(安装:https://github.com/metaphor/pingcode-cli )
2. 已完成认证:`pingcode auth status` 显示 `authenticated: true` 且 `token_valid: true`
3. 扩展配置文件存在:`.specify/extensions/pingcode/pingcode-config.yml`
4. spec 目录存在且包含 `spec.md`(`tasks.md` 非必需,本命令不读取)

任何一项不满足 → 停止并按"故障排除"一节给出指引,不要继续。

## 用户输入

$ARGUMENTS

支持的可选参数:
- `--spec <name>`:指定 spec 目录名;缺省时自动检测
- `--sprint <名称或ID>`:指定迭代;缺省时走迭代解析链
- `--product <名称/ID>`:指定需求所在产品;缺省时读配置,再缺省交互选择
- `--idea <名称/identifier/ID>`:指定关联的需求;缺省时交互选择
- `--epic <名称/identifier/ID>`:指定挂靠的已有史诗(spec 位于特性层时的唯一中间层);缺省时交互选择
- `--dry-run`:只输出将要执行的创建计划,不实际调用写接口

## 步骤

### 1. 检测 spec 目录

按优先级确定 spec 目录:

1. `--spec <name>` 参数
2. git 分支名匹配 `specs/<分支名>/`(`git branch --show-current`,去掉 `feature/`、`spec/` 等前缀后尝试)
3. 当前目录位于 `specs/<name>/` 内
4. `specs/` 下只有一个目录时直接使用

验证目录内存在 `spec.md`,否则报错列出可用 spec。输出:`📂 使用 spec: <name>`。

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

完成后用 `pingcode context list` 确认 `preferences` 含 `current_project_id`/`current_sprint_id`/`current_user_id` 三项。(若运行过 `/speckit.pingcode.init`,这三项通常已就绪,本步可跳过。)

工作区缓存 `.pingcode/cache.json` 是机器本地文件,确认它已被项目 `.gitignore` 忽略;没有则追加 `.pingcode/`。

### 3. 加载配置

读取 `.specify/extensions/pingcode/pingcode-config.yml`(不存在则提示从 `pingcode-config.template.yml` 复制并填写 `project`,终止)。

按以下优先级合并(高 → 低):环境变量 > config 文件 > 内置默认。

| 配置项 | 环境变量 | 默认 |
|---|---|---|
| `project` | `SPECKIT_PINGCODE_PROJECT` | (必填) |
| `product` | `SPECKIT_PINGCODE_PRODUCT` | `""` |
| `sprint` | `SPECKIT_PINGCODE_SPRINT` | `""` |
| `mapping.spec_artifact` | `SPECKIT_PINGCODE_SPEC_ARTIFACT` | `特性` |
| `mapping.story_artifact` | `SPECKIT_PINGCODE_STORY_ARTIFACT` | `用户故事` |
| `mapping.task_artifact` | `SPECKIT_PINGCODE_TASK_ARTIFACT` | `""` |
| `mapping.type_levels` | 无 | 内置规范表(见第 5 步) |
| `status_mapping.completed` | `SPECKIT_PINGCODE_STATUS_COMPLETED` | `已完成` |
| `priority_mapping.p1/p2/p3` | `SPECKIT_PINGCODE_PRIORITY_{P1,P2,P3}` | 高/中/低 |

确定模式:

```
spec 层级 n   = mapping.spec_artifact 在需求树中的层级(第 5 步解析)
story 卡  = story_artifact 非空:每个 `### User Story N` 章节建一张卡,挂 spec 卡下;
            其层级必须为 n+1,否则终止(不跳级)
task 卡   = task_artifact 非空且 tasks.md 存在:每个任务行建一张卡,挂所属 story/spec 卡下;
            层级必须为 n+2(story 留空时 n+1);tasks.md 不存在则本轮跳过,
            由 after_tasks hook 或手动重跑补建
spec 卡   = spec_artifact 非空:整个 spec 建一张卡
两者皆空  = 终止:没有任何映射目标
祖先层    = 第 1..n-1 层未被制品映射的层,第 8 步逐层确认;第 n 层以下永不询问
Phase     = 永不建卡
```

### 4. 解析项目与迭代

**项目**(必须):取自检步骤缓存的 `current_project_id`;若执行了 `context set-current-project`,从其 JSON 输出 `preferences.current_project_id` 获取。

**迭代**(解析链,高 → 低):

1. `--sprint` 参数(名称或 ID;名称用 `pingcode sprint list <project_id>` 按 name 匹配解析为 ID)
2. config 的 `sprint`
3. `pingcode context list` 的 `preferences.current_sprint_id`(若有)——**先校验状态**:从缓存 `sprints` 字典查该 ID 的 `status`,仅 `in_progress` 时采用;已结束或查不到 → 输出提示并跳到下一级,不得隐式挂到残留的已完成迭代
4. 动态发现:`pingcode sprint list <project_id> --status in_progress`
   - 恰好 1 个进行中 → 自动选用并向用户回显
   - 多个 → 列出(名称/编号/日期)让用户选择
   - 0 个 → 询问用户:指定迭代,或不挂迭代直接创建

解析结果必须为 sprint ID(`sprint list` 输出 `values[].id`)。用户明确选择"不挂迭代"时,创建命令省略 `--sprint`。

### 5. 解析类型与需求树层级

**层级表**:规范名内置 `史诗=1 / 特性=2 / 用户故事=3 / 任务=4`;配置 `mapping.type_levels`(init 对非规范类型名的归类结果)优先覆盖。spec 与 story 的层级:

- `mapping.spec_artifact` → spec_type_id,层级 n
- `mapping.story_artifact` → story_type_id(非空时),层级必须 = n+1
- `mapping.task_artifact` → task_type_id(非空时),层级必须 = n+2(story 留空时 n+1)
- **不跳级校验**:相邻层级不符 → 终止,引导 `/speckit.pingcode.init` 修正映射;spec=用户故事(第 3 层)时无下一层可用,story_artifact 必须为空(单卡模式,task_artifact 此时最多为第 4 层类型)

任一类型名匹配失败 → 终止,引导运行 `/speckit.pingcode.discover-context` 查看该项目实际类型名,修正配置后重试。**禁止用不存在的类型名调用创建接口。**

祖先层类型(第 8 步用):第 j 层的类型名 = `type_levels`/规范表中层级为 j 的名称;同层多个类型 → 列出让用户指认。

### 6. 解析 SPEC.md

spec 卡(spec_artifact 非空时):

- 标题:第一个 `# ` H1(无则用 spec 目录名)
- 描述:spec 正文;story_artifact 非空时剔除各 User Story 章节正文,只保留其余部分

章节卡(story_artifact 非空时),提取 User Story 章节:

- 章节:`### User Story N - ...` 起,到下一个 `###`/`##` 之前
- `us_no`:`User Story (\d+)` 取编号为 `US<N>`;无编号的 User Story 章节按出现顺序补 `US<k>`
- 优先级:从章节标题尾注捕获 `P\d+`(匹配 `(Priority: P1)` 或 `(P1)`,大小写不敏感)为 `p_no`;捕获后连同 `🎯 MVP` 从标题剥除
- 卡标题:`US<N> - <章节标题>`
- 卡描述:章节全文(含验收场景),附来源注记(见第 10 步)
- 一个章节都解析不到 → 终止,提示清空 `story_artifact` 改走单卡模式

### 7. 幂等检查

统一映射文件 `specs/pingcode-mapping.json` 中该 spec 的条目为登记单元:

- 旧版 `specs/<name>/pingcode-mapping.json` 存在且统一文件无此条目 → **自动迁移**(spec_card→artifacts.spec、stories[]→artifacts.stories、epic→ancestors.史诗),写回统一文件后删除旧文件并回显
- 条目已存在:展示登记摘要(需求/祖先/各卡 identifier 与状态),询问用户:
  1. **补建**——跳过已有 identifier 的制品,只创建缺失的(含 tasks.md 已生成但 `artifacts.tasks[]` 缺失的任务卡);条目已含 `idea`/`ancestors` 时沿用,不重选需求、不重选祖先
  2. **重建**——忽略旧登记全部重建(会在 PingCode 产生重复,需用户明确确认)
  3. **中止**——退出,让用户人工处理

默认建议补建。无条目则直接继续。

### 8. 关联祖先层(按 spec 层级动态确认)

设 spec 层级为 n,自顶向下确认第 1..n-1 层:

**前置**:第 7 步选择了"补建"且登记条目已含 `idea`/`ancestors` → 跳过本步,沿用已记录的关联。

1. **需求(idea,必问)**——记录式关联(产品域实体,不作工作项父级):

   1. 解析产品(product 解析链,高 → 低):`--product` 参数 > config 的 `product` > 交互 `pingcode product list --compact` 选择;名称精确匹配取 `values[].id`,并记录 `values[].name` 供登记
   2. 选择需求:

```bash
pingcode idea list --product <product_id> --limit 100 --compact
```

   展示 `identifier + title + 状态` 清单让用户选择,取所选项 `values[].id`,记录 `identifier`/`title`/`html_url`。**全量列出,不做归属预判**(不根据 spec 标题猜测"可能属于哪个需求");唯一允许的剔除:已完成/已关闭终态的需求不列出。`--idea` 参数按名称/identifier/id 精确匹配,命中即跳过交互。列表剔除后为空 → 告知并询问:核对 product 配置、跳过需求关联或中止。

2. **中间工作项层(j = 1..n-1,自顶向下)**:

   **史诗层(j = 1)——按是否被制品占据分派**:
   - 已被映射(`spec_artifact: "史诗"`)→ 无独立史诗动作:spec 卡本身即史诗,随第 9 步直接创建
   - 未被映射(spec 层级 ≥ 2)→ 询问用户二选一:
     a) **新建史诗**——标题默认与所选需求同名,top-level 创建(不挂父级、不挂迭代),描述内记
        `> 来源需求: <idea identifier> <idea title>` 追溯
     b) **选择已有史诗**——按下方通用规则全量列出非终态史诗供选择
   - 亦可选择跳过该层关联(spec 卡上浮为顶层);`--epic` 参数等价于 b) 的精确匹配,命中即跳过交互

   **其他中间层(j ≥ 2,如 spec=用户故事 时的特性层)——只选已有,不代建**:

```bash
pingcode workitem list --type <第 j 层类型名> --project <project_id> --limit 100
```

   - **全量列出该类型下所有非终态工作项**,不做任何预筛或归属推断:不按上一层所选项过滤 `parent_id`、不按标题相似度猜测——层级对不对由用户自己判断
   - 唯一允许的剔除:**已完成/已关闭终态项**(对照输出中的状态字段或缓存状态字典,`state_type` 为 completed/closed 的不列出);展示 `identifier + title + 状态` 清单让用户选择其一,取 `values[].id`
   - 列表剔除后为空 → 提示先在 PingCode 创建该层工作项后重跑本命令,或选择跳过该层关联
   - 选中的工作项记入 `ancestors["<层名>"]`,并作为下一层的父级参照(史诗层无论新建/选择,同样记入)

3. **spec 卡挂接父级** = 最深的已选中间层工作项;中间层被跳过或 spec=史诗(无中间层)→ 省略 `--parent`(spec 卡为顶层,需求仍按记录式关联)

`--dry-run`:在创建计划中回显 `<idea identifier> - <title>` 与各祖先层选择(或跳过),不执行创建。

### 9. 创建 spec 卡(spec_artifact 非空时)

组装描述(来源注记 + spec 正文,无任务清单):

```markdown
> 来源: specs/<name>/spec.md (spec-kit)
> 来源需求: <idea identifier> <idea title>(关联需求时)

<spec 正文;story_artifact 非空时剔除 User Story 章节正文>
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
  --parent <最深祖先工作项 id,第 8 步解析结果> \
  --description "$desc"
```

(`--priority`/`--sprint`/`--parent` 按解析结果省略可选项;无工作项父级时省略 `--parent`。spec_artifact 为空 → 跳过本步,章节卡直接挂最深祖先。)

从 JSON 输出提取 `id`、`identifier`、`html_url` 并回显:

```
✅ 已创建 spec 卡: <identifier> - <标题>
   <html_url>
```

### 10. 创建章节卡(story_artifact 非空时)

对每个 `### User Story N` 章节,挂到 spec 卡(第 9 步,若有)或最深祖先下:

```bash
pingcode workitem create \
  --title "US<N> - <章节标题>" \
  --type <story_type_id> \
  --project <project_id> \
  --sprint <sprint_id> \
  --priority "<章节优先级,解析链见下>" \
  --parent <spec_card_id 或最深祖先> \
  --description "Story from spec: <name>
US: US<N> - <章节标题>

<章节全文,含验收场景>"
```

章节卡描述为来源注记 + 章节全文,无任务清单;任务进度由 `/speckit.pingcode.sync-status` 从本地 `tasks.md` 解析。

章节卡优先级解析链(高 → 低):`priority_mapping[<p_no 小写>]`(如 P1 → p1)→ `defaults.story.priority` → 省略 `--priority`。映射名不在缓存 `work_item_priorities` 字典 → 输出提示并按下一级回落,不猜测相近名称。

story_artifact 为空 → 跳过本步,章节全文已随 spec 卡描述折叠。

### 11. 创建任务卡(task_artifact 非空且 tasks.md 存在时)

`task_artifact` 为空 → 跳过本步;`tasks.md` 不存在(after_plan 触发的常态)→ 跳过本步并在输出中注明"任务卡将在 /speckit.tasks 生成后由 after_tasks hook 或手动重跑补建"。

解析 `specs/<name>/tasks.md`:

- Phase:`## ` 标题;任务行:`- [x]/[~]/[ ] T001 描述`(保留原始标记;编号 `T\d+` 提取,无编号行按标题文本去重)
- 归属:优先任务行 `[US\d+]` 标记,其次 Phase 标题含 `User Story <N>`,都没有 → spec 级(sync-status 复用同一归属规则)

对每个任务行,挂到归属链上最深的一张**已存在**卡(story 卡 → spec 卡 → 最深祖先):

```bash
pingcode workitem create \
  --title "T001 - <描述>" \
  --type <task_type_id> \
  --project <project_id> \
  --sprint <sprint_id> \
  --parent <归属卡 id> \
  --description "> 来源: specs/<name>/tasks.md T001
归属: US1 - <章节标题>"
```

- 幂等:统一登记 `artifacts.tasks[]` 中已有同 `task_id` 且 `status: created` 的跳过
- 优先级/负责人不显式设置;创建时不流转状态(本地勾选状态由 sync-status 按 `sync` 配置处理)
- 登记条目:`{"task_id": "T001", "title": "T001 - <描述>", "status": "created", "card": {...}}`

### 12. 写统一映射登记文件

写 `specs/pingcode-mapping.json`(文件已存在时仅更新本 spec 条目,其他条目原样保留):

```json
{
  "version": 1,
  "updated_at": "<ISO8601>",
  "specs": {
    "<spec-name>": {
      "updated_at": "<ISO8601>",
      "project_id": "<id>",
      "project_name": "<名>",
      "product_id": "<id 或 null>",
      "product_name": "<名 或 null>",
      "sprint_id": "<id 或 null>",
      "sprint_name": "<名 或 null>",
      "mode": "spec-only | spec-story",
      "idea": {
        "id": "<id>",
        "identifier": "<identifier>",
        "title": "<标题>",
        "url": "<html_url>"
      },
      "ancestors": {
        "史诗": { "id": "<id>", "identifier": "<identifier>", "title": "<标题>", "url": "<html_url>" }
      },
      "artifacts": {
        "spec": {
          "title": "<标题>",
          "status": "created",
          "card": {
            "id": "<id>",
            "identifier": "<identifier>",
            "title": "<标题>",
            "type": "<spec_artifact 类型名>",
            "url": "<html_url>",
            "state": "<当前状态名>",
            "state_type": "<pending|started|completed|closed>",
            "priority": "<设置的优先级名 或 null>"
          }
        },
        "stories": [
          {
            "us_no": "US1",
            "title": "US1 - <章节标题>",
            "status": "created",
            "card": { "id": "...", "identifier": "...", "title": "...", "type": "...", "url": "...", "state": "...", "state_type": "...", "priority": "..." }
          }
        ],
        "tasks": [
          {
            "task_id": "T001",
            "title": "T001 - <描述>",
            "status": "created",
            "card": { "id": "...", "identifier": "...", "title": "...", "type": "任务", "url": "...", "state": "...", "state_type": "...", "priority": null }
          }
        ]
      },
      "errors": []
    }
  }
}
```

- `status`:`created` 或 `failed`(原因记入 `errors[]`,`card` 为 `null`);无条目 = 制品未生成
- 跳过需求关联 → `idea` 为 `null`;无中间层或跳过 → `ancestors` 为 `{}`
- 单卡模式 `stories` 为 `[]`,章节全文并入 spec 卡;`task_artifact` 未配置或 tasks.md 未生成时 `tasks` 为 `[]`
- `tasks[]` 以 `task_id` 为键;sync-status 从 `tasks.md` 解析勾选与归属,流转任务卡并回写各卡 `state`/`state_type`

### 13. 输出总结

```
═══════════════════════════════════════════
✅ PingCode 卡片创建完成(<spec-only/spec-story>)
═══════════════════════════════════════════
项目: <项目名>    迭代: <迭代名/未挂>
需求: <idea identifier> - <标题>(关联时)
史诗: <identifier> - <标题>(spec=特性 等有中间层时)
Spec 卡: <identifier> - <标题>(spec_artifact 非空时)
  <url>
Story 卡: <n> 张(spec-story 模式)
  • US1 <identifier> - <标题>
  • US2 <identifier> - <标题>
任务卡: <n> 张(task_artifact 配置且 tasks.md 已生成时)
登记: specs/pingcode-mapping.json(<spec-name> 条目)

后续:
  • /speckit.tasks 生成本地任务清单;完成后勾选并运行 /speckit.pingcode.sync-status 收尾卡片
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
| 层级校验失败(story ≠ spec 下一层) | 映射跳级:运行 `/speckit.pingcode.init` 重选映射类型,保证 spec/story 相邻 |
| 需求列表为空 | 核对 `--product`/config `product`;产品下确无需求时先在 PingCode 创建需求 |
| 史诗(中间层)列表为空 | 先在 PingCode 创建史诗后重跑,或选择跳过关联(spec 卡上浮为顶层) |
| 任务卡没建 | after_plan 触发时 tasks.md 尚未生成:运行 /speckit.tasks 后由 after_tasks hook 补建,或手动重跑本命令 |
| 状态名不识别 | 用缓存字典里的真实状态名;或按 `state_type` 兜底 |
| HTTP 429 | 等待 `x-pc-retry-after` 秒后重试 |
| 项目下无进行中迭代 | 询问用户指定迭代或不挂迭代 |
