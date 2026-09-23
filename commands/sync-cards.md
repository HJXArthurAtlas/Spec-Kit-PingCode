---
description: "实现前核对 spec 制品与 PingCode 卡片的一致性,按差异增删改卡片"
---

# 核对 spec 制品与 PingCode 卡片一致

在 `/speckit.implement` 之前运行(before_implement hook 触发,也可手动):after_plan 建卡之后、实现开始之前,spec 制品可能又被修改过——本命令把差异同步到 PingCode,保证卡片与制品一致。

**只关注制品内容与卡片的一致**;祖先关联(需求/史诗链)保持不变,除非制品有增删才为新卡解析父级。

## 前置条件

1. `pingcode` CLI 已安装且已认证(`pingcode auth status`)
2. 统一登记存在且含该 spec 条目:`specs/pingcode-mapping.json`(无 → 提示先运行 `/speckit.pingcode.specstoissues`)
3. `spec.md` 存在(`tasks.md` 仅在 `task_artifact` 已配置时需要)

## 用户输入

$ARGUMENTS

- `--spec <name>`:指定 spec;缺省时自动检测(规则同 specstoissues)
- `--dry-run`:只输出差异计划,不执行任何写操作(**建议每次先跑**)

## 步骤

### 1. 加载上下文

- 读取统一登记 `specs/pingcode-mapping.json` 中该 spec 的条目(`mode`、`idea`、`ancestors`、`artifacts`)
- 读取配置 `pingcode-config.yml`(映射/状态/优先级)
- 环境自检:`pingcode auth status`;`context list` 偏好缺失时按 specstoissues 第 2 步补齐

### 2. 解析当前制品(期望状态)

- **spec 卡**:`# ` H1 标题 + 描述(规则同 specstoissues:story_artifact 非空时剔除 User Story 章节正文)
- **story 卡**:`### User Story N` 章节 → `us_no`/标题(`US<N> - <章节标题>`)/优先级/章节全文
- **任务卡**(仅 `task_artifact` 已配置且 `tasks.md` 存在):任务行 → `task_id`(T 编号)/标题/归属(story 卡 → spec 卡)

### 3. 逐卡取远端现状

```bash
pingcode workitem get <identifier> --compact
```

记录每张卡的当前 `title` 与 `description`。单卡获取失败 → 记入 `errors[]`,该卡本轮跳过。

### 4. 计算差异计划

对比"当前制品期望"与"远端现状 + 登记条目",产出三类动作:

| 动作 | 判定 | 处理 |
|---|---|---|
| **删除** | 登记有卡,但 spec/tasks.md 中制品已不存在 | `pingcode workitem delete <identifier>` |
| **更新** | 制品仍在,标题或描述与现算内容不一致 | `pingcode workitem update <identifier> --title ... --description ...`(字段级,只改有差异的字段) |
| **新建** | 制品存在但登记无卡或 `status: failed` | 按 specstoissues 第 9/10/11 步规则创建并挂接 |

- 描述对比 = 现"期望描述全文"与远端 `description` 不一致即视为有差异(描述由制品确定性生成)
- **祖先关联不重新询问**:沿用登记中的 `idea`/`ancestors`;新建卡的父级按归属解析(story 卡 → spec 卡 → 最深祖先);仅新增制品时才需要挂父级,存量卡不动

### 5. 展示计划并确认(删除必须逐项确认)

按候选展示规范以表格输出三段计划(列:`#`/动作/编号/名称/原因):

- **删除项逐项用户确认**——删错卡不可自动恢复,任何一项未获同意就从计划中剔除并改为"保留不动"
- 更新/新建项列出后整体确认一次即可
- `--dry-run`:输出完整计划后结束,不执行

### 6. 执行

- 顺序:删除 → 更新 → 新建(新建的父级一定先于子卡存在)
- 逐条执行,HTTP 429 按 `x-pc-retry-after` 重试;单条失败记入 `errors[]` 继续,不中断

### 7. 回写登记并输出

- 更新统一登记 `specs/pingcode-mapping.json` 该 spec 条目:
  - 删除成功的制品从 `artifacts` 移除
  - 新建成功的写入对应 `artifacts` 条目(`status: created` + 卡信息)
  - 全部卡的 `state`/`state_type` 刷新;`updated_at` 更新
- 输出总结:

```
═══════════════════════════════════════════
✅ 制品与卡片已同步(<spec-name>)
═══════════════════════════════════════════
删除: <n> 张    更新: <n> 张    新建: <n> 张
  • <动作> <identifier> - <标题>(<原因>)
登记: specs/pingcode-mapping.json(<spec-name> 条目)
═══════════════════════════════════════════
```

## 故障排除

| 症状 | 处理 |
|---|---|
| 登记无该 spec 条目 | 先运行 `/speckit.pingcode.specstoissues` 建卡 |
| 更新后标题/描述仍不一致 | 检查是否有人在 PingCode 侧手工改卡;以 spec 制品为准重新执行本命令 |
| 新建卡父级不存在(祖先被删) | 先在 PingCode 恢复/重建祖先层,或跳过该卡的挂接 |
| HTTP 429 | 等待 `x-pc-retry-after` 秒后重试 |
