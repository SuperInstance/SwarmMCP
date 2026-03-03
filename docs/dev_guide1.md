
<div align="center">

# The Swarm Developer Guide
## *Build the Infrastructure for Distributed Cognition*

**From first clone to production swarm**

</div>

---

## What This Guide Is

This is the **single source of truth** for building, extending, and deploying Swarm. Whether you're fixing a typo or architecting a new provider adapter, this document shows you exactly how to do it—and do it well.

**Prerequisites assumed**: Python 3.10+, basic async/await, HTTP APIs, and some frustration with AI rate limits.

---

## The 5-Minute Victory

Get Swarm running locally before you understand it. Momentum matters.

```bash
# 1. Clone
git clone https://github.com/yourusername/swarm-mcp.git
cd swarm-mcp

# 2. Install (uv recommended for speed)
curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync

# 3. Configure (copy and edit)
cp config/example.json config/local.json
# Edit config/local.json with at least one provider key

# 4. Run the test swarm
python -m swarm_mcp.server --config config/local.json --transport stdio

# 5. Verify (in another terminal)
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python -m swarm_mcp.client
```

**If you see a list of tools, you're in.** Everything else is details.

---

## Architecture at 30,000 Feet

Understand the data flow before diving deep.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   MCP Client │────→│   Swarm     │────→│   Provider  │────→│   AI Model  │
│ (Claude Code)│     │   Server    │     │   Adapter   │     │ (Kimi/etc.) │
└─────────────┘     └──────┬──────┘     └─────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    │  The Three   │
                    │   Engines:   │
                    │              │
                    │ 1. Router    │ ← Decides who does what
                    │ 2. Executor  │ ← Spawns and monitors agents
                    │ 3. Aggregator│ ← Combines results
                    └─────────────┘
```

**Key insight**: Swarm is a **protocol translator and orchestrator**, not an AI itself. It speaks MCP on one side, HTTP on the other, and manages the messy middle.

---

## Project Structure

```
swarm-mcp/
├── src/swarm_mcp/
│   ├── __init__.py
│   ├── server.py              # MCP protocol implementation
│   ├── models/
│   │   ├── __init__.py
│   │   ├── requests.py        # Pydantic models for MCP requests
│   │   ├── responses.py       # Pydantic models for MCP responses
│   │   └── providers.py       # Provider configuration schemas
│   ├── engines/
│   │   ├── __init__.py
│   │   ├── router.py          # Task classification & provider selection
│   │   ├── executor.py        # Parallel agent lifecycle management
│   │   └── aggregator.py      # Result combination strategies
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── base.py            # Abstract ProviderAdapter class
│   │   ├── kimi.py            # Moonshot AI implementation
│   │   ├── deepseek.py        # DeepSeek AI implementation
│   │   ├── zai.py             # Z.ai (ChatGLM) implementation
│   │   ├── qwen.py            # Alibaba Qwen implementation
│   │   └── claude.py          # Anthropic Claude implementation
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── quota.py           # Unified quota tracking
│   │   ├── cost.py            # Cost estimation and logging
│   │   └── circuit_breaker.py # Fault tolerance patterns
│   └── config.py              # Configuration loading and validation
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
│   ├── architecture/            # ADRs and deep dives
│   ├── providers/               # Provider-specific guides
│   └── tutorials/               # Step-by-step walkthroughs
├── config/
│   ├── example.json             # Template configuration
│   └── schema.json              # JSON Schema for validation
├── pyproject.toml
├── README.md
└── CONTRIBUTING.md              # PR process and standards
```

**Rule of thumb**: If you're adding a feature, it lives in `engines/`. If you're adding a provider, it lives in `providers/`. If you're fixing bugs, check `utils/` first.

---

## The Core Abstractions

### 1. The Provider Adapter

Every AI provider implements this interface. Master this, and you can add any provider in 30 minutes.

```python
# src/swarm_mcp/providers/base.py
from abc import ABC, abstractmethod
from typing import AsyncIterator, Dict, Any
from pydantic import BaseModel

class AgentTask(BaseModel):
    id: str
    prompt: str
    system_message: Optional[str] = None
    temperature: float = 0.1
    max_tokens: int = 4096
    # Provider-specific extensions
    extra_params: Dict[str, Any] = {}

class AgentResult(BaseModel):
    task_id: str
    content: str
    usage: TokenUsage
    cost: float  # USD
    duration_ms: int
    provider: str

