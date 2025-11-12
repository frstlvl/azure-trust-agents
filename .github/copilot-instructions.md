# Azure Trust Agents - AI Agent Development Guide

## Overview

This is a **Microsoft Agent Framework** hackathon project that builds an enterprise-grade fraud detection system through 4 progressive challenges. The system uses a 4-agent sequential workflow with Azure AI Foundry integration, Model Context Protocol (MCP) servers, and comprehensive OpenTelemetry observability.

**Architecture Pattern**: Customer Data Agent → Risk Analyzer Agent → (Compliance Report Agent ∥ Fraud Alert Agent)

## Core Framework: Microsoft Agent Framework

All agent development uses the **Microsoft Agent Framework** (released October 2025), which unifies Semantic Kernel with AutoGen orchestration.

### Key Framework Patterns

**Executor Pattern** - All workflow steps use the `@executor` decorator:

```python
from agent_framework import executor, WorkflowContext

@executor
async def customer_data_executor(
    request: AnalysisRequest,
    ctx: WorkflowContext
) -> CustomerDataResponse:
    # Fetch data, process, return typed response
    result = CustomerDataResponse(...)
    await ctx.send_message(result)  # Pass to next executor
    return result
```

**Sequential Orchestration** - Build workflows with `WorkflowBuilder`:

```python
from agent_framework import WorkflowBuilder

workflow = (
    WorkflowBuilder()
    .add_edge("start", "customer_data")
    .add_edge("customer_data", "risk_analysis")
    .add_edge("risk_analysis", "compliance_report")
    .build()
)
```

**Azure AI Agent Integration** - Wrap Foundry agents with `ChatAgent`:

```python
from agent_framework import ChatAgent
from agent_framework.azure import AzureAIAgentClient

agent = ChatAgent(
    chat_client=AzureAIAgentClient(
        project_client=project_client,
        agent_id=os.environ["RISK_ANALYSER_AGENT_ID"]  # From .env
    ),
    tools=[get_customer_data, get_transactions],
    store=True
)
```

### Typed Message Contracts

ALL workflow messages use **Pydantic BaseModel** for type safety:

```python
class RiskAnalysisResponse(BaseModel):
    risk_analysis: str
    risk_score: str
    transaction_id: str
    status: str
    risk_factors: list = []
```

## Essential Development Workflows

### 1. Environment Setup (Always First!)

```bash
cd challenge-0

# Login to Azure CLI (required for authentication)
az login --use-device-code

# Auto-populate .env from Azure resources
./get-keys.sh --resource-group YOUR_RESOURCE_GROUP_NAME

# Verify .env has all keys from .env.sample
cat ../.env
```

**Critical**: ALL Azure resources must be pre-deployed via ARM template (`challenge-0/infra/azuredeploy.json`). The `get-keys.sh` script extracts credentials automatically.

### 2. Agent Creation & Registration

**Step 1**: Create agent in Azure AI Foundry:

```python
# See challenge-1/agents/customer_data_agent.py
created_agent = await project_client.agents.create_agent(
    model=model_deployment_name,
    name="CustomerDataAgent",
    instructions="..."  # System prompt
)
print(f"Agent ID: {created_agent.id}")  # Copy this!
```

**Step 2**: Add agent ID to `.env`:

```bash
CUSTOMER_DATA_AGENT_ID=asst_XXXXXXXXXXXXXXXXXXXXXXXX
RISK_ANALYSER_AGENT_ID=asst_YYYYYYYYYYYYYYYYYYYYYYYY
COMPLIANCE_REPORT_AGENT_ID=asst_ZZZZZZZZZZZZZZZZZZZZZZ
```

**Step 3**: Reference in workflow via environment variable (never hardcode IDs).

### 3. Local Agent Testing with DevUI

```bash
cd challenge-1/devui

# Test all agents and workflows in interactive UI
python devui_launcher.py --mode all           # All entities
python devui_launcher.py --mode directory     # Discover from filesystem
python devui_launcher.py --mode agents        # Agents only
python devui_launcher.py --mode workflow      # Workflow only
```

Access at `http://localhost:8000` for visual agent testing.

### 4. Running Workflows

```bash
# Jupyter notebook (interactive)
cd challenge-1/workflow
jupyter notebook sequential_workflow.ipynb

# Python script (production)
python sequential_workflow.py
```

Workflows analyze transaction `TX2002` by default (configurable in `AnalysisRequest`).

## Data Layer Architecture

### Cosmos DB Structure

**Database**: `FinancialComplianceDB`
**Containers**:

- `Customers` - Customer profiles with fraud history, device trust scores
- `Transactions` - Transaction records with amounts, countries, timestamps

**Query Pattern** (always enable cross-partition):

```python
from azure.cosmos import CosmosClient

cosmos_client = CosmosClient(cosmos_endpoint, cosmos_key)
database = cosmos_client.get_database_client("FinancialComplianceDB")
container = database.get_container_client("Customers")

query = f"SELECT * FROM c WHERE c.customer_id = '{customer_id}'"
items = list(container.query_items(
    query=query,
    enable_cross_partition_query=True  # Always required!
))
```

### Azure AI Search Integration

**Index**: `regulations-policies` (regulations and compliance rules)

**Integration Pattern** (Azure AI Foundry HostedFileSearchTool):

