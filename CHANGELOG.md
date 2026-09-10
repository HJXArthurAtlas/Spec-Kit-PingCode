# Changelog

本项目的所有显著变更将记录在本文件。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/),
版本遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [1.0.0] - 2026-09-10

### Added

- `extension.yml` 扩展清单(命令注册、配置模板、after_tasks 钩子)
- `/speckit.pingcode.specstoissues`:SPEC.md + TASKS.md → PingCode 工作项层级
  - 默认 2 层模式(story + task,Phase 嵌入描述);支持 3 层与极简模式
  - 迭代运行时解析链(参数 > 配置 > 上下文 > 自动发现/询问)
  - 幂等防护(补建/重建/中止)、429 限流重试、逐任务错误隔离
- `/speckit.pingcode.discover-context`:项目类型/状态/迭代字典探查与配置片段生成
- `/speckit.pingcode.sync-status`:本地勾选状态 → PingCode 状态流转,story 自动收尾,
  远端先行变更不回退(差异报告)
- `pingcode-config.template.yml` 配置模板与环境变量覆盖
- 映射产物 `pingcode-mapping.json` 与同步日志 `pingcode-sync-log.json`