class QuotaInfo(BaseModel):
    remaining: int  # tokens or requests
    resets_at: datetime
    limit_type: str  # "token", "request", "hourly"

class ProviderAdapter(ABC):
    """Abstract base for all AI provider integrations."""
    
    def __init__(self, config: ProviderConfig):
        self.config = config
        self.http_client = AsyncHTTPClient(
            timeout=config.timeout,
            retries=3
        )
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=60
        )
    
    @abstractmethod
    async def spawn_agent(self, task: AgentTask) -> str:
        """Initialize agent and return agent ID. Non-blocking."""
        pass
    
    @abstractmethod
    async def stream_agent(self, agent_id: str) -> AsyncIterator[str]:
        """Yield tokens as they arrive from provider."""
        pass
    
    @abstractmethod
    async def get_result(self, agent_id: str) -> AgentResult:
        """Get final result including usage and cost."""
        pass
    
    @abstractmethod
    async def check_quota(self) -> QuotaInfo:
        """Return real-time quota status."""
        pass
    
    @abstractmethod
    def estimate_cost(self, input_tokens: int, output_tokens: int) -> float:
        """Calculate cost in USD for hypothetical request."""
        pass
    
    # Concrete helper (shared across providers)
    async def _with_circuit_breaker(self, operation: Callable, *args):
        """Wrap operation in circuit breaker pattern."""
        if self.circuit_breaker.is_open:
            raise ProviderUnavailable(f"{self.config.name} circuit open")
        try:
            result = await operation(*args)
            self.circuit_breaker.record_success()
            return result
        except Exception as e:
            self.circuit_breaker.record_failure()
            raise
```

### 2. The Router

Decides where tasks go. This is where the "intelligence" lives.

```python
# src/swarm_mcp/engines/router.py
class TaskProfile(BaseModel):
    complexity_score: float  # 0.0 to 1.0
    latency_requirement: float  # seconds
    quality_requirement: float  # 0.0 to 1.0
    budget_ceiling: Optional[float]  # USD
    preferred_provider: Optional[str]

class RoutingDecision(BaseModel):
    provider: str
    model: str
    strategy: str  # "speed", "quality", "cost", "hybrid"
    estimated_cost: float
    estimated_duration: float
    fallback_chain: List[str]

class Router:
    def __init__(self, providers: Dict[str, ProviderAdapter]):
        self.providers = providers
        self.cost_index = CostIndex()
        self.performance_tracker = PerformanceTracker()
    
    async def route(self, task: AgentTask, profile: TaskProfile) -> RoutingDecision:
        """Select optimal provider based on task profile and real-time conditions."""
        
        # Filter: providers with available quota
        available = [
            name for name, adapter in self.providers.items()
            if (await adapter.check_quota()).remaining > task.max_tokens * 2
        ]
        
        if not available:
            raise QuotaExhausted("No providers with available quota")
        
        # Score each provider
        scores = {}
        for name in available:
            adapter = self.providers[name]
            scores[name] = self._score_provider(adapter, task, profile)
        
        # Select best
        best = max(scores, key=scores.get)
        
        return RoutingDecision(
            provider=best,
            model=self.providers[best].config.model,
            strategy=self._determine_strategy(profile),
            estimated_cost=self.providers[best].estimate_cost(
                len(task.prompt) // 4,  # rough token estimate
                task.max_tokens
            ),
            estimated_duration=self.performance_tracker.get_latency_estimate(best),
            fallback_chain=self._build_fallback_chain(best, available)
        )
    
    def _score_provider(self, adapter: ProviderAdapter, task: AgentTask, profile: TaskProfile) -> float:
        """Calculate suitability score (0.0 to 1.0)."""
        
        # Capability match (does this provider do what we need?)
        capability = adapter.config.capabilities.get(task.type, 0.5)
        
        # Cost efficiency (lower is better, normalized)
        cost = adapter.estimate_cost(1000, 1000)  # normalized to 1K tokens
        cost_score = 1.0 - (cost / 20.0)  # $20/M = 0.0, $0 = 1.0
        
        # Performance (historical latency)
        latency = self.performance_tracker.get_p99_latency(adapter.config.name)
        perf_score = 1.0 - min(latency / 10.0, 1.0)  # 10s+ = 0.0
        
        # Quota headroom (don't pick provider at 95% quota)
        quota = asyncio.run(adapter.check_quota())  # cached, non-blocking
        quota_score = min(quota.remaining / 10000, 1.0)
        
        # Weighted combination based on task priority
        if profile.latency_requirement < 2.0:
            return 0.5 * perf_score + 0.3 * capability + 0.2 * quota_score
        elif profile.budget_ceiling and profile.budget_ceiling < 1.0:
            return 0.5 * cost_score + 0.3 * capability + 0.2 * quota_score
        else:
            return 0.5 * capability + 0.3 * perf_score + 0.2 * quota_score
