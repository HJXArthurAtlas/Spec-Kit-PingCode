# Changelog

本项目的所有显著变更将记录在本文件。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/),
版本遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [2.4.0] - 2026-09-23

### Changed

- **BREAKING**:映射层级动态化——`spec_artifact`/`story_artifact` 决定制品栈在需求树
  (需求 → 史诗 → 特性 → 用户故事 → 任务)的位置,必须相邻不跳级(init 归类校验,
  非规范类型名询问后写入 `mapping.type_levels`);spec 卡上方未映射祖先层自顶向下确认:
  需求(idea)必问,中间工作项层(如史诗)**只从已有项中选择,不再自动创建同名 Epic**;
  spec 下方的层永不询问;新增 `--epic` 参数跳过史诗层交互
- **BREAKING**:映射登记从 `specs/<name>/pingcode-mapping.json` 收敛为统一文件
  `specs/pingcode-mapping.json`,记录所有 spec 的制品生成状态(status: created/failed)、
  卡片 id 与卡片状态;旧文件由 specstoissues 自动迁移后删除
- `init` 第 7 步改造为「选择迭代并初始化 CLI 运行时上下文」:上下文初始化/校验**总是执行**,
  已有三项偏好时不再静默跳过——校验 用户/迭代 偏好与四类字典(类型/状态/优先级/sprints)的
  新鲜度与完整性,按需修复;迭代留空时强制校验 `current_sprint_id` 残留状态;
  输出摘要新增上下文终态行

### Fixed

- `specstoissues` 迭代解析链第 3 级(context 当前迭代)增加状态校验:仅 `in_progress` 时采用,
  context 残留的已结束迭代不再隐式生效,继续走动态发现
- `init` 迭代留空时检查 context 已残留的 `current_sprint_id`,非进行中则提醒更新,
  避免陈旧迭代在解析链中优先于动态解析

## [2.3.0] - 2026-09-22

### Changed

- **BREAKING**:默认 hook 从 `after_tasks` 移到 `after_plan`——`/speckit.plan` 完成即提示
  创建 PingCode 卡片(此时 `tasks.md` 尚未生成,`specstoissues` 不再要求也不读取 `tasks.md`)
- **BREAKING**:任务清单不再折叠进卡描述——任务与 Phase 永不进卡片,进度只看本地
  `tasks.md`,由 `sync-status` 据此收尾
- **BREAKING**:映射默认值改为 `spec_artifact: "特性"` + `story_artifact: "用户故事"`,
  spec-story 双层成为默认模式
- **BREAKING**:父卡关联重构为需求关联——运行时先解析产品(`--product` > config `product`
  > 交互),选择产品下的「需求」(idea)后自动查找/创建同名 Epic(按标题精确匹配复用,
  描述内记 `> 来源需求:` 追溯),spec 卡挂 Epic 下、章节卡挂 spec 卡下;
  新增 `--idea`/`--product` 参数,移除 `--epic`/`--feature`

### Added

- 配置新增 `product` 字段与环境变量 `SPECKIT_PINGCODE_PRODUCT`;`init` 命令新增产品选择步骤
- `pingcode-mapping.json` 新增 `product_id`、`idea`、`epic` 字段
- `discover-context` 探查结果与配置片段补充产品清单

## [2.2.1] - 2026-09-16

### Changed

- `init` 命令第 6 步扩展为「选迭代并补全运行上下文」:迭代选定后用一次
  `printf ... | pingcode context init` 管道喂齐 项目/迭代/用户 三项偏好
  (迭代留空时仅 `set-current-user`),并以 `context list` 校验——完成后
  `specstoissues` 运行时不再缺 `workitem create` 的上下文前置
- `specstoissues` 上下文自检注明:跑过 `init` 后该兜底通常可跳过

## [2.2.0] - 2026-09-16

### Added

- `/speckit.pingcode.init` 交互式初始化命令:认证自检、选择项目与迭代(可留空走运行时
  解析链)、基于项目实际字典选择映射类型/收尾状态/优先级名,直接生成
  `pingcode-config.yml`;支持 `--project`/`--force`/`--dry-run`,覆盖前备份为 `.bak`
- `discover-context` 保留为字典重探与配置片段工具,README 快速开始改为 init 优先

## [2.1.0] - 2026-09-16

### Changed

- **BREAKING**:映射收敛为 spec 与 User Story 两级——`mapping.task_artifact` 移除,
  任务与 Phase 不再创建工作项,一律以 checklist 文本折叠进所属卡描述:
  story 卡嵌自己的任务清单(按 Phase 分组),spec 卡嵌跨故事任务清单;
  环境变量 `SPECKIT_PINGCODE_TASK_ARTIFACT` 移除
- 卡片描述构成:配置 `story_artifact` 时 spec 卡只保留非 User Story 正文,
  章节内容各自进章节卡;未配置时全文(含章节)并入 spec 卡
