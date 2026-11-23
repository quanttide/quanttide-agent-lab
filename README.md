# 量潮智能体工程实验室

QuantTide Laboratory of Agent Engineering

## 实验

| 实验 | 说明 | 数据 |
|------|------|------|
| [experiments/session-scrub](experiments/session-scrub/README.md) | 会话脱敏导出：clean-session-curate 全流程实测、七项匿名化效果、密钥漏网与二次自查 | `data/session-scrub/` |
| [experiments/pi-session-scrub](experiments/pi-session-scrub/README.md) | pi 会话接入同一流水线：发现 250 条、转换、Capsule 盲区（数组内路径）、第二道清扫拦下 1014 处身份与部署信息 | `data/pi-session-scrub/` |
| [experiments/session-scrub-v2](experiments/session-scrub-v2/README.md) | 会话脱敏导出 v2：上游 [`linmou/clean-session-curate@v0.2.0`](https://github.com/linmou/clean-session-curate/releases/tag/v0.2.0) 四阶段分片多智能体零泄漏流水线，解决实验 1–2 五项遗留问题并在零污染 Holdout 集首轮一次性通过（`0` Critical/High/Medium） | [在线报告](https://linmou.github.io/clean-session-curate/) |

## 目录

| 目录 | 用途 |
|------|------|
| `experiments/` | 实验记录：目的、命令、实测结果、结论 |
| `data/` | 实验产出的数据，按实验名分子目录 |