```

### 3. The Executor

Spawns and manages parallel agents. This is where the "swarm" happens.

```python
# src/swarm_mcp/engines/executor.py
class AgentInstance(BaseModel):
    id: str
    task: AgentTask
    provider: str
    status: str  # "pending", "running", "completed", "failed"
    result: Optional[AgentResult]
    started_at: Optional[datetime]
    completed_at: Optional[datetime]

class ExecutionPlan(BaseModel):
    tasks: List[AgentTask]
    routing_decisions: List[RoutingDecision]
    max_concurrent: int
    timeout_per_task: int

class Executor:
    def __init__(self, providers: Dict[str, ProviderAdapter]):
        self.providers = providers
        self.active_agents: Dict[str, AgentInstance] = {}
        self.semaphores: Dict[str, asyncio.Semaphore] = {
            name: asyncio.Semaphore(adapter.config.max_parallel)
            for name, adapter in providers.items()
        }
    
    async def execute_swarm(self, plan: ExecutionPlan) -> List[AgentResult]:
        """Execute all tasks with provider-specific concurrency limits."""
        
        # Group tasks by provider for efficient batching
        by_provider = defaultdict(list)
        for task, decision in zip(plan.tasks, plan.routing_decisions):
            by_provider[decision.provider].append((task, decision))
        
        # Create execution coroutines with semaphore control
        coroutines = []
        for provider_name, task_list in by_provider.items():
            semaphore = self.semaphores[provider_name]
            for task, decision in task_list:
                coro = self._execute_with_semaphore(
                    semaphore, provider_name, task, decision
                )
                coroutines.append(coro)
        
        # Execute with global concurrency limit
        results = await asyncio.gather(*coroutines, return_exceptions=True)
        
        # Handle failures and fallbacks
        final_results = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                # Trigger fallback
                task = plan.tasks[i]
                fallback_result = await self._execute_fallback(
                    task, plan.routing_decisions[i].fallback_chain
                )
                final_results.append(fallback_result)
            else:
                final_results.append(result)
        
        return final_results
    
    async def _execute_with_semaphore(
        self, 
        semaphore: asyncio.Semaphore,
        provider_name: str,
        task: AgentTask,
        decision: RoutingDecision
    ) -> AgentResult:
        """Execute single agent with concurrency control."""
        
        async with semaphore:
            adapter = self.providers[provider_name]
            agent_id = await adapter.spawn_agent(task)
            
            instance = AgentInstance(
                id=agent_id,
                task=task,
                provider=provider_name,
                status="running",
                started_at=datetime.now()
            )
            self.active_agents[agent_id] = instance
            
            try:
                # Stream and collect result
                chunks = []
                async for token in adapter.stream_agent(agent_id):
                    chunks.append(token)
                
                result = await adapter.get_result(agent_id)
                instance.status = "completed"
                instance.result = result
                instance.completed_at = datetime.now()
                
                return result
                
            except Exception as e:
                instance.status = "failed"
                raise ExecutionError(f"Agent {agent_id} failed: {e}")
    
    async def _execute_fallback(
        self,
        task: AgentTask,
        fallback_chain: List[str]
    ) -> AgentResult:
        """Try fallback providers in sequence."""
        
        for provider_name in fallback_chain:
            try:
                adapter = self.providers[provider_name]
                if (await adapter.check_quota()).remaining < task.max_tokens:
                    continue
                
                agent_id = await adapter.spawn_agent(task)
                result = await adapter.get_result(agent_id)
                return result
                
            except Exception:
                continue
        
        raise ExecutionError("All fallback providers exhausted")
```

### 4. The Aggregator

Combines results from many agents into coherent output.

```python
# src/swarm_mcp/engines/aggregator.py
class AggregationStrategy(Enum):
    CONCATENATE = "concatenate"      # Join with separators
    SUMMARIZE = "summarize"          # Use LLM to condense
    MERGE_CODE = "merge_code"        # Intelligent code merging
    JSON_MERGE = "json_merge"        # Deep merge of JSON structures
    VOTE = "vote"                    # Majority voting for correctness

