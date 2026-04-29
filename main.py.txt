
### 4. `main.py`
（这就是前面回复中的完整代码，直接复制即可）

```python
import json
from typing import List, Dict, Any, TypedDict, Optional
from langgraph.graph import StateGraph, END

# ---------- 1. 共享状态定义 ----------
class OpsState(TypedDict):
    raw_alerts: List[Dict[str, Any]]
    filtered_alerts: List[Dict[str, Any]]
    trace_data: List[Dict[str, Any]]
    root_cause_hypothesis: str
    repair_steps: List[str]
    current_agent: str
    error: Optional[str]

# ---------- 2. 模拟数据 ----------
MOCK_ALERTS = [
    {"id": 1, "service": "payment", "message": "High latency on /pay", "severity": "critical", "timestamp": "10:00:01"},
    {"id": 2, "service": "payment", "message": "DB connection pool exhausted", "severity": "critical", "timestamp": "10:00:02"},
    {"id": 3, "service": "notification", "message": "Low disk space", "severity": "warning", "timestamp": "10:00:03"},
    {"id": 4, "service": "gateway", "message": "CPU spike to 90%", "severity": "warning", "timestamp": "10:00:04"},
    {"id": 5, "service": "payment", "message": "Failed to deduct balance", "severity": "critical", "timestamp": "10:00:05"},
]

MOCK_TRACES = {
    "payment": [
        {"span_id": "abc", "operation": "/pay", "duration_ms": 8500, "status": "error",
         "logs": ["DB timeout after 5s", "Retry exhausted"], "downstream": "payment-db"},
        {"span_id": "abd", "operation": "check_balance", "duration_ms": 200, "status": "ok"},
    ]
}

# ---------- 3. Agent 节点 ----------
def log_filter_agent(state: OpsState) -> OpsState:
    print(" [Filter Agent] 正在过滤告警...")
    filtered = [a for a in state["raw_alerts"] if a["service"] == "payment" and a["severity"] == "critical"]
    state["filtered_alerts"] = filtered
    state["current_agent"] = "filter"
    print(f"   保留 {len(filtered)} 条有效告警: {[a['message'] for a in filtered]}")
    return state

def tracer_agent(state: OpsState) -> OpsState:
    print("[Tracer Agent] 正在拉取全链路追踪...")
    trace_data = []
    for alert in state["filtered_alerts"]:
        if alert["service"] in MOCK_TRACES:
            trace_data.extend(MOCK_TRACES[alert["service"]])
    state["trace_data"] = trace_data
    state["current_agent"] = "tracer"

    root_cause = "未发现明显异常"
    suspicious = [s for s in trace_data if s["status"] == "error" and s["duration_ms"] > 5000]
    if suspicious:
        root_span = suspicious[0]
        root_cause = (f"根因假设：{root_span['operation']} 调用下游 {root_span.get('downstream', 'N/A')} "
                      f"耗时 {root_span['duration_ms']}ms，触发超时。日志：{root_span['logs']}")
    state["root_cause_hypothesis"] = root_cause
    print(f"   追踪到 {len(trace_data)} 个 Span，根因假设：{root_cause}")
    return state

def advisor_agent(state: OpsState) -> OpsState:
    print("[Advisor Agent] 生成修复建议...")
    runbook = {
        "DB connection pool exhausted": [
            "1. 检查数据库连接池最大连接数配置 (max_connections)",
            "2. 排查慢查询，优化 top 3 慢 SQL",
            "3. 临时扩容只读副本以分担压力"
        ],
        "High latency": [
            "1. 检查下游依赖服务健康状态",
            "2. 查看是否有网络延迟或 DNS 解析问题"
        ]
    }
    steps = []
    for alert in state["filtered_alerts"]:
        for key, actions in runbook.items():
            if key.lower() in alert["message"].lower():
                steps.extend(actions)
    if "DB timeout" in state["root_cause_hypothesis"]:
        steps.append("4. 针对 payment-db 添加断路器和重试机制")
    state["repair_steps"] = list(set(steps))
    state["current_agent"] = "advisor"
    print(f"   生成修复步骤: {state['repair_steps']}")
    return state

def should_continue(state: OpsState) -> str:
    if not state.get("filtered_alerts"):
        state["error"] = "无有效告警，流程终止"
        return END
    return "tracer"

# ---------- 4. 构建图 ----------
def build_graph():
    workflow = StateGraph(OpsState)
    workflow.add_node("filter", log_filter_agent)
    workflow.add_node("tracer", tracer_agent)
    workflow.add_node("advisor", advisor_agent)
    workflow.set_entry_point("filter")
    workflow.add_conditional_edges("filter", should_continue, {"tracer": "tracer", END: END})
    workflow.add_edge("tracer", "advisor")
    workflow.add_edge("advisor", END)
    return workflow.compile()

# ---------- 5. 运行 ----------
if __name__ == "__main__":
    graph = build_graph()
    initial_state: OpsState = {
        "raw_alerts": MOCK_ALERTS,
        "filtered_alerts": [],
        "trace_data": [],
        "root_cause_hypothesis": "",
        "repair_steps": [],
        "current_agent": "",
        "error": None
    }
    print("===== 多 Agent 智能运维排障机器人启动 =====")
    final_state = graph.invoke(initial_state)
    print("\n===== 最终报告 =====")
    print(json.dumps({
        "filtered_alerts_count": len(final_state["filtered_alerts"]),
        "root_cause": final_state["root_cause_hypothesis"],
        "repair_steps": final_state["repair_steps"]
    }, indent=2, ensure_ascii=False))