# LIMS 系统智能化转型：MCP 接入与 Agent 架构设计规划

## 1. 项目背景与战略目标

*   **业务领域**：检验检测实验室管理系统 (LIMS)，涵盖 9 大核心模块（客户管理、销售管理、申请单、报价单、订单、分包单、接包单、报告模块、样品管理）。
*   **转型战略**：不进行大规模底层重构，采用**“核心系统 + MCP (Model Context Protocol) 适配层”**的非侵入式架构。
*   **最终目标**：允许用户通过前端 Agent Desktop（如 Claude Desktop 等）以自然语言与 LIMS 系统交互，实现复杂数据的跨模块检索、智能组装与业务统计，极大降低数据获取门槛。

---

## 2. 核心高优智能化场景 (MVP 候选)

在初期试点阶段，应聚焦于解决传统系统因“页面固化”导致的查询痛点。

### 场景 A：跨模块连环追查（全天候进度管家）
*   **痛点**：客户询问测试进度时，客服需在“订单 -> 样品 -> 分包单 -> 报告”等多个模块间频繁切换拼凑信息。
*   **Agent 能力**：用户输入自然语言指令（如“查一下 XX 公司上周单子的整体进度”），Agent 后台并发调用多个 MCP 接口，整合返回一份包含溯源链路的进度汇总报告。

### 场景 B：基于语义的标准匹配与估价（智能售前辅助）
*   **痛点**：销售面临非技术性客户需求（如“保温杯出口欧洲”）时，难以在传统系统的“关键字查询”下准确找出对应的测试标准并报价。
*   **Agent 能力**：结合静态标准库资源 (Resources) 和动态查价工具 (Tools)，Agent 能理解语义意图，推荐合适的测试套餐并拉取系统单价。

### 场景 C：管理层异常穿透分析（动态统计工具）
*   **痛点**：固定报表无法满足管理层临时性的聚合分析需求（如“分析上周各测试项目的延误率”），依赖 IT 部门提数。
*   **Agent 能力**：大模型结合允许动态聚合的底层查询接口，自动拉取异常数据，清洗并在对话框中直接生成图表与归因结论。

---

## 3. 架构演进路线 (Phased Approach)

避免“大爆炸”式重构，分步实施：

*   **Phase 1：低成本“代理模式” (As-Is Proxy)**
    保持现有 Service 层不动。新增一层轻量的 MCP Server，将现有的（UI驱动的）固定查询接口包装成 MCP Tools 暴露。解决 60% 的基础查询需求。
*   **Phase 2：“Agent-Native” 动态查询层 (Dynamic Query Service)**
    针对 Agent 灵活发散的查询需求，在底层新建专为 Agent 优化的查询接口，接收标准化的 JSON 结构体，支持多维度过滤拼装。
*   **Phase 3：引入向量语义检索（远期进阶）**
    针对国家标准文档、历史报价 PDF 等非结构化知识，引入向量数据库（如 Milvus / pgvector）与 RAG 架构。

---

## 4. 后端技术设计规范 (基于 MySQL + EF Core)

### 4.1 动态查询数据契约 (DTO)
前后端必须通过标准化的结构体通讯，禁止大模型直接写原生的 SQL 发给后端。

```csharp
// Agent 组装并发送的查询契约
public class DynamicQueryRequest
{
    public List<FilterCondition> Filters { get; set; } = new List<FilterCondition>();
    public int Limit { get; set; } = 20; // 强制兜底分页，防止大模型 Context 爆炸
}

public class FilterCondition
{
    public string Field { get; set; }    // 字段名
    public string Operator { get; set; } // 操作符 (==, Contains, >, <, IN)
    public string Value { get; set; }    // 查询值
}
```

### 4.2 EF Core 后端实现（优雅的白名单映射）
严禁拼接原始 SQL。必须利用 EF Core 的 `IQueryable` 进行安全的参数化查询。建议采用**字典映射（策略模式）**替代繁琐的 `switch-case`。