class Aggregator:
    def __init__(self, providers: Dict[str, ProviderAdapter]):
        self.providers = providers
    
    async def aggregate(
        self,
        results: List[AgentResult],
        strategy: AggregationStrategy,
        original_tasks: List[AgentTask]
    ) -> str:
        """Combine multiple agent results based on strategy."""
        
        if strategy == AggregationStrategy.CONCATENATE:
            return self._concatenate(results)
        
        elif strategy == AggregationStrategy.SUMMARIZE:
            return await self._summarize(results)
        
        elif strategy == AggregationStrategy.MERGE_CODE:
            return self._merge_code(results, original_tasks)
        
        elif strategy == AggregationStrategy.JSON_MERGE:
            return self._json_merge(results)
        
        elif strategy == AggregationStrategy.VOTE:
            return self._vote(results)
        
        else:
            raise ValueError(f"Unknown strategy: {strategy}")
    
    def _concatenate(self, results: List[AgentResult]) -> str:
        """Simple join with clear separators."""
        parts = []
        for r in results:
            parts.append(f"## Result from {r.provider}\n\n{r.content}\n")
        return "\n---\n".join(parts)
    
    async def _summarize(self, results: List[AgentResult]) -> str:
        """Use cheapest provider to condense results."""
        # Pick cheapest available provider for summarization
        cheapest = min(
            self.providers.values(),
            key=lambda p: p.estimate_cost(1000, 500)
        )
        
        combined = "\n\n".join([r.content for r in results])
        prompt = f"Summarize the following {len(results)} results concisely:\n\n{combined}"
        
        # Spawn summarization agent
        task = AgentTask(id="summary", prompt=prompt, max_tokens=2000)
        agent_id = await cheapest.spawn_agent(task)
        summary_result = await cheapest.get_result(agent_id)
        
        return summary_result.content
    
    def _merge_code(self, results: List[AgentResult], tasks: List[AgentTask]) -> str:
        """Intelligent merging of code changes."""
        # Parse file paths from task prompts
        file_changes = defaultdict(list)
        
        for task, result in zip(tasks, results):
            # Extract file path from task prompt (e.g., "Convert src/auth.ts...")
            match = re.search(r'(\S+\.(ts|js|py|rs|go))', task.prompt)
            if match:
                filepath = match.group(1)
                file_changes[filepath].append(result.content)
        
        # For files with multiple changes, use most recent or merge intelligently
        merged = {}
        for filepath, changes in file_changes.items():
            if len(changes) == 1:
                merged[filepath] = changes[0]
            else:
                # Simple strategy: take longest (most comprehensive) change
                # Advanced: 3-way merge or AST-based merging
                merged[filepath] = max(changes, key=len)
        
        # Format as unified output
        output = []
        for filepath, content in merged.items():
            output.append(f"### {filepath}\n\n```\n{content}\n```\n")
        
        return "\n".join(output)
```

---

## Adding a New Provider

The true test of our architecture: can you add a new AI provider in 30 minutes?

### Step 1: Create the Adapter

```python
# src/swarm_mcp/providers/minimax.py
from .base import ProviderAdapter, AgentTask, AgentResult, QuotaInfo
from pydantic import BaseModel

class MiniMaxConfig(BaseModel):
    api_key: str
    base_url: str = "https://api.minimaxi.chat/v1"
    model: str = "MiniMax-Text-01"
    max_parallel: int = 50
    
    # MiniMax-specific
    group_id: Optional[str] = None  # For enterprise accounts

