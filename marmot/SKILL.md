---
name: marmot
description: >
  Marmot (土拨鼠) — Python 轻量级告警框架的知识库。
  当用户需要：编写告警监控代码、设置阈值规则、配置通知渠道（钉钉/企微/飞书/Email/Phone/Webhook）、
  实现心跳检测、Job 监控、指标聚合、状态机管理、或任何关于 Marmot 框架的使用问题时，
  使用此 Skill。即使没有明确提到 "Marmot"，只要用户的需求涉及"业务告警"、"阈值监控"、
  "异常通知"、"heartbeat 监控"、"job 失败告警"、"指标聚合告警"等场景，也应使用此 Skill。
  关键词：alert, monitoring, threshold, heartbeat, ping, job monitor, notification,
  钉钉机器人, 企微 webhook, 飞书通知, dedup, silence, escalation, aggregate.
metadata:
  author: Marmot
  version: "0.2.0"
---

# Marmot (土拨鼠) — Python 轻量级告警框架

Marmot 是一个面向业务侧的轻量级告警框架，核心设计原则：**框架不替开发者做决策，开发者传什么框架信什么**。
零外部依赖（仅 Python 标准库 + SQLite），开箱即用。

## 核心概念速览

| 概念 | 说明 |
|------|------|
| **ThresholdRule** | 阈值规则，report() 上报指标值，超过阈值则告警 |
| **Rule** | 通用规则，用于 heartbeat/job 监控 |
| **AlertEvent** | 告警事件实体，贯穿状态机全生命周期 |
| **RunRecord** | Job/心跳执行记录 |
| **Notifier** | 通知渠道基类，7 种内置实现 |
| **MetricBucket** | 滑动窗口指标聚合（avg/max/min/sum/count） |

## 告警状态机

```
PENDING → FIRING → RESOLVING → RESOLVED
              ↘ SILENCED → FIRING
              ↘ ESCALATED → RESOLVING
```

- **PENDING**: 等待连续命中（consecutive_count）
- **FIRING**: 已触发告警
- **SILENCED**: 静默窗口内，不重复通知
- **ESCALATED**: 已升级（通知更高级别接收人）
- **RESOLVING**: 指标恢复正常，等待确认
- **RESOLVED**: 告警已恢复

## 快速开始

详细 API 和完整示例请阅读 `references/api-reference.md` 和 `references/examples.md`。

```python
import marmot

# 1. 初始化（默认使用 SQLite 持久化）
marmot.configure("alerts.db")

# 2. 注册通知渠道
marmot.register_notifier("ding", marmot.DingTalkNotifier(
    webhook_url="https://oapi.dingtalk.com/robot/send?access_token=xxx",
    secret="SEC...",
))

# 3. 注册阈值规则
marmot.register_threshold_rule(marmot.ThresholdRule(
    name="cpu_usage",
    thresholds=[
        marmot.ThresholdLevel(value=80, severity="warning"),
        marmot.ThresholdLevel(value=95, severity="critical"),
    ],
    consecutive_count=3,          # 连续 3 次超阈值才告警
    silence_seconds=300,           # 触发后静默 5 分钟
    notify_targets=["ding"],
))

# 4. 上报指标
marmot.report("cpu_usage", 92.5, labels={"host": "prod-1"})
```

## 四种核心动作

### 1. report() — 指标上报 + 阈值判定

```python
marmot.report("cpu_usage", 92.5, labels={"host": "prod-1"})
```

- 适用于持续型指标（CPU、内存、磁盘、QPS 等）
- 支持 labels 做 dedup（相同 labels 合并为同一告警）
- 支持 multi-level thresholds（不同阈值不同 severity）

### 2. fire() — 手动触发告警

```python
marmot.fire("payment_failure", "支付网关超时",
            severity="critical", notify_targets=["ding"])
```

- 绕过阈值判定，直接触发
- 适用于异常捕获、手动告警等场景

### 3. ping() — 心跳上报

```python
marmot.ping("data_pipeline", labels={"worker": "w-1"},
            message="Pipeline 正常")
```

- 配合 Rule（expected_interval / timeout）自动检测心跳丢失
- 收到 ping 自动恢复对应告警

### 4. job() — Job 装饰器监控

```python
@marmot.job("cleanup", timeout="30m", notify="ding")
def cleanup_job():
    # job 失败自动 fire 告警
    # job 成功自动 resolve 告警
    ...
```

- 装饰器模式，零侵入
- 失败自动告警，成功自动恢复

## 指标聚合

当需要监控一组实例的整体状态时（如 100 个 ES 集群的平均磁盘），使用聚合能力：

```python
marmot.register_threshold_rule(marmot.ThresholdRule(
    name="es_disk",
    thresholds=[marmot.ThresholdLevel(value=85, severity="warning")],
    aggregate=marmot.AggregateConfig(fn="avg", window=300),  # 5分钟窗口 avg
    notify_targets=["ding"],
))

# 各实例独立上报，框架自动聚合
for cluster in clusters:
    marmot.report("es_disk", disk_usage, labels={"cluster": cluster.name})
```

支持的聚合函数：`avg`、`max`、`min`、`sum`、`count`。

聚合模式下的告警以规则名为 dedup_key，labels 中包含 `aggregate_fn` 和 `sample_count`。

## 通知渠道

| 渠道 | 类名 | 说明 |
|------|------|------|
| 控制台 | `ConsoleNotifier` | 开发调试用，直接 print |
| 通用 Webhook | `WebhookNotifier(url)` | 原始 JSON POST |
| Markdown Webhook | `MarkdownWebhookNotifier(url)` | Slack/Discord/通用 Markdown |
| 钉钉 | `DingTalkNotifier(webhook_url, secret)` | 支持签名 |
| 企微 | `WeComNotifier(webhook_url)` | 支持 @mention |
| 飞书 | `FeishuNotifier(webhook_url, secret)` | 支持签名 + 卡片消息 |
| 邮件 | `EmailNotifier(send_fn, to)` | 回调模式，零依赖 |
| 电话/短信 | `PhoneNotifier(send_fn, to)` | 回调模式，仅 critical/error |

也可以继承 `Notifier` 基类实现自定义渠道：

```python
class MyNotifier(marmot.Notifier):
    def send(self, n) -> bool:
        # n: Notification 对象，包含 rule_name, message, severity 等
        ...
        return True
```

## 静默与升级

### 静默（Silence）
通过 `ThresholdRule.silence_seconds` 配置，告警触发后进入静默窗口，不重复通知。

### 升级（Escalation）
```python
marmot.ThresholdRule(
    name="critical_alert",
    ...,
    escalation_steps=[
        marmot.EscalationStep(after_seconds="15m", notify=["oncall"]),
        marmot.EscalationStep(after_seconds="30m", notify=["manager"]),
    ],
)
```

升级检查器自动在后台运行（`configure()` 时默认启动）。

## 手动恢复

```python
marmot.resolve("payment_failure", message="网关已恢复")
```

## Web 监控面板

```python
ui = marmot.start_ui_server(app, host="0.0.0.0", port=8765)
# 访问 http://localhost:8765 查看告警面板
# ui.stop() 关闭
```

内置 API 端点：`/api/alerts`、`/api/history`、`/api/runs`、`/api/notifications`、`/api/rules`。

## 架构文件参考

编写 Marmot 相关代码时，根据需要阅读以下参考文件：

| 文件 | 何时阅读 |
|------|----------|
| `references/api-reference.md` | 需要查看完整的类/方法签名、参数说明时 |
| `references/examples.md` | 需要完整的端到端使用示例时 |
