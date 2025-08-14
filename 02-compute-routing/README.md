# 02 企业 AI 算力方案（智能路由）

本章节聚焦"多云多模型智能路由"，以成本、时延与质量三要素为核心，构建企业级稳定、可控与可演进的推理基础设施。

## 1. 问题与诉求

- **多模型/多云供应商差异大**：价格、速度、能力、配额、地域与合规
- **期望目标**：稳定可用（SLA）、成本可控（Cost/Saving）、质量可评（QoS/Eval）

## 2. 架构概览

- **请求入口**：API 网关（鉴权、配额、限流、灰度）
- **路由引擎**：策略（静态/动态）、多臂赌博、质量反馈、降级/重试
- **评估与反馈**：离线评测、在线指标、回放与对比
- **支撑组件**：特征缓存、提示模板管理、可观测（Tracing/Logging/Metrics)

## 3. 路由策略与关键指标

- **指标**：时延、成功率、成本、质量评分（自动或人评）、风控标签
- **策略**：按地域/时段、按负载、按业务重要度、按质量/成本上限
- **机制**：熔断/降级、重试、旁路（shadow）、A/B 与流量回放

## 4. 策略配置示例（YAML）

```yaml
# 路由策略示例（按质量与成本加权）
targets:
  - name: vendor_a:gpt-4o
    price: 15.0
    region: ap-sg
  - name: vendor_b:llama3-70b
    price: 3.2
    region: cn-bj
policy:
  objective: "min_cost_under_latency_qos"
  weights: { cost: 0.6, latency: 0.2, quality: 0.2 }
  latency_budget_ms: 2000
  quality_threshold: 0.78
  retry:
    max_attempts: 2
    backoff_ms: 200
```

## 5. 路由伪代码（带函数级注释）

```python
def route_request(ctx, candidates, policy):
    """
    路由主函数：根据策略在候选模型中选择最优目标，并处理重试与降级。
  
    参数:
      - ctx: 请求上下文（业务重要度、地域、SLA、超时等）
      - candidates: 候选推理端点（带实时指标/健康状况）
      - policy: 路由策略（成本/时延/质量权重、阈值、重试配置）
  
    返回:
      - best_target: 选定的推理端点信息
    """
    # 预过滤（健康、地域、合规模型清单）
    filtered = [c for c in candidates if healthy(c) and region_ok(ctx, c)]
  
    # 评分（可接入多臂赌博/贝叶斯优化/质量回传）
    scored = [(c, score(c, policy)) for c in filtered]
    best = max(scored, key=lambda x: x[1])[0]
  
    # 重试/降级
    for attempt in range(policy.retry.max_attempts):
        resp = call(best, ctx)
        if ok(resp, policy):
            return best
        best = next_best(scored, tried=best)
  
    return fallback_endpoint(ctx)
```

## 6. 可观测与评测

- **埋点**：模型、时延、成本、失败类型、质量分、重试次数
- **监控**：SLA 目标、异常告警、容量/配额
- **评测**：离线基准集 + 在线真实流量回放与对比

## 7. 安全与合规

- **供应商合规域**：数据出境控制、地域限制
- **安全防护**：敏感词/提示注入防护
- **审计与留痕**：完整的请求链路追踪

## 8. 扩展阅读与实践要点

- 在路由策略中加入"预算护栏、模型白名单、流量灰度、优先队列与重试/熔断"，并配合回放体系滚动评估
- 建议将 Prompt/模板版本化，结合提示缓存与结果缓存，降低成本提升稳定性

## 9. 路由目标与核心组件

### 路由目标

- **安全优先**：数据安全与合规性第一
- **性价比最优**：在满足质量前提下优化成本
- **性能稳定**：保证响应时间和可用性
- **系统韧性**：故障自愈和降级能力

### 核心组件

1. **数据分级器**：P0/P1/P2，入路由前脱敏与标注
2. **能力注册表**：可跑模型/上下文/显存/加速库/开销系数/可用率/心跳
3. **策略引擎**：安全级、模型需求、SLA、成本因子，权重+规则混合
4. **队列与调度器**：优先级、限流、超时、重试、熔断、批处理
5. **遥测与反馈**：延迟/成功率/每令牌成本/能耗，周期性回填成本模型
6. **回退与降级**：同层替补→跨层、本地→云；模型降阶、上下文裁剪