class MiniMaxAdapter(ProviderAdapter):
    """MiniMax (稀宇科技) - Strong Chinese model, good for productivity tasks."""
    
    def __init__(self, config: MiniMaxConfig):
        super().__init__(config)
        self.config = config
    
    async def spawn_agent(self, task: AgentTask) -> str:
        """MiniMax uses a completion-style API."""
        
        headers = {
            "Authorization": f"Bearer {self.config.api_key}",
            "Content-Type": "application/json"
        }
        
        # MiniMax uses /text/chatcompletion_v2
        payload = {
            "model": self.config.model,
            "messages": [
                {"role": "system", "content": task.system_message or "You are a helpful assistant."},
                {"role": "user", "content": task.prompt}
            ],
            "temperature": task.temperature,
            "max_tokens": task.max_tokens,
            "stream": True,
            **task.extra_params.get("minimax", {})
        }
        
        response = await self.http_client.post(
            f"{self.config.base_url}/text/chatcompletion_v2",
            headers=headers,
            json=payload
        )
        
        data = response.json()
        # MiniMax returns base_resp with status_code and task_id
        if data.get("base_resp", {}).get("status_code") != 0:
            raise ProviderError(f"MiniMax error: {data}")
        
        return data["task_id"]  # Or whatever ID field they use
    
    async def stream_agent(self, agent_id: str) -> AsyncIterator[str]:
        """Stream tokens from MiniMax."""
        # Implementation depends on their streaming protocol
        # Usually Server-Sent Events (SSE)
        pass
    
    async def get_result(self, agent_id: str) -> AgentResult:
        """Fetch final result and usage stats."""
        pass
    
    async def check_quota(self) -> QuotaInfo:
        """MiniMax quota checking."""
        # They may not have a direct quota endpoint
        # Return conservative estimate or fetch from dashboard API
        return QuotaInfo(
            remaining=100000,  # Conservative default
            resets_at=datetime.now() + timedelta(days=1),
            limit_type="token"
        )
    
    def estimate_cost(self, input_tokens: int, output_tokens: int) -> float:
        """MiniMax pricing: $0.15/M input, $1.20/M output (Standard)"""
        input_cost = (input_tokens / 1_000_000) * 0.15
        output_cost = (output_tokens / 1_000_000) * 1.20
        return input_cost + output_cost
```

### Step 2: Register in Factory

```python
# src/swarm_mcp/providers/__init__.py
from .kimi import KimiAdapter, KimiConfig
from .deepseek import DeepSeekAdapter, DeepSeekConfig
from .minimax import MiniMaxAdapter, MiniMaxConfig  # Add this

PROVIDER_REGISTRY = {
    "kimi": (KimiAdapter, KimiConfig),
    "deepseek": (DeepSeekAdapter, DeepSeekConfig),
    "minimax": (MiniMaxAdapter, MiniMaxConfig),  # Add this
    # ... etc
}
```

### Step 3: Add Tests

```python
# tests/unit/providers/test_minimax.py
import pytest
from swarm_mcp.providers.minimax import MiniMaxAdapter, MiniMaxConfig

@pytest.fixture
def adapter():
    config = MiniMaxConfig(api_key="test-key", group_id="test-group")
    return MiniMaxAdapter(config)

@pytest.mark.asyncio
async def test_estimate_cost(adapter):
    cost = adapter.estimate_cost(1_000_000, 500_000)
    assert cost == 0.15 + 0.60  # $0.75 total

@pytest.mark.asyncio
async def test_spawn_agent_mock(adapter, httpx_mock):
    httpx_mock.add_response(
        url="https://api.minimaxi.chat/v1/text/chatcompletion_v2",
        json={
            "base_resp": {"status_code": 0},
            "task_id": "task-123"
        }
    )
    
    from swarm_mcp.providers.base import AgentTask
    task = AgentTask(id="test", prompt="Hello", max_tokens=100)
    agent_id = await adapter.spawn_agent(task)
    
    assert agent_id == "task-123"
```

### Step 4: Document

Add to `docs/providers/minimax.md`:

```markdown
# MiniMax Provider

**Status**: Experimental  
**Maintainer**: @yourusername  
**Last verified**: 2025-03-04

## Configuration

```json
{
  "providers": {
    "minimax": {
      "api_key": "YOUR_MINIMAX_KEY",
      "group_id": "YOUR_GROUP_ID",  // Optional, for enterprise
      "model": "MiniMax-Text-01",
      "max_parallel": 50
    }
  }
}
```

## Capabilities

- **Strengths**: Office productivity, document processing, Chinese language
- **Weaknesses**: Code generation (use Kimi or DeepSeek instead)
- **Cost**: $0.15/M input (Standard), $0.30/M (Lightning)

## Quota Behavior

MiniMax does not expose real-time quota via API. We conservatively estimate 
100K tokens remaining and refresh daily. Monitor your dashboard at 
https://www.minimaxi.com/platform.

## Known Issues

- Streaming protocol uses non-standard SSE format (workaround implemented)
- Group ID required for some enterprise features
```

**Done.** You've added a provider. The system now routes to it automatically based on task profiles and availability.

---

## Testing Strategy

### The Testing Pyramid

```
                    ▲
                   / \
                  / E2E \     5% - Full MCP flow with real providers (mocked)
                 /-------\
                /         \
               / Integration \  15% - Router + Executor + real provider adapters
              /---------------\
             /                 \
            /     Unit Tests     \  80% - Individual functions, mocked dependencies
           /---------------------\