- 父卡关联按顶层卡类型泛化:故事类卡选特性挂靠,特性类卡选史诗挂靠(均可跳过);
  章节卡固定挂 spec 卡下
- `status_mapping` 精简为仅 `completed`(中间状态不由扩展流转);
  环境变量 `SPECKIT_PINGCODE_STATUS_PENDING/IN_PROGRESS` 移除
- `sync-status` 精简为卡片收尾:某张卡关联任务全部 `[x]` → 流转该卡到完成态
  (spec-story 模式逐 story 卡判断,spec-only 模式判断 spec 卡);
  逐任务流转与差异报告移除,日志改为逐卡收尾记录
- `pingcode-mapping.json` 移除 `tasks[]`/`minimal`/`summary`,`mode` 取值改为
  `spec-only | spec-story`;任务归属由 sync-status 直接从 `tasks.md` 解析

### Removed

- 任务工作项创建流程与极简模式概念(任务永远只是卡内 checklist)

## [2.0.0] - 2026-09-16

### Changed

- **BREAKING**:`mapping.phase_artifact` 移除,代之以 `mapping.story_artifact` —— 映射轴从
  TASKS.md 的 Phase 标题改为 SPEC.md 的 `### User Story N` 章节。Phase 是实施阶段而非交付物,
  不再可映射为工作项,始终以分组 checklist 嵌入所属卡描述。
  环境变量 `SPECKIT_PINGCODE_PHASE_ARTIFACT` 更名为 `SPECKIT_PINGCODE_STORY_ARTIFACT`
- `story_artifact` 留空(默认)= 单卡模式,行为与旧版 2 层模式一致:整个 spec 一张
  `spec_artifact` 卡;设为类型名 = 按章节模式,每个 User Story 章节建一张卡,
  `spec_artifact` 可选作章节卡的父卡(建议留空或设更高需求类型如 "特性")
- `pingcode-mapping.json` 结构调整:`story` 单对象 + `phases[]` 嵌套任务 → 顶层 `stories[]`
  (单卡模式为单个 `us_no: null` 条目)+ `spec_card` + 顶层 `tasks[]`(新增 `us_no` 字段);
  `mode` 取值改为 `single | by-story`,极简模式改由独立的 `minimal` 布尔字段表达
- `sync-status` 收尾逻辑改为逐卡:每张 story 卡在其关联任务(`us_no` 匹配)全部完成后流转;
  按章节模式的 spec 父卡不自动收尾

### Added

- 按章节模式:任务行按 `[US#]` 标记路由到所属章节卡;无标记的跨故事任务
  (Setup/Foundational/Polish 等)挂 spec 父卡,无父卡时挂所选特性/史诗
- 按章节模式章节卡携带优先级:从章节标题尾注捕获 `P1/P2/P3`,经新增 `priority_mapping`
  (默认 高/中/低,环境变量 `SPECKIT_PINGCODE_PRIORITY_P1/P2/P3`)映射为项目优先级名;
  无编号章节回落 `defaults.story.priority`,仍未设置则不设优先级。映射文件 `stories[]`
  条目新增 `priority` 字段记录实际设置值

## [1.1.0] - 2026-09-14

### Added

- `specstoissues` 支持 story 关联 Epic/Feature:创建 story 前交互选择史诗与特性,
  以 `--parent` 挂到特性下(呈现 史诗 → 特性 → 用户故事 完整层级);无特性时可直接挂史诗,
  也可选择不关联。新增 `--epic`/`--feature` 参数(按名称/identifier/id 精确匹配)跳过交互;
  补建模式下 story 已存在时跳过选择;映射文件结构不变。

## [1.0.1] - 2026-09-14

### Changed

- `specstoissues` 故障排除表新增「仓库子目录里执行命令后上下文丢失」条目:
  pingcode CLI 自 bd4977f 起将相对缓存路径锚定到 git 仓库根,旧版 CLI 需在仓库根执行
  或将 `PINGCODE_WORKSPACE_CACHE` 设为绝对路径。

## [1.0.0] - 2026-09-10

### Added

- `extension.yml` 扩展清单(扩展 id `pingcode`,命令注册、配置模板、after_tasks 钩子)
- `/speckit.pingcode.specstoissues`:SPEC.md + TASKS.md → PingCode 工作项层级
  - 默认 2 层模式(story + task,Phase 嵌入描述);支持 3 层与极简模式
  - 迭代运行时解析链(参数 > 配置 > 上下文 > 自动发现/询问)
  - 幂等防护(补建/重建/中止)、429 限流重试、逐任务错误隔离
- `/speckit.pingcode.discover-context`:项目类型/状态/迭代字典探查与配置片段生成
- `/speckit.pingcode.sync-status`:本地勾选状态 → PingCode 状态流转,story 自动收尾,
  远端先行变更不回退(差异报告)
- `pingcode-config.template.yml` 配置模板与环境变量覆盖
- 映射产物 `pingcode-mapping.json` 与同步日志 `pingcode-sync-log.json`
