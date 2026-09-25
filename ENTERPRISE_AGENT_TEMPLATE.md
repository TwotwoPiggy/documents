# 通用企业级 Agent 架构与开发模板 (Enterprise Agent Template)

> **设计目标**：构建一套“插拔式、多模型、带计费计量、强人机协同”的标准化 Agent 研发底座。  
> 无论是**报价单创建**、**差旅报销**、**IT 工单**还是**采购申请**，任何业务只需实现一个业务插件（提供 Prompt 和业务 Tools），即可 10 分钟快速上线，完全复用多模型路由、会话持久化、Token 计费与流式通信底座。

---

## 一、 整体架构全景图

```mermaid
flowchart TD
    User([前端用户 / 客户端]) -->|1. 自然语言交互 (携带 SessionId, ModelId, BusinessCode)| API[通用统一接入层 GenericAgentController]
    
    subgraph Infrastructure [通用基础设施底座]
        API --> Factory[企业级 Agent 工厂 EnterpriseAgentFactory]
        API --> SessionMgr[会话持久化管理器 ISessionStorage]
        API --> Tracker[Token 用量与成本跟踪器 SessionUsageTracker]
        
        Factory --> Router[多模型路由器 Multi-LLM Keyed Services]
        Router --> GPT4o[Azure OpenAI / GPT-4o]
        Router --> Mini[GPT-4o-mini]
        Router --> DeepSeek[DeepSeek V3 / R1]
        Router --> LocalLLM[内网本地私有模型 Ollama / Qwen]
    end

    subgraph BusinessPlugins [业务插件层 (即插即用)]
        Factory --> PluginRegistry{业务插件注册表}
        PluginRegistry -->|businessCode = 'quotation'| QuotationPlugin[报价单业务插件]
        PluginRegistry -->|businessCode = 'expense'| ExpensePlugin[费用报销业务插件]
        PluginRegistry -->|businessCode = 'ticket'| TicketPlugin[工单流转业务插件]
    end

    subgraph ExistingAPIs [现有业务系统与定价引擎]
        QuotationPlugin -->|Tools 调用| ERP_CRM[现有 ERP / CRM API]
        ExpensePlugin -->|Tools 调用| Finance[现有财务系统 API]
        TicketPlugin -->|Tools 调用| ITSM[现有工单系统 API]
    end

    API -->|2. SSE 打字机流式 / 结构化卡片 / 用量元数据| User
```

---

## 二、 核心架构代码抽象 (C# / .NET)

### 2.1 业务插件契约 (`IAgentBusinessPlugin`)
任何新业务只需实现此接口，实现与业务系统 API 和专属 Prompt 的绑定：

```csharp
using Microsoft.Extensions.AI;

namespace EnterpriseAgent.Core.Plugins;

public interface IAgentBusinessPlugin
{
    /// <summary>业务唯一英文代号，例如 "quotation", "expense", "ticket"</summary>
    string BusinessCode { get; }

    /// <summary>业务中文名称</summary>
    string DisplayName { get; }

    /// <summary>动态生成专属的业务提示词（包含槽位完整性检查、防幻觉、人机确认规则）</summary>
    string GetSystemInstructions(UserContext user);

    /// <summary>返回该业务暴露给 LLM 调用的现有系统 API 工具集合</summary>
    IEnumerable<AITool> GetTools(UserContext user);
}
```

---

### 2.2 统一多模型与 Agent 工厂 (`EnterpriseAgentFactory`)
将选定的底层模型（`IChatClient`）与选定的业务插件组装为标准的 `AIAgent`：

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.DependencyInjection;

namespace EnterpriseAgent.Core.Factory;

public class EnterpriseAgentFactory
{
    private readonly IServiceProvider _serviceProvider;
    private readonly IEnumerable<IAgentBusinessPlugin> _plugins;

    public EnterpriseAgentFactory(IServiceProvider serviceProvider, IEnumerable<IAgentBusinessPlugin> plugins)
    {
        _serviceProvider = serviceProvider;
        _plugins = plugins;
    }