## 10. 算力层次与适配策略

- **L0：终端设备**（PC/Mac）：低/中负载对话、轻量 RAG、图片批处理
- **L1：本地服务器**（GPU 工作站）：主力推理、向量化、ASR/TTS、视频处理
- **L2：云端资源**（云 API/云 GPU）：尖峰流量、最新前沿模型、重型多模态/视频生成（P0 禁止）

## 11. 路由评分算法

### 优先级原则

安全 > 能力匹配 > SLA 延迟 > 成本

### 评分公式

core = w_sec * S + w_fit * F + w_sla * L + w_cost * C

- S：数据分级是否满足（不满足为 0）
- F：型号/上下文/显存是否满足（完全匹配=1）
- L：预计延迟（越低越高分）
- C：单位 token 成本与电费（越低越高分）


## 12. 企业级策略配置

```yaml
defaults:
  max_latency_ms: 1500
  retry: 2
  fallback_order: [L0, L1, L2]

security:
  P0: { allow: [L0, L1], forbid: [L2], require_masking: true }
  P1: { allow: [L0, L1], cloud_if_anonymized: true }
  P2: { allow: [L0, L1, L2] }

models:
  gpt-4o-mini:
    need_vram_gb: 8
    min_context: 8k
    prefer_layers: [L0, L1]
  llama3.1-70b-instruct:
    need_vram_gb: 48
    prefer_layers: [L1, L2]
  clip_vision:
    type: "embedding"
    prefer_layers: [L1]

scoring_weights: { security: 0.4, fit: 0.25, latency: 0.2, cost: 0.15 }

queues:
  chat:  { priority: 100, max_concurrency: 50 }
  rag:   { priority: 80,  max_concurrency: 40 }
  media: { priority: 60,  max_concurrency: 16, batch_window_ms: 5000 }
```

## 13. 任务类型路由策略

- **高敏任务（财务/合同）**：强制本地 GPU/Mac Studio，数据不出企业
- **通用文本问答**：本地优先，超载时云溢出（平衡性能与成本）
- **轻量任务**：优先闲置 PC/Mac（节省核心算力）
- **大批量多模态**：本地处理小任务，批量任务云扩容（降低延迟与本地压力）

## 14. 节点心跳与能力注册

```json
{
  "node_id": "pc-amy",
  "layer": "L0",
  "gpus": [{"name":"RTX 4060","vram_gb":8,"free_gb":5}],
  "models": ["llama3-8b","clip_vision"],
  "drivers": {"cuda":"12.4"},
  "thermal": 71,
  "price_factor": 0.4,
  "uptime_s": 86400
}
```

## 15. 路由 API 接口

```yaml
post /inference:
  auth: SSO-JWT
  body:
    task_type: ["chat","rag","embed","media"]
    security_level: ["P0","P1","P2"]
    model_hint: "llama3.1-70b-instruct"
    sla:
      max_latency_ms: 1200
      min_tokens_per_s: 30
    payload: {...}
  response:
    job_id: "uuid"
    route: { layer: "L1", node: "srv-gpu-02" }
```

## 16. 事件总线集成

支持的事件类型：

- `route.decided`：路由决策完成
- `node.offline`：节点离线
- `policy.updated`：策略更新
- `budget.exceeded`：预算超限
- `sla.violation`：SLA 违约

可供 n8n/Dify 订阅处理后续自动化流程。

## 17. 治理与验收标准

### 治理机制

- **策略调参**：动态调整权重/限额/模型白名单
- **数据保鲜**：TTL/撤稿/重嵌入策略
- **预算护栏与降阶**：自动成本控制
- **基线扫描**：依赖与镜像安全扫描

### 验收标准（PoC）

- **P0 合规验证**：合同问答确保数据不出本地
- **弹性扩容测试**：峰值自动云溢出
- **故障恢复测试**：节点失联 30s 内迁移
- **动态配置测试**：策略热更新立即生效
- **性能基准测试**：满足 SLA 要求的响应时间和吞吐量
