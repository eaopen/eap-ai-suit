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

## 2.1 二级路由架构设计

企业级智能路由采用二级路由实现从资源发现到模型执行的全链路优化：

- 一级路由（资源/工具路由）：在工具/资源层进行选择，例如通过 MCP 调用本地“企业微信记录”连接器，或改为云端 API。
- 二级路由（模型/算力路由）：在具体算力和模型层面进行调度，例如选择本地 GPU、边缘 NPU，或云端推理服务。

路由流程

1) 请求解析 → 识别任务类型、安全等级、资源需求
2) 一级路由 → 工具链与数据源选择（本地 MCP / 云 API / 混合）
3) 二级路由 → 模型与算力选择（本地/云、具体模型）
4) 执行与监控 → 全链路追踪、反馈回传、异常处理

术语与边界

- 本地客户端路由：发生在终端设备内部的轻量智能路由，支持离线策略与边缘推理。
- 工具链路由：针对 MCP/插件等工具调用的路由策略，需合规优先与数据驻留优先。
- 混合路由：一级在本地、二级在云端或相反，需满足数据合规前提。

## 2.2 本地客户端智能路由

组件架构

- 本地路由引擎：缓存策略、离线可用、轻量评分与降级
- 资源感知：CPU/GPU/NPU/内存/温度与电源模式监控
- 安全与隐私控制：数据驻留、脱敏、最小化数据共享
- 云端协作：策略同步、匿名化指标上报、云溢出通道

工作模式

- 完全离线：基于本地缓存策略决策，适用于 P0/P1 高敏数据
- 混合模式：本地优先，必要时二级路由到云端
- 云端辅助：云端提供新策略/新模型可用性与健康状态

## 2.3 MCP 获取企业微信记录场景

业务说明
当助手需要访问企业微信聊天记录进行分析时，涉及高敏信息，需优先保障合规与数据驻留。

端到端流程

1) 用户请求解析 → 识别需要微信数据
2) 一级路由（资源层）→ 选择本地 MCP 工具（企业微信连接器），数据不出设备
3) 数据获取与脱敏 → 本地完成 PII/敏感字段脱敏
4) 二级路由（模型层）→ 在本地选择合规模型分析（本地 GPU/NPU 或轻量 CPU 模型）
5) 结果处置 → 输出仅含必要洞察，确保隐私合规

安全与回退

- 数据驻留：强制本地处理，禁云；内存加密、及时清理
- 访问控制：角色/会话授权、完整审计
- 回退：工具不可用则提示用户提供摘要；本地算力不足降级到轻量模型/分片处理；若仍不满足合规则拒绝执行

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
def emit(event_name: str, payload: dict):
    """
    事件上报函数：用于将路由与执行阶段的关键事件发送到事件总线。
    参数:
      - event_name: 事件名（如 'tool.route.started', 'route.subdecided', 'route.decided'）
      - payload: 事件携带的上下文信息
    返回:
      - None
    """
    pass


def route_resource(ctx, tools, policy):
    """
    一级路由（资源/工具路由）：根据任务与合规需求选择工具链与数据源。
    参数:
      - ctx: 请求上下文（task_type、security_level、need_wechat 等）
      - tools: 可用工具列表（本地 MCP 工具、云端 API 工具等，含健康/合规标记）
      - policy: 策略（合规优先级、数据驻留要求、工具白名单、重试配置）
    返回:
      - selected_tool: 被选中的工具（优先本地/合规），或 None 表示无需工具
    """
    candidates = [t for t in tools if t.healthy and t.allowed_for(ctx.security_level)]
    # 合规优先：P0/P1 强制本地驻留工具，例如本地 MCP 企业微信
    if ctx.need_wechat:
        local_wechat_tools = [t for t in candidates if t.name == "mcp_wechat_local"]
        if ctx.security_level in ("P0", "P1") and local_wechat_tools:
            return local_wechat_tools[0]
        # 不满足合规或本地工具不可用，触发回退与告警
        if ctx.security_level in ("P0", "P1"):
            raise RuntimeError("合规限制：P0/P1 场景禁止云端微信数据访问")
    # 其他工具按健康度/成本/延迟简单加权选择
    if candidates:
        return max(candidates, key=lambda t: t.score(policy))
    return None