    public AIAgent CreateAgent(string businessCode, string modelId, UserContext user)
    {
        // 1. 定位业务插件
        var plugin = _plugins.FirstOrDefault(p => p.BusinessCode.Equals(businessCode, StringComparison.OrdinalIgnoreCase))
            ?? throw new NotSupportedException($"未注册的业务插件: {businessCode}");

        // 2. 解析多模型提供者（从 Keyed Services 解析，未指定则使用默认模型）
        var chatClient = _serviceProvider.GetKeyedService<IChatClient>(modelId)
            ?? _serviceProvider.GetRequiredKeyedService<IChatClient>("gpt-4o-mini");

        // 3. 构造基础 AIAgent（后期可平滑改为 chatClient.AsHarnessAgent(...)）
        return chatClient.AsAIAgent(new ChatClientAgentOptions
        {
            Name = $"{plugin.BusinessCode}-agent",
            Description = plugin.DisplayName,
            ChatOptions = new ChatOptions
            {
                Instructions = plugin.GetSystemInstructions(user),
                Tools = plugin.GetTools(user).ToList()
            }
        });
    }
}
```

---

### 2.3 全局 Token 用量与成本跟踪器 (`SessionUsageTracker`)
支持多模型计费字典，自动计算每轮对话与整个 Session 的累加开销：

```csharp
using Microsoft.Extensions.AI;

namespace EnterpriseAgent.Core.Metering;

public class SessionUsageTracker
{
    public long TotalInputTokens { get; set; }
    public long TotalOutputTokens { get; set; }
    public long TotalTokens => TotalInputTokens + TotalOutputTokens;
    public decimal TotalCostCny { get; set; }

    public (long turnTokens, decimal turnCost) TrackTurn(UsageDetails? usage, string modelId)
    {
        if (usage == null) return (0, 0);

        long input = usage.InputTokenCount ?? 0;
        long output = usage.OutputTokenCount ?? 0;
        long turnTokens = input + output;

        var (inRate, outRate) = ModelPricingConfig.GetRates(modelId);
        decimal turnCost = (input * inRate) + (output * outRate);

        TotalInputTokens += input;
        TotalOutputTokens += output;
        TotalCostCny += turnCost;

        return (turnTokens, turnCost);
    }
}

public static class ModelPricingConfig
{
    // 价格表：元 / 百万 Token (CNY / 1M Tokens)
    private static readonly Dictionary<string, (decimal InRate, decimal OutRate)> Pricing = new(StringComparer.OrdinalIgnoreCase)
    {
        ["gpt-4o-mini"] = (1.05m / 1_000_000m, 4.20m / 1_000_000m),
        ["gpt-4o"]      = (17.5m / 1_000_000m, 70.0m / 1_000_000m),
        ["deepseek-v3"] = (1.00m / 1_000_000m, 2.00m / 1_000_000m),
        ["local-qwen"]  = (0.00m, 0.00m) // 私有部署算力免费
    };

    public static (decimal InRate, decimal OutRate) GetRates(string modelId) =>
        Pricing.TryGetValue(modelId, out var rates) ? rates : (1.0m / 1_000_000m, 2.0m / 1_000_000m);
}
```

---

### 2.4 通用统一接入 API 控制器 (`GenericAgentController`)

```csharp
using Microsoft.AspNetCore.Mvc;
using EnterpriseAgent.Core.Factory;
using EnterpriseAgent.Core.Metering;
using EnterpriseAgent.Core.Session;

namespace EnterpriseAgent.Api.Controllers;

[ApiController]
[Route("api/agent")]
public class GenericAgentController : ControllerBase
{
    private readonly EnterpriseAgentFactory _agentFactory;
    private readonly ISessionStorage _sessionStorage;

    public GenericAgentController(EnterpriseAgentFactory agentFactory, ISessionStorage sessionStorage)
    {
        _agentFactory = agentFactory;
        _sessionStorage = sessionStorage;
    }

