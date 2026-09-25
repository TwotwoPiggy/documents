# 智能报价单 Agent 系统开发计划 (Development & Execution Plan)

> **文档定位**：本文档为“智能报价单创建 Agent”的端到端技术实现蓝图。  
> **使用说明**：文档中涉及现有系统具体代码、数据库和 API 的部分已使用 `[TODO: 规划 Agent 填充]` 规范占位。**负责规划的 Agent** 应优先扫描现有代码库补齐具体上下文，形成终版计划后交由**执行 Agent** 落地。

---

## 一、 项目背景与架构蓝图

### 1.1 核心目标
将现有系统中繁琐的表单录入式报价单创建流程，升级为**“自然语言对话驱动”**的智能化流程。用户只需在对话界面以人话提出需求，Agent 负责意图理解、槽位补齐、调用现有系统计算草稿，并在用户确认后正式落库。

### 1.2 核心技术特性
1. **现有系统零破坏**：不重构现有业务逻辑，将现有 API 封装为 Agent 标准工具（Tools）。
2. **多模型自由切换 (Multi-LLM Switcher)**：前端支持动态切换模型（轻量/高智商/内网私有），切换过程会话上下文与草稿不丢失。
3. **精准用量与计费统计 (Token & Cost Metering)**：实时按轮次、按会话统计 Token 消耗量与预估金额，并在前端展示。
4. **强确定性与人机确认 (Human-in-the-Loop)**：严禁模型自主定价，金额由现有系统定价引擎计算；生成正式单据前必须经过用户显式确认。
5. **架构平滑演进**：第一阶段基于 `AIAgent + Tools + AgentSession` 敏捷落地；底层保留扩展接口，未来可无缝平移至 `HarnessAgent` 处理复杂长任务。

---

## 二、 现有系统上下文占位表 (规划 Agent 填槽区)

> **👉 请负责规划的 Agent 深入当前代码库，检索并填充以下板块：**

### 2.1 现有系统技术栈与环境
- **后端运行时与框架**：`[TODO: 规划 Agent 填充: 如 .NET 8 / .NET 9 Web API]`
- **数据访问层**：`[TODO: 规划 Agent 填充: 如 EF Core / Dapper / 现有仓储接口]`
- **用户身份与鉴权机制**：`[TODO: 规划 Agent 填充: 如 JWT / ClaimsPrincipal / CurrentUserContext 提取方式]`
- **缓存与状态存储**：`[TODO: 规划 Agent 填充: 如 Redis / MemoryCache / 数据库会话表]`

### 2.2 现有报价单业务接口与模型映射
- **客户检索接口**：
  - 现有服务/方法：`[TODO: 规划 Agent 填充: 如 ICustomerService.SearchAsync(string keyword)]`
  - 核心字段：`[TODO: 规划 Agent 填充: CustomerId, CustomerName, CreditLimit, DefaultDiscount]`
- **产品与库存检索接口**：
  - 现有服务/方法：`[TODO: 规划 Agent 填充: IProductService.SearchProductsAsync(...)]`
  - 核心字段：`[TODO: 规划 Agent 填充: SkuId, SkuCode, Name, Spec, BasePrice, AvailableStock]`
- **定价引擎与草稿试算接口**：
  - 现有服务/方法：`[TODO: 规划 Agent 填充: IQuotationCalculationService.CalculateAsync(...)]`
  - 试算输入：`[TODO: 规划 Agent 填充: CustomerId, List<ItemInput> (SkuId, Quantity)]`
  - 试算产出：`[TODO: 规划 Agent 填充: DraftId, TotalAmount, TaxAmount, DiscountAmount, ExpireTime]`
- **正式报价单创建与落库接口**：
  - 现有服务/方法：`[TODO: 规划 Agent 填充: IQuotationService.CreateFromDraftAsync(string draftId)]`
  - 产出结果：`[TODO: 规划 Agent 填充: QuotationId, QuotationNo, Status, DetailUrl]`

---

## 三、 分阶段实施路线图 (Phase-based Roadmap)

### Phase 1：业务 API 工具化适配层 (Tooling Layer)
*目标：将现有报价单核心能力封装为标准 `AIFunction`，确保模型调用的类型安全与防幻觉。*

- [ ] **Task 1.1: 定义工具交互 DTO 契约**
  - 定义给 Agent 使用的入参和出参模型（精简非必要冗余字段，优化 Token 开销）。
- [ ] **Task 1.2: 编写 `QuotationSystemTools` 服务类**
  - 封装四个核心方法：
    1. `SearchCustomerAsync(string keyword)`
    2. `SearchProductsAsync(string keyword)`
    3. `PreviewQuotationDraftAsync(string customerId, List<ItemDto> items, string? notes)`
    4. `SubmitQuotationAsync(string draftId)`
  - 为每个方法和参数打上详尽清晰的 `[Description]` 元数据注解。
- [ ] **Task 1.3: 防幻觉与业务守则埋点**
  - 在 `PreviewQuotationDraftAsync` 中严格使用现有系统定价引擎，严禁模型自主加减乘除计算。
  - 为 `draftId` 注入防重复提交机制（幂等性 Token）。

