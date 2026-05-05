# Marmot Skill

DuMate skill for [Marmot](https://github.com/yxkaze/marmot) - a lightweight Python alerting framework.

## What is Marmot?

Marmot (土拨鼠) is a lightweight alerting framework for Python with zero external dependencies. It provides:

- **Threshold-based alerting** - Monitor metrics like CPU, memory, disk usage
- **Heartbeat monitoring** - Detect when services stop responding
- **Job monitoring** - Automatic alerts for failed scheduled tasks
- **Metric aggregation** - Monitor cluster-wide averages
- **7 notification channels** - DingTalk, WeCom, Feishu, Email, Phone, Webhook, Console
- **State machine** - PENDING → FIRING → RESOLVING with silence/escalation support

## Installation

### Option 1: Download .skill file

Download `marmot.skill` from [Releases](https://github.com/yxkaze/marmot-skill/releases) and install in DuMate.

### Option 2: Clone this repo

```bash
git clone https://github.com/yxkaze/marmot-skill.git
# Copy marmot/ folder to your DuMate skills directory
```

## Quick Start

Once installed, DuMate will automatically use this skill when you ask about:

- Alert monitoring and threshold rules
- Notification setup (DingTalk, Feishu, WeCom, etc.)
- Heartbeat detection and job monitoring
- Metric aggregation for clusters

**Example prompts:**

- "Help me set up CPU monitoring with DingTalk alerts"
- "I need heartbeat monitoring for my data pipeline"
- "How do I use Marmot's job decorator for scheduled tasks?"
- "Configure escalation: notify oncall after 15min, manager after 30min"

## Resources

- [Marmot GitHub](https://github.com/yxkaze/marmot) - Main framework repository
- [API Reference](marmot/references/api-reference.md) - Complete API documentation
- [Examples](marmot/references/examples.md) - End-to-end usage examples

## License

MIT
