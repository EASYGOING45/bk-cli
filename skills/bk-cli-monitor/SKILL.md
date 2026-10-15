---
name: bk-cli-monitor
description: 当需要通过 `bk-cli monitor` 以用户权限查询蓝鲸观测数据时使用：业务空间、指标、日志、告警、事件、APM/Trace/Profiling、仪表盘、关联关系、告警配置、日志采集或日志提取状态。用户给出 bk_biz_id、告警 ID、索引集 ID、APM 应用、trace_id、指标表等标识，或要求判断某个观测工具能否执行时，优先使用本 skill。
---

# bk-cli monitor — 蓝鲸观测用户态查询

CRITICAL — 开始前 MUST 先读取 `../bk-cli-shared/SKILL.md`。共享 skill 负责 context、认证、dry-run 和通用输入；本 skill 只说明 `monitor` 的分组、参数和判定。

## 先判断命令形态

- 业务查询使用 `bk-cli monitor <module> <tool_name>`；自检使用 `bk-cli monitor status`，配置使用 `bk-cli monitor profile list/show/check`。
- action 名和 flag 名保持 Unified MCP 原名，不自造别名，例如 `search_logs`、`list_alerts`、`bk_biz_id`、`index_set_id`。
- 不知道工具时先运行 `bk-cli monitor tools search --bk_biz_id <id> --category <module>`。返回里的 `command` 就是 CLI 入口。
- 权限问题使用 `bk-cli monitor permission ...`；不要绕过 `monitor` 直接调用 IAM。

| 用户意图 | module |
|---|---|
| 空间、业务 ID、BCS 集群 | `space` |
| 指标表、指标字段、PromQL、SQL | `metric` |
| 日志场景、索引集、字段、检索、上下文、聚类 | `log` |
| 告警列表、告警详情、告警事件、关联日志/Trace | `alert` |
| 自定义事件、日志关键字事件 | `event` |
| APM 应用、服务、Span、Trace、Profiling | `apm` |
| 仪表盘目录和详情 | `dashboard` |
| 资源关联关系 | `relation` |
| 告警策略、屏蔽、通知组、处理套餐（只读） | `alarm` |
| 日志采集项、索引集分组、清洗预览（只读） | `collect` |
| 日志提取范围和任务状态（只读） | `extract` |
| 查工具、看契约 | `tools` |
| 看权限、预检、获取申请链接 | `permission` |
| SRE 交付的具名资源配置 | `profile` |

## 优先使用业务提供的 Profile

用户或业务 Skill 已指定 Profile 时，使用 `--profile NAME`，不要每次重新发现索引、猜表名或生成 ID 关联表。不确定名字时运行 `bk-cli monitor profile list`，多个候选由用户选择。

```bash
export BK_CLI_MONITOR_PROFILE_FILE=./team-profiles.yaml
bk-cli monitor profile show --profile checkout-prod
bk-cli monitor status --profile checkout-prod
bk-cli monitor log search_logs --profile checkout-prod --since 15m --query_string ERROR --limit 5
bk-cli monitor permission apply --profile checkout-prod --tool search_logs
```

V2 业务 Profile 还需选择 `--component`；同 module 有多个具名查询时选择 `--binding`。先用 `profile show` 查看实际 Source，不把被观测环境名当成数据存放位置。普通查询不必每次预检。Profile 只补默认参数，不是权限或凭据；context 仍负责环境与登录。冲突时报错，不自动切环境或换日志模式。没有 Profile 时沿用显式参数和目录发现。

SRE 定制 Profile / 多应用 / 多 namespace / FDE 交付时，读取 [Profile 使用与交付](references/profiles.md)，示例是 [profiles.yaml](references/profiles.yaml)。二进制内也可用 `bk-cli skills read bk-cli-monitor references/profiles.md` 读取。

自检失败看 `checks`；`warn` 和 `skipped` 不代表验证通过。`status --offline` 不联网，`status --check_update` 才检查官方发布版本，不自动升级。

## 输入约定