```

### Unit Tests (Fast, Isolated)

```python
# tests/unit/engines/test_router.py
@pytest.mark.asyncio
async def test_router_prefers_cheap_for_simple_tasks():
    """Economic test: simple tasks go to cheapest provider."""
    
    # Setup: Kimi ($0.60), DeepSeek ($0.28), Claude ($15.00)
    providers = {
        "kimi": MockAdapter(cost_per_1m=0.60),
        "deepseek": MockAdapter(cost_per_1m=0.28),
        "claude": MockAdapter(cost_per_1m=15.00),
    }
    
    router = Router(providers)
    
    # Simple task: "Fix typo in README"
    task = AgentTask(id="t1", prompt="Fix typo in README", max_tokens=100)
    profile = TaskProfile(
        complexity_score=0.1,  # Very simple
        latency_requirement=10.0,  # Not urgent
        quality_requirement=0.3,  # Low quality bar
    )
    
    decision = await router.route(task, profile)
    
    assert decision.provider == "deepseek"  # Cheapest
    assert decision.estimated_cost < 0.01  # Pennies
```

### Integration Tests (Real APIs, Controlled)

```python
# tests/integration/test_kimi_integration.py
import os
import pytest

# Skip if no API key
pytestmark = pytest.mark.skipif(
    not os.getenv("KIMI_API_KEY"),
    reason="KIMI_API_KEY not set"
)

@pytest.mark.asyncio
async def test_kimi_real_quota_check():
    """Verify quota checking works with real API."""
    from swarm_mcp.providers.kimi import KimiAdapter, KimiConfig
    
    config = KimiConfig(api_key=os.getenv("KIMI_API_KEY"))
    adapter = KimiAdapter(config)
    
    quota = await adapter.check_quota()
    
    assert quota.remaining > 0
    assert quota.resets_at > datetime.now()
    print(f"Kimi quota remaining: {quota.remaining}")
```

### E2E Tests (Full Flow)

```python
# tests/e2e/test_mcp_flow.py
@pytest.mark.asyncio
async def test_full_swarm_spawn():
    """Test complete MCP flow: request → route → execute → aggregate."""
    
    # Start server subprocess
    server = await asyncio.create_subprocess_exec(
        "python", "-m", "swarm_mcp.server",
        "--config", "config/test.json",
        "--transport", "stdio",
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
    )
    
    # Send MCP initialize
    init_request = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "initialize",
        "params": {"protocolVersion": "2024-11-05", "capabilities": {}}
    }
    
    server.stdin.write(json.dumps(init_request).encode() + b"\n")
    await server.stdin.drain()
    
    # Read response
    response = await server.stdout.readline()
    data = json.loads(response)
    
    assert data["result"]["protocolVersion"] == "2024-11-05"
    
    # Send tool call
    tool_request = {
        "jsonrpc": "2.0",
        "id": 2,
        "method": "tools/call",
        "params": {
            "name": "swarm_spawn_agents",
            "arguments": {
                "tasks": ["Say hello", "Say world"],
                "provider": "kimi",
                "max_concurrent": 2
            }
        }
    }
    
    server.stdin.write(json.dumps(tool_request).encode() + b"\n")
    await server.stdin.drain()
    
    response = await server.stdout.readline()
    data = json.loads(response)
    
    assert "result" in data
    assert len(data["result"]["content"]) == 2  # Two results
    
    server.terminate()
```

### The Test Fixtures

```python
# tests/fixtures/providers.py
"""Mock providers for testing without API keys."""

class MockAdapter(ProviderAdapter):
    """Deterministic mock for unit tests."""
    
    def __init__(self, name: str, cost_per_1m: float, latency_ms: int = 100):
        self.name = name
        self.cost_per_1m = cost_per_1m
        self.latency_ms = latency_ms
        self.spawned_agents = []
    
    async def spawn_agent(self, task: AgentTask) -> str:
        agent_id = f"{self.name}-{len(self.spawned_agents)}"
        self.spawned_agents.append((agent_id, task))
        return agent_id
    
    async def stream_agent(self, agent_id: str) -> AsyncIterator[str]:
        yield f"[Mock {self.name} output]"
    
    async def get_result(self, agent_id: str) -> AgentResult:
        return AgentResult(
            task_id=agent_id,
            content=f"Mock result from {self.name}",
            usage=TokenUsage(input=100, output=50),
            cost=self.estimate_cost(100, 50),
            duration_ms=self.latency_ms,
            provider=self.name
        )
    
    async def check_quota(self) -> QuotaInfo:
        return QuotaInfo(
            remaining=1000000,
            resets_at=datetime.now() + timedelta(hours=1),
            limit_type="token"
        )
    
    def estimate_cost(self, input_tokens: int, output_tokens: int) -> float:
        return (input_tokens + output_tokens) / 1_000_000 * self.cost_per_1m
