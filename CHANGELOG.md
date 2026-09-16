# Changelog

本项目的所有显著变更将记录在本文件。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/),
版本遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

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