def route_model(ctx, candidates, policy, scope=None):
    """
    二级路由（模型/算力路由）：在允许的算力与模型范围内进行精确选择与重试/降级。
    参数:
      - ctx: 请求上下文（SLA、时延预算、模型提示）
      - candidates: 可用推理端点/模型（带实时指标/健康状况/成本）
      - policy: 路由策略（权重、阈值、重试与降级配置）
      - scope: 可选的资源范围约束（如要求 L0/L1、本地-only）
    返回:
      - best_target: 选定的推理端点
    """
    # 预过滤：健康、地域、合规与层级（如 P0 禁止 L2 云）
    def allowed_layer(c):
        if ctx.security_level == "P0":
            return c.layer in ("L0", "L1")
        return True

    filtered = [c for c in candidates if c.healthy and allowed_layer(c)]
    if scope == "local_only":
        filtered = [c for c in filtered if c.layer in ("L0", "L1")]

    if not filtered:
        raise RuntimeError("无可用候选模型（已被合规或健康检查过滤）")

    # 评分（可接入多臂赌博/质量回传），示例：安全>匹配>SLA>成本
    def score(c):
        s = c.security_fit(ctx) * policy.weights["security"]
        f = c.capability_fit(ctx) * policy.weights["fit"]
        l = c.latency_fit(ctx) * policy.weights["latency"]
        k = c.cost_fit(ctx)    * policy.weights["cost"]
        return s + f + l + k

    scored = sorted(filtered, key=lambda x: score(x), reverse=True)

    # 重试与降级
    attempts = 0
    tried = set()
    while attempts <= policy.retry.max_attempts and scored:
        target = next(c for c in scored if c.id not in tried)
        resp = call_model(target, ctx)
        if ok(resp, policy):
            return target
        tried.add(target.id)
        attempts += 1

    # 降级策略：模型降阶/上下文裁剪/切换层级
    fallback = fallback_endpoint(ctx)
    return fallback


def route_request(ctx, tools, models, policy):
    """
    路由主函数：执行二级路由（先资源/工具，再模型/算力），并处理埋点与降级。
    参数:
      - ctx: 请求上下文（task_type、security_level、need_wechat、SLA 等）
      - tools: 工具列表（含 MCP 等）
      - models: 模型/算力候选列表（L0/L1/L2）
      - policy: 路由策略
    返回:
      - decision: { tool, model } 最终决策
    """
    # 一级：资源/工具路由
    emit("tool.route.started", {"ctx": ctx.to_dict()})
    tool = route_resource(ctx, tools, policy)
    emit("route.subdecided", {"tool": getattr(tool, "name", None)})

    # 工具执行（如需）：例如本地 MCP 获取企业微信记录并在本地脱敏
    if tool:
        data = call_tool(tool, ctx)
        ctx.attach_tool_result(data)

    # 二级：模型/算力路由（若工具涉及高敏数据则限制为本地范围）
    scope = "local_only" if (ctx.need_wechat and ctx.security_level in ("P0", "P1")) else None
    model = route_model(ctx, models, policy, scope=scope)

    emit("route.decided", {"tool": getattr(tool, "name", None), "model": model.id})
    return {"tool": tool, "model": model}
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
  "models": ["qwen3-8b","clip_vision"],
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
    model_hint: "qwen3-72b"
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

- `tool.route.started`：开始进行一级工具/资源路由（便于对接自动化与审计）
- `route.subdecided`：一级路由（资源/工具）决策完成
- `route.decided`：二级路由（模型/算力）决策完成
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

## 18. 典型应用案例

### 18.1 案例 A：传统企业本地化部署方案

compute_layers:
  L0: # 终端设备
    - type: "workstation"
      gpu: "RTX 4060/4070"
      models: ["gpt-oss-20b", "qwen3-4b"]
      role: "轻量推理、预处理"

  L1: # 本地服务器
    primary_inference:
      - gpu: "H20"
        models: ["gpt-oss-120b", "qwen3-72b", "deepseek-r1-67b"]
        role: "主力推理服务"
    cost_effective:
      - gpu: "A30"
        models: ["gpt-oss-20b", "qwen3-32b", "glm-4.5-9b"]
        role: "批量处理、轻量任务"

