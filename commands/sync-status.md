---
description: "本地任务全部勾选后,将对应的 PingCode 卡片流转到完成态"
---

# 同步任务完成状态到 PingCode

任务本身默认不是工作项(仅本地 checklist);配置 `mapping.task_artifact` 后每个任务行有一张任务卡。本命令做两件事,均受 `sync.complete_story_when_tasks_done` 开关控制:

1. **任务卡流转**(前置:`mapping.task_artifact` 已配置;未配置时本项整体跳过,任务仅是本地 checklist):本地任务行 `- [x]` 且登记中对应任务卡未完成 → 把该任务卡流转到 `status_mapping.completed`
2. **卡片收尾**:某张 story/spec 卡关联的任务全部 `- [x]` → 把该卡流转到完成态(方向:本地 → PingCode,单向、只收尾、不回退)

**方向:本地 → PingCode,单向、只收尾。** 不做中间状态流转,不回退远端状态,不改本地文件;远端已推进的卡不动。

## 前置条件

1. `pingcode` CLI 已安装且已认证
2. 卡片已通过 `/speckit.pingcode.specstoissues` 创建
3. 统一映射登记存在且含该 spec 条目:`specs/pingcode-mapping.json`(旧版按 spec 分散的 `specs/<name>/pingcode-mapping.json` 由 specstoissues 运行时自动迁移)
4. `tasks.md` 中有完成标记

## 用户输入

$ARGUMENTS

可选参数:
- `--spec <name>`:指定要同步的 spec;缺省自动检测
- `--dry-run`:只输出收尾计划,不实际更新

## 步骤

### 1. 检测 spec 目录

按优先级:

1. `--spec <name>` 参数
2. git 分支名匹配且统一登记含该 spec 条目
3. 当前目录在 `specs/<name>/` 内
4. 统一登记 `specs/pingcode-mapping.json` 的 `specs` 下恰有一个条目 → 直接使用;多个条目则列出让用户选择,0 个则报错提示先运行 `/speckit.pingcode.specstoissues`

### 2. 环境自检

```bash
pingcode auth status
pingcode context list
```

若字典为空,用映射文件中的 `project_name` 执行 `pingcode context set-current-project "<project_name>"` 重建。加载 `pingcode-config.yml` 的 `status_mapping.completed` 与 `sync` 段(环境变量覆盖规则同 specstoissues)。

`pingcode workitem get <identifier> --compact` 返回的 `state` 与 `state_type` 是扁平字符串,直接可用;`status_mapping.completed` 不在缓存状态字典时,按 `state_type=completed`(`work_item_states["<project_id>::<type_id>"]`)取第一个状态名兜底,并输出注明。

### 3. 解析本地任务状态

解析 `specs/<name>/tasks.md`:

- `- [x]` → completed;`- [~]` / `- [ ]` → 未完成
- 以 `task_id`(T 编号)为键;没有编号的行按标题文本匹配

### 4. 计算各卡完成度

任务的卡归属:优先任务行 `[US#]` 标记,其次 Phase 标题含 `User Story <N>`,都没有 → spec 级。

| 模式 | 卡 | 关联任务 |
|---|---|---|
| spec-story | 每张 story 卡 | `us_no` 匹配的任务 |
| spec-story | spec 卡 | 不自动收尾 |
| spec-only | spec 卡 | 全部任务 |

逐卡核对该卡当前状态(`mode`、`artifacts.spec`、`artifacts.stories` 读自统一登记条目):

```bash
pingcode workitem get <identifier> --compact
```

已是 completed 的卡跳过。

### 5. 流转计划

**任务卡**(配置 `task_artifact` 时):登记 `artifacts.tasks[]` 中每个 `task_id`,在本地 `tasks.md` 中勾选为 `- [x]` 且其卡片 `state_type` 非 completed → 计划流转:

```bash
pingcode workitem update <task 卡 identifier> --state "<status_mapping.completed>"
```

本地未勾选而远端任务卡已完成 → 不回退,仅输出差异。

**story/spec 卡收尾**:某卡关联任务全部 completed 且 `sync.complete_story_when_tasks_done: true` → 计划流转该卡:

```bash
pingcode workitem update <identifier> --state "<status_mapping.completed>"
```

未全部完成 → 不动该卡,输出进度。远端已 completed 而本地未全勾 → 不回退,输出提示。

`--dry-run`:输出完整计划后结束,不执行更新。

### 6. 执行流转

逐条执行计划中的更新:

- HTTP 429:按 `x-pc-retry-after` 等待后重试
- 单条失败:记入 errors 继续,不中断整批

### 7. 写同步日志

写 `specs/<name>/pingcode-sync-log.json`:

```json
{
  "synced_at": "<ISO8601>",
  "spec": "<name>",
  "cards": [
    {"us_no": "US1", "identifier": "<identifier>", "closed": true},
    {"us_no": null, "identifier": "<identifier>", "closed": false, "note": "spec 卡不自动收尾"}
  ],
  "errors": [],
  "progress": {"completed": 40, "total": 42, "percent": 95}
}
```

同时回写统一登记 `specs/pingcode-mapping.json` 中该 spec 条目里各卡的 `state`/`state_type` 与条目 `updated_at`。

### 8. 输出总结

```
═══════════════════════════════════════════
✅ 状态同步完成
═══════════════════════════════════════════
卡片(逐卡):
  • US1 <identifier> - <标题>   已收尾
  • US2 <identifier> - <标题>   3 / 5,未收尾
进度: 40 / 42 (95%)

日志: specs/<name>/pingcode-sync-log.json
═══════════════════════════════════════════
```