```csharp
public async Task<List<OrderDTO>> GetOrdersDynamicAsync(DynamicQueryRequest request)
{
    var query = _dbContext.Orders.AsQueryable();

    // 1. 定义白名单与对应的 EF Core 表达式构建策略
    var filterStrategies = new Dictionary<string, Func<IQueryable<Order>, FilterCondition, IQueryable<Order>>>(StringComparer.OrdinalIgnoreCase)
    {
        { "CustomerName", (q, f) => f.Operator == "Contains" ? q.Where(x => x.CustomerName.Contains(f.Value)) : q.Where(x => x.CustomerName == f.Value) },
        { "Status", (q, f) => q.Where(x => x.Status == f.Value) }
        // 只有明确注册在这里的字段，Agent 才能查询
    };

    // 2. 遍历 Agent 传来的条件，动态挂载
    foreach (var filter in request.Filters)
    {
        if (filterStrategies.TryGetValue(filter.Field, out var strategy))
        {
            query = strategy(query, filter);
        }
    }
    return await query.Take(request.Limit).ToListAsync();
}
```

---

## 5. 架构设计核心 Q&A (关键决策记录)

本章节记录了在架构设计初期的核心战略与技术疑虑，作为后续研发的重要指导思想。

### Q1：AI 应用发展处于初期，如果全面转向 Agent Desktop (接入 MCP)，未来有了更好的前沿应用不再使用 MCP，现在的投资是否会打水漂？
**决策记录：不会浪费，反而是极具价值的 API 资产沉淀。**
*   **业务解耦**：开发 MCP Server 的核心工作，90% 是在梳理底层业务逻辑、编写数据库查询 API。仅有 10% 是协议包装。
*   **范式不变**：无论未来协议如何变迁，AI 依托“Tool Calling (工具调用)”与外部系统交互的核心范式不会改变。
*   **架构防御**：采用“核心系统 -> REST API -> MCP Adapter”的三层架构。即便 MCP 淘汰，底层沉淀的强大 API 资产依然可对接任何新形态的 AI 平台。

### Q2：什么是“语义检索工具”？传统 API 为什么做不到？
**决策记录：传统系统依赖字面匹配，语义检索依赖意图匹配。**
*   **传统痛点**：在标准库中搜索“保温杯”，如果正式文件叫“食品接触用不锈钢容器”，传统 SQL 查询会返回 0 条结果。
*   **语义检索**：依托向量数据库和大模型 Embeddings 技术，AI 能理解“保温杯”和“不锈钢容器”在概念上的关联，从而实现基于自然语言意图的精准检索。这在处理非结构化专业知识（如检验标准）时不可或缺。

### Q3：如果现有 API 查询字段固定，能否让 Agent 先把全量数据加载到“内存（上下文）”中，再自己进行条件过滤？
**决策记录：坚决禁止。必须在数据库层完成过滤。**
*   **Context 爆炸**：大量冗余数据会瞬间撑爆大模型的上下文窗口，导致 API 费用飙升或直接崩溃。
*   **准确率下降**：大模型面对海量数据时容易产生“大海捞针”综合征，漏看或幻觉率极高。
*   **正确做法**：赋予 Agent 组装动态查询条件的能力（见上文 4.1 DTO），将运算和过滤压力下沉到后端的 MySQL 数据库中，仅向 Agent 返回精准的小批量结果。

### Q4：在 .NET EF Core 环境下做动态查询，如何防范 SQL 注入？
**决策记录：使用 `IQueryable` 延迟查询机制。**
*   绝对禁止通过字符串插值直接拼接原生 SQL（如 `ExecuteSqlRaw` 拼接参数）。
*   只要使用 EF Core 的 LINQ 特性（如 `.Where(o => o.Name == input)`）或使用安全的扩展库 `System.Linq.Dynamic.Core`，底层都会自动将其翻译为带有 `@p0` 标志的**参数化查询 (Parameterized Queries)**，从根本上杜绝 SQL 注入漏洞。

### Q5：为了支持动态查询，为什么必须使用“白名单”，而不是更灵活的“黑名单”（如利用反射排除敏感字段即可）？
**决策记录：在面向 AI 暴露接口时，安全与可控永远优先于代码灵活性。必须采用白名单。**
*   **防破窗 (Fail-Safe)**：未来新增的敏感表字段（如利润率），如果忘了加入黑名单，会立刻向 AI 暴露。白名单则默认拒绝所有未知字段。
*   **Prompt Token 优化**：使用白名单可以在 MCP 描述中清晰告诉大模型“只有这 5 个字段可用”，大模型按图索骥。如果用黑名单，难道要将表里剩余的 150 个字段全部告诉大模型？这会造成 Context 浪费和严重幻觉。
*   **防止慢查询拖垮 DB**：白名单强制研发人员仅暴露出**已建立数据库索引**的字段。防止大模型胡乱组合出类似长文本模糊搜索的恶意查询，导致 MySQL 生产库全表扫描而宕机。