```

---

## Debugging & Observability

### Structured Logging

Every component logs with context:

```python
# src/swarm_mcp/utils/logging.py
import structlog

logger = structlog.get_logger()

# In router.py
logger.info(
    "routing_decision",
    task_id=task.id,
    selected_provider=decision.provider,
    estimated_cost=decision.estimated_cost,
    strategy=decision.strategy,
    fallback_chain=decision.fallback_chain,
)

# In executor.py
logger.info(
    "agent_spawned",
    agent_id=agent_id,
    provider=provider_name,
    semaphore_queue_length=semaphore._value,  # Current slots available
)

logger.warning(
    "provider_degraded",
    provider=provider_name,
    error=str(e),
    circuit_breaker_status="OPEN",
    retry_after=60,
)
```

### The Debug Dashboard

Run with `--debug` to get real-time swarm visualization:

```bash
python -m swarm_mcp.server --config config/local.json --debug
```

Opens at `http://localhost:8080`:

```
┌─────────────────────────────────────────┐
│           SWARM DEBUG DASHBOARD         │
├─────────────────────────────────────────┤
│ Active Agents: 47                       │
│ ├─ Kimi: 32 [████████████░░░░░░░░░░]    │
│ ├─ DeepSeek: 12 [████░░░░░░░░░░░░░░]    │
│ └─ Z.ai: 3 [█░░░░░░░░░░░░░░░░░░░]      │
│                                         │
│ Queue Depth: 12 (waiting for quota)     │
│                                         │
│ Recent Decisions:                       │
│ 14:32:23 → Task-1842 → Kimi ($0.04)     │
│ 14:32:24 → Task-1843 → DeepSeek ($0.02) │
│ 14:32:24 → Task-1844 → Kimi ($0.04)     │
│                                         │
│ Cost This Hour: $12.47                  │
│ Est. Savings vs Claude: $187.53 (93%)   │
└─────────────────────────────────────────┘
```

### Tracing

Every request gets a trace ID:

```python
# In server.py
trace_id = generate_trace_id()
with structlog.contextvars.bind_contextvars(trace_id=trace_id):
    result = await handle_request(request)
    
# Now all logs include trace_id
# Filter: trace_id=abc-123-xyz
```

---

## Deployment Patterns

### Pattern 1: Local Development (stdio)

```bash
# Claude Code spawns Swarm as subprocess
claude mcp add swarm-dev -- python -m swarm_mcp.server --config config/dev.json
```

**Best for**: Daily development, debugging, single-user

### Pattern 2: Team Server (SSE)

```bash
# Run as persistent HTTP service
docker run -p 3000:3000 \
  -v $(pwd)/config:/config \
  swarm-mcp:latest \
  --transport sse --port 3000 --config /config/team.json
```

**Best for**: Team sharing, CI/CD integration, remote access

### Pattern 3: Kubernetes (Production)

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swarm-mcp
spec:
  replicas: 3  # Horizontal scaling
  selector:
    matchLabels:
      app: swarm-mcp
  template:
    spec:
      containers:
      - name: swarm
        image: swarm-mcp:v1.2.3
        args: ["--transport", "sse", "--port", "3000"]
        env:
        - name: SWARM_CONFIG
          valueFrom:
            configMapKeyRef:
              name: swarm-config
              key: config.json
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Pattern 4: Serverless (Lambda/Cloud Functions)

**Not recommended** for Swarm—cold start latency kills the "spawn 100 agents instantly" value proposition. Use persistent containers.

---

## Performance Tuning

### The Bottlenecks

