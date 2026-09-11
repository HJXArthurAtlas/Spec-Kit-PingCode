---
description: "将本地任务完成状态同步到 PingCode 工作项"
---

# 同步任务完成状态到 PingCode

读取本地 `tasks.md` 的勾选状态,把已建好的 PingCode 工作项流转到对应状态;任务全部完成时将 story 一并收尾。

**方向:本地 → PingCode,单向执行。** PingCode 侧的状态不回写本地文件;发现两侧不一致时只报告不擅自改动(除非本地标记为完成)。

## 前置条件

1. `pingcode` CLI 已安装且已认证
2. 工作项已通过 `/speckit.pingcode-cli.specstoissues` 创建
3. 映射文件存在:`specs/<spec-name>/pingcode-mapping.json`
4. `tasks.md` 中有完成标记

## 用户输入

$ARGUMENTS

可选参数:
- `--spec <name>`:指定要同步的 spec;缺省自动检测
- `--dry-run`:只输出将要执行的流转计划,不实际更新

## 步骤

### 1. 检测 spec 目录

按优先级:

1. `--spec <name>` 参数
2. git 分支名匹配且存在 `specs/<分支名>/pingcode-mapping.json`
3. 当前目录在 `specs/<name>/` 内
4. `specs/` 下恰有一个含 `pingcode-mapping.json` 的 spec;多个则列出让用户选择,0 个则报错提示先运行 `/speckit.pingcode-cli.specstoissues`

### 2. 环境自检与配置加载

```bash
pingcode auth status
pingcode context list
```

若字典为空,用映射文件中的 `project_name` 执行 `pingcode context set-current-project "<project_name>"` 重建字典。加载 `pingcode-config.yml` 的 `status_mapping` 与 `sync` 段(环境变量覆盖规则同 specstoissues)。

状态字典在 `.pingcode/cache.json` 的键为 `work_item_states["<project_id>::<type_id>"]` —— **按工作项类型分组**;解析某任务的目标状态时,用该任务类型的键。`pingcode workitem get <id> --compact` 返回的 `state` 与 `state_type` 是扁平字符串,直接可用。

### 3. 解析本地任务状态

解析 `specs/<name>/tasks.md`,得到每个任务的当前标记:

- `- [x]` → completed
- `- [~]` → in_progress
- `- [ ]` → pending

以 `task_id`(T 编号)为键;没有编号的行按标题文本与映射中的 `title` 匹配。

### 4. 读取映射并核对远端

读 `specs/<name>/pingcode-mapping.json`。对每个有对应工作项的任务:

```bash
pingcode workitem get <identifier> --compact
```

得到远端 `state` 与 `state_type`。

### 5. 计算流转计划

| 本地 | 远端 state_type | 动作 |
|---|---|---|
| completed | completed | 无(跳过) |
| completed | started/pending | `workitem update --state <status_mapping.completed>` |
| in_progress | started | 无(跳过) |
| in_progress | pending/completed | started→in_progress 更新;completed → **不回退**,记入"差异报告" |
| pending | pending | 无(跳过) |
| pending | started | **不回退**,记入差异报告(PingCode 侧推进以远端为准) |
| pending | completed | **不回退**,记入差异报告 |

状态名解析:优先用 `status_mapping` 的名称;若该名称不在项目状态字典中,回退为该 `state_type` 在缓存字典里的第一个状态名,并在输出中注明。

`--dry-run`:输出完整计划后结束,不执行更新。

### 6. 执行流转

逐条执行计划中的更新:

```bash
pingcode workitem update <identifier> --state "<目标状态名>"
```

- HTTP 429:按 `x-pc-retry-after` 等待重试
- 单条失败:记入 errors 继续,不中断整批

### 7. Story 收尾

统计:本地 completed 任务数 / 总任务数。

- 全部完成且 `sync.complete_story_when_tasks_done: true` → 将 story 流转到完成态:

```bash
pingcode workitem update <story_identifier> --state "<status_mapping.completed>"
```

- 未全部完成 → 不动 story,输出进度百分比

### 8. 写同步日志

写 `specs/<name>/pingcode-sync-log.json`:

```json
{
  "synced_at": "<ISO8601>",
  "spec": "<name>",
  "story_identifier": "<identifier>",
  "transitions": [
    {"task_id": "T001", "identifier": "WYT-1001", "from": "进行中", "to": "已完成"}
  ],
  "divergences": [
    {"task_id": "T003", "identifier": "WYT-1003", "local": "pending", "remote": "已完成", "note": "远端已完成,不回退"}
  ],
  "errors": [],
  "progress": {"completed": 40, "total": 42, "percent": 95}
}
```

同时更新映射文件中各任务的 `local_status` 与 `state` 字段、`updated_at`。

### 9. 输出总结

```
═══════════════════════════════════════════
✅ 状态同步完成
═══════════════════════════════════════════
Story: <identifier> - <标题>
进度: 40 / 42 (95%)

流转(<n> 条):
  • T001 WYT-1001  进行中 → 已完成
  • ...

差异(不回退,<m> 条):
  • T003 WYT-1003  本地未勾选,但 PingCode 已完成 —— 请人工确认

日志: specs/<name>/pingcode-sync-log.json
═══════════════════════════════════════════
```
