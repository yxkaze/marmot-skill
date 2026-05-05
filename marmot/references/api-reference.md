# Marmot API Reference

> 完整的类、方法、参数说明。按模块组织。

## 目录

- [枚举类型](#枚举类型)
- [数据模型](#数据模型)
- [核心引擎 MarmotApp](#核心引擎-marmotapp)
- [模块级 API（单例模式）](#模块级-api单例模式)
- [通知渠道 Notifier](#通知渠道-notifier)
- [指标聚合 MetricBucket](#指标聚合-metricbucket)
- [存储层 SQLiteStorage](#存储层-sqlitestorage)

---

## 枚举类型

### AlertState
告警状态机状态。

| 值 | 说明 |
|----|------|
| `pending` | 等待连续命中 |
| `firing` | 已触发 |
| `silenced` | 静默中 |
| `escalated` | 已升级 |
| `resolving` | 恢复确认中 |
| `resolved` | 已恢复（终态） |

### Severity
严重程度。

| 值 | 说明 |
|----|------|
| `info` | 信息 |
| `warning` | 警告 |
| `error` | 错误 |
| `critical` | 严重 |

### AlertStage
触发机制。

| 值 | 说明 |
|----|------|
| `threshold` | 阈值触发 |
| `timeout` | 超时触发 |
| `heartbeat` | 心跳丢失 |
| `manual` | 手动 fire() |

### RunStatus
Job 执行状态。

| 值 | 说明 |
|----|------|
| `running` | 执行中 |
| `success` | 成功 |
| `failed` | 失败 |
| `timeout` | 超时 |

### NotificationStatus
通知发送状态。

| 值 | 说明 |
|----|------|
| `pending` | 待发送 |
| `sent` | 已发送 |
| `failed` | 发送失败 |

### AggregateFn
聚合函数。

| 值 | 说明 |
|----|------|
| `avg` | 平均值 |
| `max` | 最大值 |
| `min` | 最小值 |
| `sum` | 求和 |
| `count` | 计数 |

---

## 数据模型

### ThresholdRule

阈值规则。用于 `report()` 场景。

```python
marmot.ThresholdRule(
    name: str,                              # 规则名称（唯一标识）
    thresholds: list[ThresholdLevel],        # 阈值级别列表
    consecutive_count: int = 1,              # 连续命中几次才触发（默认1，立即触发）
    silence_seconds: float = 300,            # 触发后静默秒数（默认5分钟）
    notify_targets: list[str] = [],          # 通知渠道名称列表
    escalation_steps: list[EscalationStep] = [],  # 升级步骤
    group_key: str | None = None,            # 分组键（预留）
    aggregate: AggregateConfig | None = None, # 聚合配置（可选）
)
```

**evaluate(value: float) → ThresholdLevel | None**

返回匹配的最高阈值级别，未匹配返回 None。value >= threshold.value 即匹配。

### ThresholdLevel

单个阈值级别。

```python
marmot.ThresholdLevel(
    value: float,                    # 阈值
    severity: str,                   # 严重程度
    notify: list[str] = [],          # 覆盖通知目标（可选）
    silence_seconds: float = 0,      # 覆盖静默时间（可选）
)
```

### Rule

通用规则。用于 heartbeat / job 监控。通常通过 `Rule.from_inputs()` 创建。

```python
marmot.Rule.from_inputs(
    name: str,
    expected_interval: str | float | None = None,  # 预期间隔 "5m"
    timeout: str | float | None = None,             # 超时时间 "30m"
    silence: str | float | None = None,             # 静默时间
    group_by: str | None = None,                    # 分组键
    severity: str = "error",                        # 严重程度
    notify: str | list[str] | None = None,          # 通知渠道
    escalate: list | None = None,                   # 升级步骤
)
```

### EscalationStep

升级步骤。

```python
marmot.EscalationStep(
    after_seconds: float,              # 触发升级的秒数
    notify: list[str] = [],            # 升级通知目标
)
```

也支持从字典创建：`EscalationStep.from_value({"after": "15m", "notify": "oncall"})`

### AggregateConfig

聚合配置。

```python
marmot.AggregateConfig(
    fn: str = "avg",          # 聚合函数：avg/max/min/sum/count
    window: float = 300.0,    # 滑动窗口秒数
)
```

### AlertEvent

告警事件实体。

**字段：**
- `id: int | None` — 数据库自增 ID
- `rule_name: str` — 规则名
- `dedup_key: str` — 去重键
- `state: str` — 当前状态（AlertState 值）
- `severity: str` — 严重程度
- `stage: str` — 触发机制
- `message: str` — 告警消息
- `labels: dict` — 标签（聚合模式下含 aggregate_fn, sample_count）
- `current_value: float | None` — 当前指标值
- `consecutive_hits: int` — 连续命中次数
- `consecutive_misses: int` — 连续未命中次数
- `fired_at: datetime` — 触发时间
- `resolved_at: datetime | None` — 恢复时间
- `silenced_until: datetime | None` — 静默到期时间
- `escalated_at: datetime | None` — 升级时间
- `last_notified_at: datetime | None` — 最后通知时间
- `notification_count: int` — 通知次数

**方法：**
- `to_dict() → dict` — 转为可序列化字典
- `from_row(row) → AlertEvent` — 从数据库行构造

### RunRecord

Job 执行记录。

**字段：** `id`, `rule_name`, `dedup_key`, `status`, `message`, `error`, `labels`, `started_at`, `finished_at`

**属性：** `duration_ms: float` — 执行耗时（毫秒）

### Notification

通知记录。

**字段：** `alert_event_id`, `rule_name`, `dedup_key`, `status`, `state`, `message`, `severity`, `labels`, `stage`, `notifier_name`, `sent_at`

---

## 核心引擎 MarmotApp

```python
app = marmot.MarmotApp(db_path: str = "marmot.db")
```

### 注册方法

#### register_rule(rule: Rule) → None
注册通用规则（heartbeat/job）。

#### register_threshold_rule(rule: ThresholdRule) → None
注册阈值规则。

#### register_notifier(name: str, notifier: Notifier) → None
注册通知渠道。

#### unregister_notifier(name: str) → None
注销通知渠道。

### 核心动作

#### report(metric, value, *, labels=None) → AlertEvent | None
上报指标值，框架自动做阈值判定。

- `metric: str` — 规则名
- `value: float` — 当前值
- `labels: dict` — 可选标签（用于 dedup）
- 返回 AlertEvent 或 None（无匹配规则时）

当规则配置了 `aggregate` 时，自动进入聚合模式：
- 收集数据点到滑动窗口
- 计算聚合值后再做阈值判定
- 告警以规则名为 dedup_key

#### fire(name, message, *, severity="error", labels=None, notify_targets=None) → AlertEvent
手动触发告警，绕过阈值判定。

#### ping(name, *, labels=None, message="") → None
心跳上报。收到 ping 自动恢复对应告警。

#### resolve(name, *, labels=None, message="") → AlertEvent | None
手动恢复告警。

#### job(name, *, expected_interval=None, timeout=None, notify=None, labels=None) → Callable
装饰器，监控一个函数作为定时 Job。

```python
@app.job("pipeline", timeout="10m", notify="ding")
def run_pipeline():
    ...
```

#### run_job(func, name, *, ...) → Any
非装饰器 API，直接运行一个监控 Job。

### 生命周期

#### start_escalation_checker(interval_seconds=10.0) → None
启动后台升级检查线程。

#### stop_escalation_checker() → None
停止升级检查线程。

#### shutdown() → None
优雅关闭（停止线程、关闭数据库）。

---

## 模块级 API（单例模式）

```python
marmot.configure(db_path="marmot.db", *, start_escalation=True) → MarmotApp
```
初始化默认单例。后续可直接使用模块级函数：

```python
marmot.register_rule(rule)
marmot.register_threshold_rule(rule)
marmot.register_notifier(name, notifier)
marmot.report(metric, value, labels=...)
marmot.fire(name, message, ...)
marmot.ping(name, ...)
marmot.resolve(name, ...)
marmot.job(name, ...)       # 装饰器
marmot.shutdown()
marmot.get_app()            # 获取单例
```

---

## 通知渠道 Notifier

### 基类

```python
class Notifier(abc.ABC):
    @abc.abstractmethod
    def send(self, n: Notification) -> bool: ...
```

### ConsoleNotifier

```python
marmot.ConsoleNotifier()  # 无参数
```

### WebhookNotifier

```python
marmot.WebhookNotifier(
    url: str,
    headers: dict[str, str] | None = None,
    timeout: float = 5.0,
)
```

### MarkdownWebhookNotifier

```python
marmot.MarkdownWebhookNotifier(
    url: str,
    headers: dict[str, str] | None = None,
    timeout: float = 5.0,
)
```

### DingTalkNotifier

```python
marmot.DingTalkNotifier(
    webhook_url: str,                    # 完整 webhook URL（含 access_token）
    secret: str | None = None,           # 签名密钥（可选）
    timeout: float = 5.0,
)
```

### WeComNotifier

```python
marmot.WeComNotifier(
    webhook_url: str,                    # 完整 webhook URL（含 key）
    mentioned_list: list[str] | None = None,  # @mention 列表（["@all"]）
    timeout: float = 5.0,
)
```

### FeishuNotifier

```python
marmot.FeishuNotifier(
    webhook_url: str,
    secret: str | None = None,           # 签名密钥
    timeout: float = 5.0,
)
```

### EmailNotifier

```python
marmot.EmailNotifier(
    send_fn: Callable[[str, str, list[str]], bool],  # (subject, body, to) → bool
    to: list[str],                                     # 收件人列表
    from_addr: str | None = None,
)
```

### PhoneNotifier

```python
marmot.PhoneNotifier(
    send_fn: Callable[[str, list[str]], bool],  # (message, phones) → bool
    to: list[str],                                  # 手机号列表
)
```

> 注意：PhoneNotifier 仅在 severity 为 critical 或 error 时发送。

---

## 指标聚合 MetricBucket

```python
bucket = marmot.MetricBucket()
```

### add(rule_name: str, value: float) → None
添加数据点到指定规则的缓冲区。

### compute(rule_name: str, fn: str, window: float) → tuple[float | None, int]
计算聚合值。

- `fn`: "avg" / "max" / "min" / "sum" / "count"
- `window`: 滑动窗口秒数（自动清理过期数据）
- 返回 `(聚合值, 样本数)`，无数据时为 `(None, 0)`

### clear(rule_name: str | None = None) → None
清空缓冲区。`rule_name=None` 时清空全部。

### sample_count(rule_name: str) → int
当前缓冲区数据点数（诊断用）。

---

## 存储层 SQLiteStorage

```python
storage = marmot.SQLiteStorage(path: str = "marmot.db")
```

通常不需要直接使用，`MarmotApp` 内部管理。支持 `:memory:` 用于测试。

### AlertEvent 方法
- `create_alert_event(event) → AlertEvent`
- `update_alert_event(event) → None`
- `get_alert(alert_id: int) → AlertEvent | None`
- `get_active_alert(dedup_key: str) → AlertEvent | None`
- `list_active_alerts() → list[AlertEvent]`
- `list_alert_history(limit=100) → list[AlertEvent]`
- `list_silenced_alerts() → list[AlertEvent]`
- `list_escalatable_alerts() → list[AlertEvent]`

### RunRecord 方法
- `create_run(run) → RunRecord`
- `update_run(run) → None`
- `get_run(run_id) → RunRecord | None`
- `get_latest_run(dedup_key) → RunRecord | None`
- `list_runs(limit=100) → list[RunRecord]`

### Notification 方法
- `record_notification(n) → int`
- `list_notifications(alert_event_id=None, limit=200) → list[dict]`

### Rule 方法
- `upsert_rule(rule) / get_rule(name) / list_rules() / delete_rule(name)`
- `upsert_threshold_rule(rule) / get_threshold_rule(name) / list_threshold_rules() / delete_threshold_rule(name)`

---

## Web UI

```python
ui = marmot.start_ui_server(app, host="0.0.0.0", port=8765)
```

| 端点 | 说明 |
|------|------|
| `GET /` | HTML 监控面板 |
| `GET /api/alerts` | 活跃告警列表 |
| `GET /api/alerts/:id` | 告警详情（含通知记录） |
| `GET /api/history` | 已恢复告警历史 |
| `GET /api/runs` | Job 执行记录 |
| `GET /api/notifications` | 通知日志 |
| `GET /api/rules` | 已注册规则 |

`ui.stop()` 关闭服务。