| Bottleneck | Symptom | Fix |
|-----------|---------|-----|
| **Provider latency** | Slow individual agents | Add more providers, use faster ones for latency-critical tasks |
| **Quota exhaustion** | 429 errors, queue buildup | Implement aggressive fallback, monitor quota proactively |
| **Memory pressure** | OOM kills with large swarms | Stream results, don't buffer all in memory |
| **Context window** | Truncated results | Use aggregation strategies, chunk large outputs |
| **Network limits** | Connection pool exhaustion | Tune `aiohttp` limits, use HTTP/2 where available |

### Tuning Checklist

```python
# config/production.json
{
  "performance": {
    "http": {
      "connection_pool_size": 100,  # Per provider
      "keepalive_timeout": 30,
      "http2": true
    },
    "executor": {
      "max_concurrent_global": 200,  # Across all providers
      "task_timeout_seconds": 120,
      "enable_streaming": true  # Don't buffer full responses
    },
    "router": {
      "quota_refresh_interval_seconds": 30,  # Check quotas often
      "latency_window_size": 100  # Last 100 requests for p99
    },
    "aggregator": {
      "max_result_size_tokens": 10000,  # Summarize if larger
      "enable_parallel_summarization": true
    }
  }
}
```

---

## Common Tasks

### Task: Add a new routing strategy

```python
# src/swarm_mcp/engines/router.py
class RoutingStrategy(Enum):
    COST_OPTIMIZED = "cost_optimized"
    SPEED_DEMON = "speed_demon"
    QUALITY_OR_DIE = "quality_or_die"
    MY_CUSTOM = "my_custom"  # Add this

def _apply_strategy(self, strategy: RoutingStrategy, scores: Dict[str, float]):
    if strategy == RoutingStrategy.MY_CUSTOM:
        # Your logic here
        return {k: v * self._custom_weight(k) for k, v in scores.items()}
```

### Task: Fix a flaky provider adapter

```python
# Add retry logic with exponential backoff
from tenacity import retry, stop_after_attempt, wait_exponential

class KimiAdapter(ProviderAdapter):
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10),
        retry=retry_if_exception_type((TimeoutError, 503))
    )
    async def spawn_agent(self, task: AgentTask) -> str:
        # Implementation
```

### Task: Add metrics export

```python
# src/swarm_mcp/utils/metrics.py
from prometheus_client import Counter, Histogram, Gauge

swarm_agents_spawned = Counter(
    'swarm_agents_spawned_total',
    'Total agents spawned',
    ['provider']
)

swarm_routing_duration = Histogram(
    'swarm_routing_duration_seconds',
    'Time to make routing decision'
)

swarm_cost_usd = Gauge(
    'swarm_cost_usd',
    'Estimated cost of current operation'
)

# In executor.py
swarm_agents_spawned.labels(provider=provider_name).inc()
```

---

## The Checklist

Before submitting a PR:

- [ ] **Tests**: Unit tests pass (`pytest tests/unit/`)
- [ ] **Integration**: Integration tests pass if you have API keys (`pytest tests/integration/ -v`)
- [ ] **Type check**: `mypy src/swarm_mcp/` passes
- [ ] **Lint**: `ruff check .` and `ruff format --check .` pass
- [ ] **Docs**: Updated if you changed behavior
- [ ] **ADR**: Written if you changed architecture (see `docs/architecture/adr/`)
- [ ] **Cost**: Documented if you changed pricing or routing

---

## Getting Help

**Stack Overflow**: Tag questions with `swarm-mcp`  
**Discord**: `#dev-help` for implementation, `#architecture` for design  
**Office Hours**: Tuesdays 18:00 UTC, `#office-hours` voice channel  
**Emergency**: Ping `@warden-oncall` for production issues

---

## The Final Pattern

Every feature in Swarm follows this lifecycle:

```
1. PROBLEM → Write an ADR explaining why we need this
2. DESIGN → Get feedback on approach from 2+ maintainers  
3. SPIKE → Build minimal proof-of-concept (throwaway code)
4. IMPLEMENT → Production code with tests and docs
5. REVIEW → PR with clear description, tests pass
6. MERGE → Squash merge, version bump if needed
7. OBSERVE → Monitor in production, iterate
```

**Speed matters, but sustainability matters more.** Build for the maintainer at 3 AM six months from now.

---

<div align="center">

**Now build something.**

[← Back to Architecture](./architecture.md) | [Provider Reference →](./providers/)

</div>

---

This Developer Guide gives you everything needed to contribute effectively: concrete code patterns, testing strategies, debugging tools, and deployment options—while maintaining the clarity and purpose that make Swarm worth building.