# 案例B：IT中小企业混合部署方案

compute_layers:
  L0: # 终端设备（MacBook Pro）
    - type: "macbook_pro_m3"
      memory: "36GB/128GB"
      models: ["gpt-oss-20b", "qwen3-4b"]
      role: "个人助手、代码补全、轻量推理"

  L1: # 本地服务器（Mac Studio）
    - type: "mac_studio_m3_ultra"
      memory: "512GB"
      models: ["gpt-oss-120b", "qwen3-72b", "deepseek-r1-67b"]
      role: "团队共享推理、重型任务"

  L2: # 云端服务（选择性使用）
    - providers: ["openai", "anthropic", "azure"]
      models: ["gpt-4o", "claude-4-sonnet", "gpt-oss-120b"]
      role: "尖峰负载、最新模型"

routing_policy:
  data_classification:
    P0: "local_only"  # 客户核心代码
    P1: "local_preferred"  # 内部文档
    P2: "cloud_allowed"  # 通用任务

  cost_optimization:
    daily_hours: "9-18 local_preferred"
    off_hours: "cloud_spillover"
    weekend: "local_only"

```

**智能路由策略**：
```python
def route_for_it_company(ctx, policy):
    """
    IT中小企业智能路由策略：平衡安全、性能和成本
    参数:
      - ctx: 请求上下文（包含数据分级、时间等信息）
      - policy: 路由策略配置
    返回:
      - route_decision: 路由决策结果
    """
    # 数据分级判断
    if ctx.contains_client_code or ctx.security_level == "P0":
        # 客户代码强制本地处理
        return route_local_only(ctx, prefer_mac_studio=True)
  
    # 工作时间优先本地，节省成本
    if is_business_hours() and ctx.security_level == "P1":
        local_capacity = check_local_capacity()
        if local_capacity > 0.7:  # 本地资源充足
            return route_local(ctx)
        else:
            return route_hybrid(ctx, local_ratio=0.6)
  
    # 非工作时间或P2数据可以云端处理
    if ctx.security_level == "P2" or not is_business_hours():
        return route_cloud_preferred(ctx, cost_limit=True)
  
    return route_local(ctx)  # 默认本地
```

**应用场景**：

- 代码审查和重构建议（本地）
- 技术文档生成（混合）
- 客户需求分析（本地）
- 市场调研和内容创作（云端）
- 团队协作和项目管理（混合）

**成本效益**：

- 初期投资：25-35万（Mac Studio + 终端升级）
- 月运营成本：5000-8000元（电费+云服务）
- 灵活性：根据项目需求动态调整
- 开发效率提升：30-50%

### 18.3 案例对比总结

| 维度               | 传统企业方案     | IT中小企业方案     |
| ------------------ | ---------------- | ------------------ |
| **安全等级** | P0严格本地化     | P0/P1/P2分级处理   |
| **硬件投资** | 80-120万         | 25-35万            |
| **运营成本** | 15-25万/年       | 6-10万/年          |
| **灵活性**   | 低，稳定优先     | 高，敏捷优先       |
| **技术门槛** | 中等，需专业运维 | 低，开发团队可维护 |
| **扩展性**   | 硬件扩展         | 云端弹性扩展       |
| **适用规模** | 大中型企业       | 中小型企业/工作室  |

### 18.4 方案选择指导

**选择传统企业方案的条件**：

- 年营收超过10亿，有充足IT预算
- 处理高度敏感数据（金融、医疗、军工等）
- 有专业IT运维团队
- 对数据出境有严格限制
- 追求长期稳定性和可控性

**选择IT中小企业方案的条件**：

- 团队规模50-200人
- 技术团队具备一定运维能力
- 业务数据敏感度分级明确
- 追求成本效益和部署灵活性
- 可接受部分云服务依赖

这两个典型案例展示了智能路由方案如何根据不同企业的安全需求、技术能力和成本预算进行灵活适配，实现从完全本地化到混合部署的全覆盖，为中型企业提供了简化而实用的AI算力解决方案。