```python
# Risk Analyzer Agent uses built-in search tool
# No direct SDK calls - agent handles search automatically
instructions = """
Use Azure AI Search to query regulations index for KYC, AML, EDD requirements.
Search for relevant regulations based on transaction characteristics.
"""
```

## Hybrid Rule-Based + AI Architecture

This system combines **deterministic compliance rules** with **AI-powered pattern recognition**.

### Rule-Based Risk Thresholds (Hardcoded)

```python
# High-risk countries (always flag)
HIGH_RISK_COUNTRIES = ["NG", "IR", "RU", "KP"]  # Nigeria, Iran, Russia, N. Korea

# Amount thresholds
HIGH_AMOUNT_THRESHOLD = 10000  # USD

# Account age risk
NEW_ACCOUNT_DAYS = 30  # Accounts < 30 days are high risk

# Device trust
LOW_DEVICE_TRUST = 0.5  # Scores < 0.5 are suspicious
```

**Why Rule-Based?** Ensures regulatory compliance with auditable, transparent decisions. These thresholds appear in:

- `challenge-1/workflow/sequential_workflow.py` (customer_data_executor)
- `challenge-1/agents/risk_analyser_agent.py` (instructions)

### AI-Powered Components

- **Risk Analyzer Agent**: Uses Azure AI Search to interpret regulations dynamically
- **Compliance Report Agent**: Generates natural language audit reports
- **Pattern Detection**: AI identifies subtle fraud patterns beyond simple thresholds

## MCP Server Integration (Challenge 2)

**Pattern**: Expose existing REST APIs as MCP servers via Azure API Management.

### MCP Tool Usage

```python
from azure.ai.agents.models import McpTool

mcp_tool = McpTool(
    endpoint=os.environ["MCP_SERVER_ENDPOINT"],
    headers={"Ocp-Apim-Subscription-Key": os.environ["APIM_SUBSCRIPTION_KEY"]}
)

# Agent creation with MCP tool
agent = agents_client.create_agent(
    model=model_deployment_name,
    name="fraud-alert-agent",
    instructions="Use MCP tool to create fraud alerts...",
    tools=[mcp_tool]  # MCP server becomes agent capability
)
```

**Key Insight**: MCP servers let agents call external APIs without custom code. APIM handles the MCP protocol translation automatically.

### APIM Configuration Steps

1. Import API via OpenAPI/Swagger spec
2. Navigate to "MCP Servers (Preview)" in APIM portal
3. Export API as MCP server
4. Use generated endpoint in `MCP_SERVER_ENDPOINT` env var

## Observability with OpenTelemetry (Challenge 3)

### Three-Tier Tracing

```python
from challenge-3.telemetry import TelemetryManager

telemetry = TelemetryManager()
telemetry.initialize_observability()

# Tier 1: Application-level trace
with telemetry.create_workflow_span("fraud_detection_application"):
    # Tier 2: Workflow-level trace
    with telemetry.create_workflow_span("fraud_detection_workflow"):
        # Tier 3: Executor-level trace
        with telemetry.create_processing_span(
            executor_id="customer_data_executor",
            executor_type="DataRetrieval",
            message_type="AnalysisRequest"
        ):
            # Processing logic
```

### Observability Backends (Choose via .env)

- **Application Insights** (production): Set `APPLICATIONINSIGHTS_CONNECTION_STRING`
- **OTLP Endpoint** (custom systems): Set `OTLP_ENDPOINT`
- **VS Code Extension** (dev only): Set `VS_CODE_EXTENSION_PORT`

**Pattern**: Always call `telemetry.initialize_observability()` before workflow execution. The framework automatically instruments agents.

## Authentication & Credentials

### Local Development

```python
from azure.identity.aio import AzureCliCredential

async with AzureCliCredential() as credential:
    # Uses `az login` credentials automatically
```

### Production/General

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()  # Supports managed identity, environment vars, etc.
```

**Never commit** `.env` files or hardcode credentials. Always use Azure identity libraries.

## Common Pitfalls

1. **Missing Agent IDs**: Always add agent IDs to `.env` after creation
2. **Cosmos DB queries**: Must set `enable_cross_partition_query=True`
3. **Type safety**: Return Pydantic models from executors, not plain dicts
4. **Authentication**: Run `az login` before executing any scripts
5. **Data seeding**: Run `challenge-0/seed_data.sh` before testing workflows
6. **DevUI imports**: Requires `from agent_framework.devui import serve`

## Key Files Reference

- `challenge-1/workflow/sequential_workflow.py` - Core 3-executor workflow implementation
- `challenge-3/telemetry.py` - TelemetryManager with OpenTelemetry setup
- `challenge-0/get-keys.sh` - Environment configuration automation
- `.env.sample` - Complete environment variable reference
- `requirements.txt` - Agent Framework dependencies (azure-ai-agents, agent_framework, azure-ai-projects)

## Testing Strategy

**Individual Agents**: Run agent scripts directly (`python customer_data_agent.py`)
**DevUI**: Use `devui_launcher.py` for interactive testing
**Workflows**: Test in Jupyter notebooks first, then Python scripts
**Observability**: Verify traces in Application Insights after workflow runs

---

_This is a hackathon learning project. Code prioritizes educational clarity over production optimizations._