- 普通标量参数用同名 flag；数组参数可重复或逗号分隔；对象和对象数组传 JSON。
- 所有工具都支持 `--body` 传完整 `tool_args`。同名 flag 会覆盖 body 中的字段。
- 顶层 start/end 可用 `--since 15m` 或显式日期/RFC3339/时间戳，二者互斥（body 中也算）。无偏移日期默认 Asia/Shanghai，可加 `--timezone`；数字不按位数猜单位，支持显式 s/ms 后缀。CLI 按契约生成秒/毫秒及整数/字符串。涉及时间窗口或单位时读 [查询时间](references/time-inputs.md)，不要把此规则套到 query_list 或日志定位字段。
- `search_logs` / `get_scene_log_fields` 支持 `--scene host --where KEY=VALUE` 生成选表路由。namespace 等日志行字段用 `search_logs --filter FIELD=VALUE` 或 Profile 的 `log.filters`，不要误放进 `where`。
- `search_index_set_context` 支持 `--from hit.json`，只复制检索结果里的原始定位字段。
- 不要猜业务 ID、索引集 ID、告警 ID、应用名、trace_id 或指标表。缺少时使用错误输出中的 `next`。

## 常用场景

```bash
# 按名称找业务
bk-cli monitor space search_spaces --space_name 蓝鲸

# 最近告警 → 告警详情 → 关联日志
bk-cli monitor alert list_alerts --bk_biz_id <id> --status ABNORMAL --since 1h
bk-cli monitor alert get_alert_info --bk_biz_id <id> --id <alert_id>
bk-cli monitor alert get_alert_log_relations --bk_biz_id <id> --alert_id <alert_id>

# 场景日志 → 索引集日志 → 上下文
bk-cli monitor log search_logs --bk_biz_id <id> --scene host --query_string ERROR --since 15m
bk-cli monitor log search_logs --bk_biz_id <id> --index_set_id <index_set_id> --limit 1 --since 1h > hit.json
bk-cli monitor log search_index_set_context --bk_biz_id <id> --from hit.json --size 5

# 指标表 → 指标 → PromQL
bk-cli monitor metric list_time_series_groups --bk_biz_id <id>
bk-cli monitor metric list_time_series_metrics --bk_biz_id <id> --table_id <table_id>
bk-cli monitor metric execute_range_query --bk_biz_id <id> --promql '<promql>' --since 1h

# APM 应用 → Span → Trace
bk-cli monitor apm list_apm_applications --bk_biz_id <id>
bk-cli monitor apm search_spans --bk_biz_id <id> --app_name <app> --since 15m
bk-cli monitor apm get_trace_detail --bk_biz_id <id> --app_name <app> --trace_id <trace_id>
```

## 权限与申请

每个模块和工具的 `--help` 都列出所需权限。运行时以服务端判定为准。

```bash
# 看自己在业务下的权限状态
bk-cli monitor permission list --bk_biz_id <id>
bk-cli monitor permission list --bk_biz_id <id> --module log --missing_only

# 单个工具和具体资源；不读取数据
bk-cli monitor permission check --bk_biz_id <id> --tool get_index_set_fields --body '{"index_set_id":123}'

# 获取申请链接；不自动提交
bk-cli monitor permission apply --bk_biz_id <id> --tool list_alerts
bk-cli monitor permission apply --bk_biz_id <id> --module alert
```

- 普通查询不必先跑权限命令。执行被拒时，错误会给出 `permission apply` 命令。
- MCP 权限和 SaaS 原生权限满足其一即可。`apply` 会列出两种申请入口。
- `apply` 只返回 IAM 链接。用户需要在浏览器打开并确认；CLI 不自动申请、不授权。
- `needs_resource`：还缺索引集、应用、仪表盘等具体资源；不是无权。
- `resource_not_visible` / `resource_not_found`：重新从列表命令选择资源。
- `permission_missing`：服务端明确无权；使用 `permission apply` 给出的链接。
- `resource_blocked`：服务端元数据问题；不要引导申请更多权限。
- 不查询他人权限，也不对全部资源实例逐个枚举。
- 当前只暴露 65 个只读工具；写入、任务、导出不会生成命令。
