# Multi-Agent Ops Troubleshooter (多 Agent 协作智能运维排障机器人)

一个基于 LangGraph 的多 Agent 协作原型，实现了“筛选—诊断—修复建议”三级自动排障。

## 核心痛点
线上故障平均定位耗时 45 分钟，初级工程师面对海量告警时无法快速关联跨服务日志，误判率高达 30%。

## 逻辑流
1. **Filter Agent** – 从告警流中过滤关键严重告警。
2. **Tracer Agent** – 根据告警拉取分布式全链路追踪，进行长链推理定位根因。
3. **Advisor Agent** – 结合历史 Runbook 生成修复建议与具体步骤。

三个 Agent 通过 LangGraph 的共享状态（事件总线模式）串联，支持真实 Kafka/Jaeger 替换。

## 运行
```bash
pip install -r requirements.txt
python main.py