### Phase 2：Agent 核心中枢与多模型接入架构 (Agent Core & Multi-LLM)
*目标：建立统一的 Agent 工厂，接入多种底层大模型，支持动态无感切换。*

- [ ] **Task 2.1: 注册多模型提供者 (`IChatClient`)**
  - 在 DI 中使用 .NET Keyed Services 注册不同模型：
    - `gpt-4o-mini`（通用轻量）
    - `gpt-4o`（复杂高精）
    - `deepseek-v3`（超高性价比）
    - `local-model`（可选，内网私有化模型）
- [ ] **Task 2.2: 编写业务系统提示词 (Prompt Engineering)**
  - 确立 Agent 行为守则：
    - 槽位不齐必须主动追问（客户、型号、数量为硬性前置条件）。
    - 检索到多结果时列举供用户挑选，禁止擅自猜测。
    - 调完试算草稿后，必须输出结构化 Markdown 表格并触发**人机确认**。
    - 未获得用户明确确认前，禁止调用 `SubmitQuotationAsync`。
- [ ] **Task 2.3: 实现 `QuotationAgentFactory`**
  - 统一组装 `IChatClient` + 共享的 `ChatOptions`（Instructions + Tools）。

### Phase 3：会话状态管理与用量计费监控 (Session & Metering)
*目标：实现跨轮次上下文连续性，并在模型切换时无缝继承状态；精准统计用量和成本。*

- [ ] **Task 3.1: 会话持久化适配器 (`AgentSession`)**
  - 建立会话与当前用户的绑定关系。
  - 实现基于 Redis 或数据库的 `AgentSession` 存储机制，保障多实例部署下的状态共享。
- [ ] **Task 3.2: Token 消耗与价格计算引擎 (`SessionUsageTracker`)**
  - 配置各模型的输入/输出阶梯定价字典（汇率按人民币或美元换算）。
  - 每次 Agent 回复后解析 `response.Usage`，支持跨多步工具调用的 Token 汇总累加。
- [ ] **Task 3.3: 跨模型切换会话无缝继承验证**
  - 验证用户在中途切换模型后，上下文历史与当前已生成的 `DraftId` 不丢失。

### Phase 4：API 暴露与前端交互设计 (API & UI Integration)
*目标：打通前后端数据流，提供流式响应、用量挂牌展示与草稿确认卡片。*

- [ ] **Task 4.1: 设计 Web API 接口**
  - 路径：`POST /api/quotation-agent/chat`
  - 协议支持：普通 JSON 响应及 SSE（Server-Sent Events）流式响应。
  - 入参：`{ sessionId, message, modelId }`
  - 出参：`{ replyText, modelId, usage: { turnTokens, sessionTotalTokens, sessionTotalCost } }`
- [ ] **Task 4.2: 前端组件集成**
  - 顶部模型切换下拉框（实时生效）。
  - 顶部/侧边栏 Token 与估算金额动态计数牌。
  - 报价草稿结构化卡片渲染（含 `[确认提交]` 与 `[修改]` 快捷操作按钮）。

### Phase 5：测试、风控与质量门禁 (Verification & Quality Gates)
*目标：确保业务安全合规，防止越权、幻觉与重复提交。*

- [ ] **Task 5.1: 槽位抽取与缺漏追问测试**（测试模糊表述、缺数量、缺客户名称时的多轮纠偏能力）。
- [ ] **Task 5.2: 严苛定价防幻觉测试**（比对 Agent 最终出具草稿与现有 ERP 计算结果，误差必须为 0）。
- [ ] **Task 5.3: 权限隔离测试**（验证销售 A 无法搜出销售 B 的客户与底价折扣）。
- [ ] **Task 5.4: 幂等与防重测试**（高频连续点击确认提交，确保只生成单张正式报价单）。

---

## 四、 给规划 Agent 与执行 Agent 的协作指南

### 4.1 规划 Agent (Planning Agent) 的行动步骤：
1. **代码扫描**：在现有工程中定位所有与报价单相关的 Service、DTO、Controller 及数据库实体。
2. **填补本文档第二章的占位槽**：将确切的类名、方法签名、参数结构替换掉所有的 `[TODO: ...]`。
3. **细化实现依赖**：确认当前项目引用的包版本（如 `Microsoft.Agents.AI`、`Microsoft.Extensions.AI` 等），若缺包则在 Phase 1 补充安装步骤。
4. **拆解落地任务清单**：生成详细的施工工单（如每个文件的路径与新增/修改职责）。

### 4.2 执行 Agent (Implementation Agent) 的执行守则：
1. **优先复用原则**：严格调用规划 Agent 整理出的现有系统底层接口，严禁在 Agent 层自建一套报价逻辑或直接裸写 SQL 改动单据。
2. **严守数据防线**：任何涉及价格、折扣的数值展示，数据源头必须 100% 来自现有定价 API 返回的字段。
3. **完成一项打勾一项**：对照 Phase 1 到 Phase 5 的任务列表逐步实施，每阶段完成后运行单元测试与端到端模拟链路。
