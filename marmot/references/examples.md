# Marmot 使用示例

> 端到端的完整示例，覆盖所有核心场景。可直接复制使用。

## 目录

- [基础：阈值监控](#基础阈值监控)
- [多级阈值 + 静默](#多级阈值--静默)
- [手动告警](#手动告警)
- [Job 监控（装饰器）](#job-监控装饰器)
- [Job 监控（非装饰器）](#job-监控非装饰器)
- [心跳检测](#心跳检测)
- [手动恢复](#手动恢复)
- [指标聚合（集群平均磁盘）](#指标聚合集群平均磁盘)
- [指标聚合（错误计数）](#指标聚合错误计数)
- [多渠道通知](#多渠道通知)
- [升级策略](#升级策略)
- [自定义通知渠道](#自定义通知渠道)
- [Web 监控面板](#web-监控面板)
- [MarmotApp 实例模式（非单例）](#marmotapp-实例模式非单例)
- [测试最佳实践](#测试最佳实践)

---

## 基础：阈值监控

最简单的用法 — 注册阈值规则，上报指标：

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

marmot.register_threshold_rule(marmot.ThresholdRule(
    name="cpu_usage",
    thresholds=[
        marmot.ThresholdLevel(value=80, severity="warning"),
    ],
    consecutive_count=1,
    notify_targets=["console"],
))

# 上报 → 立即触发（consecutive_count=1）
event = marmot.report("cpu_usage", 92.5, labels={"host": "prod-1"})
# [2026-04-11T...] ⚠️ [WARNING] 🔥 cpu_usage: cpu_usage = 92.5 (warning) ...

# 上报正常值 → 自动恢复
event = marmot.report("cpu_usage", 45.0, labels={"host": "prod-1"})
# 告警自动 resolved

marmot.shutdown()
```

---

## 多级阈值 + 静默

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

marmot.register_threshold_rule(marmot.ThresholdRule(
    name="cpu_usage",
    thresholds=[
        marmot.ThresholdLevel(value=70, severity="info"),
        marmot.ThresholdLevel(value=80, severity="warning"),
        marmot.ThresholdLevel(value=95, severity="critical"),
    ],
    consecutive_count=2,       # 需连续 2 次超阈值
    silence_seconds=300,        # 触发后静默 5 分钟
    notify_targets=["console"],
))

# 连续 2 次超过 80 → warning 触发
marmot.report("cpu_usage", 85.0, labels={"host": "prod-1"})
marmot.report("cpu_usage", 87.0, labels={"host": "prod-1"})
# → state=firing, severity=warning

# 升级到 critical
marmot.report("cpu_usage", 96.0, labels={"host": "prod-1"})
# → severity=crITICAL

marmot.shutdown()
```

---

## 手动告警

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

event = marmot.fire(
    "payment_failure",
    "支付网关连续超时 5 次",
    severity="critical",
    labels={"service": "payment-gateway"},
    notify_targets=["console"],
)
print(f"Alert ID: {event.id}, State: {event.state}")

marmot.shutdown()
```

---

## Job 监控（装饰器）

最推荐的方式 — 零侵入：

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

@marmot.job("data_pipeline", timeout="30m", notify="console")
def run_pipeline():
    """定时任务。失败自动告警，成功自动恢复。"""
    # 正常执行
    return "success"

result = run_pipeline()
# → Job 成功，无告警

# 如果 job 抛异常 → 自动 fire 告警
@marmot.job("flaky_job", notify="console")
def flaky_job():
    raise RuntimeError("连接数据库超时")

try:
    flaky_job()
except RuntimeError:
    pass  # 异常会被重新抛出，同时触发告警

marmot.shutdown()
```

---

## Job 监控（非装饰器）

当装饰器不方便时（如动态函数、已有函数）：

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

def my_cleanup():
    return "done"

# 直接运行并监控
result = marmot.get_app().run_job(
    my_cleanup,
    name="cleanup",
    timeout="10m",
    notify="console",
)

marmot.shutdown()
```

---

## 心跳检测

监控一个定期运行的服务是否存活：

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

# 注册心跳规则
marmot.register_rule(marmot.Rule.from_inputs(
    name="data_pipeline",
    expected_interval="5m",
    notify="console",
))

# 模拟心跳丢失 → 手动触发
marmot.fire("data_pipeline", "心跳丢失超过 15 分钟", notify_targets=["console"])

# 收到心跳 → 自动恢复
marmot.ping("data_pipeline", message="Pipeline 已恢复")

marmot.shutdown()
```

---

## 手动恢复

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

# 先触发一个告警
marmot.fire("disk_space", "磁盘使用率 98%", severity="critical", notify_targets=["console"])

# 手动恢复
event = marmot.resolve("disk_space", message="磁盘已清理，使用率降至 45%")
print(f"Resolved at: {event.resolved_at}")

marmot.shutdown()
```

---

## 指标聚合（集群平均磁盘）

**场景**：100 个 ES 集群，当平均磁盘使用率超过 85% 时告警。

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

marmot.register_threshold_rule(marmot.ThresholdRule(
    name="es_disk",
    thresholds=[
        marmot.ThresholdLevel(value=85, severity="warning"),
        marmot.ThresholdLevel(value=95, severity="critical"),
    ],
    consecutive_count=1,
    aggregate=marmot.AggregateConfig(fn="avg", window=300),  # 5分钟窗口
    notify_targets=["console"],
))

# 各集群独立上报
clusters = [f"es-{i:03d}" for i in range(100)]
for name in clusters:
    disk_usage = 82.0 + (hash(name) % 20)  # 模拟 82~101 的磁盘使用率
    marmot.report("es_disk", min(disk_usage, 99.0), labels={"cluster": name})

# 检查告警
active = marmot.get_app().storage.list_active_alerts()
for alert in active:
    print(f"Alert: {alert.rule_name}, value={alert.current_value}, "
          f"samples={alert.labels.get('sample_count')}")

marmot.shutdown()
```

---

## 指标聚合（错误计数）

**场景**：5 分钟窗口内错误总数超过 100 时告警。

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

marmot.register_threshold_rule(marmot.ThresholdRule(
    name="error_count",
    thresholds=[marmot.ThresholdLevel(value=100, severity="error")],
    consecutive_count=1,
    aggregate=marmot.AggregateConfig(fn="sum", window=300),
    notify_targets=["console"],
))

# 各服务上报错误
for i in range(60):
    marmot.report("error_count", 1.0, labels={"service": "api"})

marmot.shutdown()
```

---

## 多渠道通知

同时发送到钉钉和企微：

```python
import marmot

marmot.configure(":memory:", start_escalation=False)

marmot.register_notifier("ding", marmot.DingTalkNotifier(
    webhook_url="https://oapi.dingtalk.com/robot/send?access_token=xxx",
    secret="SEC...",
))
marmot.register_notifier("wecom", marmot.WeComNotifier(
    webhook_url="https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx",
    mentioned_list=["@all"],
))

marmot.register_threshold_rule(marmot.ThresholdRule(
    name="cpu_usage",
    thresholds=[marmot.ThresholdLevel(value=90, severity="critical")],
    consecutive_count=1,
    notify_targets=["ding", "wecom"],  # 同时发两个渠道
))

marmot.report("cpu_usage", 95.0, labels={"host": "prod-1"})
# → 钉钉和企微同时收到通知

marmot.shutdown()
```

---

## 升级策略

告警 15 分钟未处理通知 oncall，30 分钟通知 manager：

```python
import marmot

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("console", marmot.ConsoleNotifier())

marmot.register_threshold_rule(marmot.ThresholdRule(
    name="critical_service",
    thresholds=[marmot.ThresholdLevel(value=0, severity="critical")],
    consecutive_count=1,
    notify_targets=["console"],
    escalation_steps=[
        marmot.EscalationStep(after_seconds="15m", notify=["oncall"]),
        marmot.EscalationStep(after_seconds="30m", notify=["manager"]),
    ],
))

marmot.shutdown()
```

---

## 自定义通知渠道

继承 `Notifier` 基类：

```python
import marmot

class SlackNotifier(marmot.Notifier):
    def __init__(self, webhook_url: str):
        self.url = webhook_url

    def send(self, n) -> bool:
        import urllib.request, json
        payload = {
            "text": f"[{n.severity.upper()}] {n.rule_name}: {n.message}",
        }
        body = json.dumps(payload).encode("utf-8")
        req = urllib.request.Request(
            self.url, data=body,
            headers={"Content-Type": "application/json"},
        )
        try:
            with urllib.request.urlopen(req, timeout=5) as resp:
                return 200 <= resp.status < 300
        except Exception:
            return False

marmot.configure(":memory:", start_escalation=False)
marmot.register_notifier("slack", SlackNotifier("https://hooks.slack.com/..."))
marmot.register_threshold_rule(marmot.ThresholdRule(
    name="test", thresholds=[marmot.ThresholdLevel(value=0, severity="info")],
    consecutive_count=1, notify_targets=["slack"],
))
marmot.report("test", 1.0)
marmot.shutdown()
```

---

## Web 监控面板

```python
import marmot

app = marmot.configure("alerts.db")
app.register_notifier("console", marmot.ConsoleNotifier())

# ... 注册规则、上报数据 ...

# 启动 Web 面板
ui = marmot.start_ui_server(app, port=8765)
print(f"Panel: {ui.url}")  # http://0.0.0.0:8765

# 后台运行，访问浏览器查看告警面板
# ui.stop() 关闭
```

---

## MarmotApp 实例模式（非单例）

适用于需要多个独立实例的场景：

```python
from marmot.app import MarmotApp
from marmot.notifiers import ConsoleNotifier
from marmot.models import ThresholdRule, ThresholdLevel

app = MarmotApp(":memory:")
app.register_notifier("console", ConsoleNotifier())
app.register_threshold_rule(ThresholdRule(
    name="cpu",
    thresholds=[ThresholdLevel(value=80, severity="warning")],
    consecutive_count=1,
    notify_targets=["console"],
))

event = app.report("cpu", 90.0, labels={"host": "prod-1"})

app.shutdown()
```

---

## 测试最佳实践

```python
import pytest
from marmot.app import MarmotApp
from marmot.models import ThresholdRule, ThresholdLevel, Notification
from marmot.notifiers import Notifier

class FakeNotifier(Notifier):
    def __init__(self):
        self.sent = []

    def send(self, n: Notification) -> bool:
        self.sent.append(n)
        return True

@pytest.fixture
def app():
    a = MarmotApp(":memory:")  # 内存数据库，测试完自动销毁
    a.register_notifier("fake", FakeNotifier())
    return a

def test_threshold_fires(app):
    app.register_threshold_rule(ThresholdRule(
        name="cpu",
        thresholds=[ThresholdLevel(value=80, severity="warning")],
        consecutive_count=1,
        notify_targets=["fake"],
    ))
    event = app.report("cpu", 90.0)
    assert event.state == "firing"
    assert len(app.notifiers["fake"].sent) == 1

def test_no_alert_below_threshold(app):
    app.register_threshold_rule(ThresholdRule(
        name="cpu",
        thresholds=[ThresholdLevel(value=80, severity="warning")],
        consecutive_count=1,
        notify_targets=["fake"],
    ))
    event = app.report("cpu", 50.0)
    assert event is None  # 未超阈值，无告警
```