    [HttpPost("{businessCode}/chat")]
    public async Task<IActionResult> Chat([FromRoute] string businessCode, [FromBody] AgentChatRequest req)
    {
        var currentUser = HttpContext.Items["CurrentUser"] as UserContext 
            ?? new UserContext("guest", "default-tenant");

        // 1. 获取持久化 Session
        var session = await _sessionStorage.GetOrCreateSessionAsync(req.SessionId, businessCode, currentUser.UserId);

        // 2. 通过工厂动态生成绑定该业务与模型的 Agent
        var agent = _agentFactory.CreateAgent(businessCode, req.ModelId, currentUser);

        // 3. 运行 Agent
        var response = await agent.RunAsync(req.Message, session);

        // 4. 统计并持久化用量
        var tracker = await _sessionStorage.GetTrackerAsync(req.SessionId);
        var (turnTokens, turnCost) = tracker.TrackTurn(response.Usage, req.ModelId);
        await _sessionStorage.SaveAsync(req.SessionId, session, tracker);

        // 5. 格式化输出
        return Ok(new AgentChatResponse
        {
            SessionId = req.SessionId,
            ReplyText = response.Text,
            CurrentModel = req.ModelId,
            Usage = new UsageSummaryDto
            {
                TurnTokens = turnTokens,
                TurnCostCny = Math.Round(turnCost, 4),
                SessionTotalTokens = tracker.TotalTokens,
                SessionTotalCostCny = Math.Round(tracker.TotalCostCny, 4)
            }
        });
    }
}

public record AgentChatRequest(string SessionId, string Message, string ModelId = "gpt-4o-mini");
public record AgentChatResponse
{
    public string SessionId { get; init; } = "";
    public string ReplyText { get; init; } = "";
    public string CurrentModel { get; init; } = "";
    public UsageSummaryDto? Usage { get; init; }
}
public record UsageSummaryDto
{
    public long TurnTokens { get; init; }
    public decimal TurnCostCny { get; init; }
    public long SessionTotalTokens { get; init; }
    public decimal SessionTotalCostCny { get; init; }
}
public record UserContext(string UserId, string TenantId);
```

---

## 三、 示例：如何用该模板在 10 分钟内接入新业务？

以**“报价单创建业务”**为例，开发人员**只需新建一个类实现 `IAgentBusinessPlugin`**：

```csharp
public class QuotationBusinessPlugin : IAgentBusinessPlugin
{
    private readonly IQuotationApiService _apiService;

    public QuotationBusinessPlugin(IQuotationApiService apiService)
    {
        _apiService = apiService;
    }

    public string BusinessCode => "quotation";
    public string DisplayName => "报价单智能助理";

    public string GetSystemInstructions(UserContext user) => """
        你是现有 ERP 系统的智能报价单助手。
        工作流程规范：
        1. 意图与槽位收集：向用户收集客户名称、产品型号和数量。若缺失必须反问。
        2. 工具调用防幻觉：必须调用现有搜索工具检索有效 ID，严禁自造 SKU。
        3. 计算草稿：调用试算接口由后端定价引擎计算总价与折扣。
        4. 人机确认：以 Markdown 表格呈现报价草稿明细，明确询问用户“确认是否提交？”。
        5. 提交落库：仅在用户明确确认后，方可调用最终提交接口。
        """;

    public IEnumerable<AITool> GetTools(UserContext user)
    {
        var tools = new QuotationSystemTools(_apiService, user);
        return
        [
            AIFunctionFactory.Create(tools.SearchCustomerAsync),
            AIFunctionFactory.Create(tools.SearchProductsAsync),
            AIFunctionFactory.Create(tools.PreviewQuotationDraftAsync),
            AIFunctionFactory.Create(tools.SubmitQuotationAsync)
        ];
    }
}
```

同理，若接入**“差旅报销业务”**，只需新建 `ExpenseBusinessPlugin`，配置财务预算检查和报销单创建接口，其他所有通用功能（API、会话、前端展示、计费、多模型）即可全部无缝复用！

---

## 四、 升级演进：平滑过渡到 HarnessAgent

当某个业务（例如大宗项目采购询价）从简单的 2~3 轮问答升级为包含“30 页 PDF 解析”、“长任务 Todo 拆解”、“上下文自动压缩”、“特价审批挂起”时，只需在 `EnterpriseAgentFactory` 中将：
```csharp
chatClient.AsAIAgent(...)
```
替换为：
```csharp
chatClient.AsHarnessAgent(new HarnessAgentOptions { ... })
```
所有的业务插件（`IAgentBusinessPlugin`）和业务 Tools 代码**一行不用改**，直接升级至工业级长任务脚手架！
