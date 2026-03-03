
<div align="center">

# The Swarm Developer Guide
## *Build, Extend, Orchestrate*

**The complete technical reference for developing with and contributing to Swarm MCP**

</div>

---

## Table of Contents

1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Getting Started](#getting-started)
4. [Architecture Deep Dive](#architecture-deep-dive)
5. [Building Provider Adapters](#building-provider-adapters)
6. [Extending the Router](#extending-the-router)
7. [Testing & Debugging](#testing--debugging)
8. [Performance Tuning](#performance-tuning)
9. [Contributing](#contributing)
10. [API Reference](#api-reference)
11. [Troubleshooting](#troubleshooting)

---

## Introduction

### What You're Building

Swarm MCP is a **multi-agent orchestration bridge**. It sits between AI clients (like Claude Code) and AI providers (like Kimi, DeepSeek, Z.ai), enabling:

- **Parallel execution**: Spawn 100 agents simultaneously instead of 3-5
- **Intelligent routing**: Automatically select the cheapest capable provider
- **Resilient fallback**: Continue working when providers fail or rate-limit
- **Cost optimization**: Reduce AI infrastructure costs by 10-50×

### Who This Guide Is For

| Reader | Start Here |
|--------|-----------|
| **User** | [Getting Started](#getting-started) → [Configuration](#configuration) |
| **Integrator** | [Architecture](#architecture-deep-dive) → [Provider Adapters](#building-provider-adapters) |
| **Contributor** | [Contributing](#contributing) → [Testing](#testing--debugging) |
| **Architect** | [Router Extension](#extending-the-router) → [Performance](#performance-tuning) |

### Prerequisites

- **Python 3.10+** (async/await, type hints, dataclasses)
- **Understanding of async Python** (`asyncio`, `aiohttp`, context managers)
- **Familiarity with OpenAI API schema** (most providers use this)
- **Basic MCP knowledge** (we'll teach the rest)

---

## Core Concepts

### The Mental Model

Think of Swarm as a **smart load balancer for AI requests**. Like NGINX or HAProxy, but understanding:
- **Cognitive cost** (not just $/request, but quality/price tradeoffs)
- **Semantic routing** (what kind of thinking does this task need?)
- **Parallel decomposition** (how many agents can work on this simultaneously?)

### Key Abstractions

```
┌─────────────────────────────────────────────────────────────┐
│                        SWARM MCP                              │
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │    Task     │───→│    Agent    │───→│   Result    │      │
│  │  (Request)  │    │  (Worker)   │    │  (Response) │      │
│  └─────────────┘    └─────────────┘    └─────────────┘      │
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │   Router    │    │   Provider  │    │  Aggregator │      │
│  │ (Decides)   │    │  (Executes) │    │ (Combines)  │      │
│  └─────────────┘    └─────────────┘    └─────────────┘      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Task**: A unit of work to be performed (e.g., "refactor this file")
**Agent**: An isolated LLM instance with its own context
**Result**: The output from one agent
**Router**: Decides which provider(s) to use
**Provider**: External API endpoint (Kimi, DeepSeek, etc.)
**Aggregator**: Combines multiple results into coherent output

### The Lifecycle of a Request

```
1. Claude Code sends MCP request
           ↓
2. Swarm validates and parses request
           ↓
3. Router classifies task and selects provider(s)
           ↓
4. Execution Engine spawns N parallel agents
           ↓
5. Each agent streams results from provider
           ↓
6. Aggregator combines results
           ↓
7. Response returned to Claude Code
```

---

## Getting Started

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/swarm-mcp.git
cd swarm-mcp

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install in development mode
pip install -e ".[dev]"

# Verify installation
python -m swarm_mcp --version
# Should output: swarm-mcp 0.1.0
```

### Configuration

Create your provider configuration:

```bash
mkdir -p ~/.config/swarm-mcp
cat > ~/.config/swarm-mcp/config.json <<'EOF'
{
  "providers": {
    "kimi": {
      "api_key": "sk-your-moonshot-key",
      "base_url": "https://api.moonshot.cn/v1",
      "model": "kimi-k2-5",
      "max_parallel": 100,
      "timeout": 60,
      "cost_per_1m_input": 0.60,
      "cost_per_1m_output": 2.50,
      "capabilities": ["coding", "long_context", "parallel"],
      "health_check_interval": 30
    },
    "deepseek": {
      "api_key": "sk-your-deepseek-key",
      "base_url": "https://api.deepseek.com/v1",
      "model": "deepseek-chat",
      "max_parallel": 50,
      "timeout": 45,
      "cost_per_1m_input": 0.28,
      "cost_per_1m_output": 0.42,
      "capabilities": ["coding", "reasoning", "math"],
      "cache_discount": 0.90
    }
  },
  "routing": {
    "default_strategy": "cost_optimized",
    "fallback_enabled": true,
    "max_retries": 3,
    "retry_backoff": "exponential"
  },
  "logging": {
    "level": "INFO",
    "format": "json",
    "output": "~/.local/share/swarm-mcp/logs/"
  }
}
EOF
```

### Connecting to Claude Code

```bash
# Add Swarm MCP to Claude Code
claude mcp add swarm-mcp -- python -m swarm_mcp.server

# Or manually edit ~/.claude.json
{
  "mcpServers": {
    "swarm-mcp": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "swarm_mcp.server"],
      "env": {
        "SWARM_CONFIG_PATH": "~/.config/swarm-mcp/config.json",
        "SWARM_LOG_LEVEL": "DEBUG"
      }
    }
  }
}
```

### First Test

```bash
# Start Claude Code
claude

# Inside Claude Code, test the connection
> What tools are available?
[Claude should list: swarm_spawn_agents, swarm_smart_route, swarm_quota_status, swarm_fallback_chain]

# Test a simple parallel task
> Use swarm_spawn_agents to generate Python docstrings for these functions: 
  def add(a, b): return a + b
  def subtract(a, b): return a - b
  def multiply(a, b): return a * b
```

---

## Architecture Deep Dive

### Project Structure

```
swarm-mcp/
├── src/
│   └── swarm_mcp/
│       ├── __init__.py
│       ├── server.py              # MCP server entry point
│       ├── config.py              # Configuration loading
│       ├── models.py              # Pydantic models
│       │
│       ├── core/                  # Core orchestration
│       │   ├── __init__.py
│       │   ├── router.py          # Intelligent routing
│       │   ├── engine.py          # Execution engine
│       │   ├── aggregator.py      # Result aggregation
│       │   └── quota.py           # Quota management
│       │
│       ├── providers/             # Provider adapters
│       │   ├── __init__.py
│       │   ├── base.py            # Abstract adapter
│       │   ├── kimi.py            # Moonshot AI
│       │   ├── deepseek.py        # DeepSeek AI
│       │   ├── zai.py             # Z.ai (ChatGLM)
│       │   ├── qwen.py            # Alibaba Qwen
│       │   └── openai_compatible.py  # Generic adapter
│       │
│       ├── routing/               # Routing strategies
│       │   ├── __init__.py
│       │   ├── strategies.py      # Built-in strategies
│       │   ├── classifier.py      # Task classification
│       │   └── cost_index.py      # Price tracking
│       │
│       └── utils/
│           ├── __init__.py
│           ├── logging.py         # Structured logging
│           ├── circuit_breaker.py # Fault tolerance
│           └── http.py            # HTTP client
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│
├── docs/
├── scripts/
└── pyproject.toml
```

### The Server Lifecycle

```python
# src/swarm_mcp/server.py
from mcp.server import Server
from mcp.types import TextContent, Tool

from swarm_mcp.config import load_config
from swarm_mcp.core.router import Router
from swarm_mcp.core.engine import ExecutionEngine

class SwarmMCPServer:
    def __init__(self):
        self.config = load_config()
        self.router = Router(self.config)
        self.engine = ExecutionEngine(self.config)
        self.server = Server("swarm-mcp")
        self._register_tools()
    
    def _register_tools(self):
        @self.server.list_tools()
        async def list_tools():
            return [
                Tool(
                    name="swarm_spawn_agents",
                    description="...",
                    inputSchema={...}
                ),
                # ... other tools
            ]
        
        @self.server.call_tool()
        async def call_tool(name: str, arguments: dict):
            if name == "swarm_spawn_agents":
                return await self._handle_spawn_agents(arguments)
            # ... other handlers
    
    async def _handle_spawn_agents(self, args: dict):
        # Parse and validate
        tasks = args["tasks"]
        provider_hint = args.get("provider", "auto")
        
        # Route to provider(s)
        provider = self.router.select_provider(tasks, provider_hint)
        
        # Execute
        results = await self.engine.spawn_parallel(
            tasks=tasks,
            provider=provider,
            max_concurrent=args.get("max_concurrent", 20)
        )
        
        # Aggregate and return
        aggregated = self.engine.aggregate(results, args.get("strategy"))
        return [TextContent(type="text", text=aggregated)]
    
    async def run(self):
        # MCP runs over stdio by default
        from mcp.server.stdio import stdio_server
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream,
                write_stream,
                self.server.create_initialization_options()
            )

if __name__ == "__main__":
    import asyncio
    server = SwarmMCPServer()
    asyncio.run(server.run())
```

### The Router: Decision Logic

```python
# src/swarm_mcp/core/router.py
from typing import List, Optional
from dataclasses import dataclass

from swarm_mcp.models import Task, TaskProfile, Provider, RoutingDecision

class Router:
    def __init__(self, config):
        self.providers = self._load_providers(config)
        self.cost_index = CostIndex(config)
        self.classifier = TaskClassifier()
    
    def select_provider(
        self,
        tasks: List[Task],
        hint: str = "auto",
        constraints: Optional[dict] = None
    ) -> Provider:
        """
        Select optimal provider based on task profile and current conditions.
        """
        if hint != "auto":
            return self._get_provider_by_name(hint)
        
        # Classify the task
        profile = self.classifier.analyze(tasks)
        
        # Score each provider
        scores = []
        for provider in self.providers:
            score = self._score_provider(provider, profile, constraints)
            scores.append((provider, score))
        
        # Return highest score
        best = max(scores, key=lambda x: x[1])
        return best[0]
    
    def _score_provider(
        self,
        provider: Provider,
        profile: TaskProfile,
        constraints: dict
    ) -> float:
        score = 0.0
        
        # Capability match (0-40 points)
        score += self._capability_match(provider, profile) * 40
        
        # Cost efficiency (0-30 points)
        estimated_cost = self.cost_index.estimate(provider, profile)
        score += self._cost_score(estimated_cost, constraints) * 30
        
        # Availability (0-20 points)
        quota_status = provider.get_quota_status()
        score += self._availability_score(quota_status) * 20
        
        # Latency (0-10 points)
        score += self._latency_score(provider.latency_ms) * 10
        
        return score
    
    def _capability_match(self, provider: Provider, profile: TaskProfile) -> float:
        """Return 0-1 score based on capability alignment."""
        required = profile.required_capabilities
        available = set(provider.capabilities)
        
        matches = len(required & available)
        total = len(required)
        
        # Bonus for "parallel" capability if task count > 5
        if profile.task_count > 5 and "parallel" in available:
            matches += 0.5
        
        return min(matches / total, 1.0) if total > 0 else 1.0
```

### The Execution Engine: Parallel Orchestration

```python
# src/swarm_mcp/core/engine.py
import asyncio
from typing import List, AsyncIterator
from dataclasses import dataclass

@dataclass
class AgentResult:
    task_id: str
    status: str  # "success", "error", "timeout"
    content: str
    tokens_used: int
    cost: float
    latency_ms: int

class ExecutionEngine:
    def __init__(self, config):
        self.config = config
        self.semaphores = {
            name: asyncio.Semaphore(p.max_parallel)
            for name, p in config.providers.items()
        }
    
    async def spawn_parallel(
        self,
        tasks: List[str],
        provider: Provider,
        max_concurrent: int
    ) -> List[AgentResult]:
        """
        Spawn up to max_concurrent agents in parallel.
        """
        # Create semaphore for this specific operation
        # (may be lower than provider's global limit)
        semaphore = asyncio.Semaphore(max_concurrent)
        
        # Create agent tasks
        agent_tasks = [
            self._run_agent(task, provider, semaphore, i)
            for i, task in enumerate(tasks)
        ]
        
        # Execute with gather (true parallelism)
        results = await asyncio.gather(*agent_tasks, return_exceptions=True)
        
        # Convert exceptions to error results
        processed = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                processed.append(AgentResult(
                    task_id=f"task_{i}",
                    status="error",
                    content=str(result),
                    tokens_used=0,
                    cost=0,
                    latency_ms=0
                ))
            else:
                processed.append(result)
        
        return processed
    
    async def _run_agent(
        self,
        task: str,
        provider: Provider,
        semaphore: asyncio.Semaphore,
        task_id: int
    ) -> AgentResult:
        """
        Run single agent with resource constraints.
        """
        async with semaphore:  # Acquire slot
            start_time = asyncio.get_event_loop().time()
            
            try:
                # Call provider API
                response = await provider.execute(task)
                
                # Calculate metrics
                latency_ms = int(
                    (asyncio.get_event_loop().time() - start_time) * 1000
                )
                cost = provider.calculate_cost(response.usage)
                
                return AgentResult(
                    task_id=f"task_{task_id}",
                    status="success",
                    content=response.content,
                    tokens_used=response.usage.total_tokens,
                    cost=cost,
                    latency_ms=latency_ms
                )
                
            except asyncio.TimeoutError:
                return AgentResult(
                    task_id=f"task_{task_id}",
                    status="timeout",
                    content="",
                    tokens_used=0,
                    cost=0,
                    latency_ms=self.config.timeout * 1000
                )
```

---

## Building Provider Adapters

### The Base Adapter Interface

```python
# src/swarm_mcp/providers/base.py
from abc import ABC, abstractmethod
from typing import AsyncIterator, Optional
from dataclasses import dataclass

@dataclass
class TokenUsage:
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int

@dataclass
class ProviderResponse:
    content: str
    usage: TokenUsage
    model: str
    finish_reason: str

@dataclass
class QuotaStatus:
    remaining_requests: Optional[int]
    remaining_tokens: Optional[int]
    reset_time: Optional[str]
    is_healthy: bool

class ProviderAdapter(ABC):
    """
    Abstract base for all provider adapters.
    Implement this to add a new AI provider to Swarm.
    """
    
    def __init__(self, config: dict):
        self.config = config
        self.name = config["name"]
        self.max_parallel = config["max_parallel"]
        self.timeout = config["timeout"]
        self._client = None  # Initialize in connect()
    
    @abstractmethod
    async def connect(self) -> None:
        """Initialize HTTP client and verify connectivity."""
        pass
    
    @abstractmethod
    async def execute(self, task: str, context: Optional[dict] = None) -> ProviderResponse:
        """
        Execute task and return complete response.
        For streaming, accumulate and return full content.
        """
        pass
    
    @abstractmethod
    async def execute_stream(
        self,
        task: str,
        context: Optional[dict] = None
    ) -> AsyncIterator[str]:
        """
        Execute task and yield tokens as they arrive.
        Used for real-time feedback to user.
        """
        pass
    
    @abstractmethod
    async def get_quota_status(self) -> QuotaStatus:
        """Return current quota availability."""
        pass
    
    @abstractmethod
    def calculate_cost(self, usage: TokenUsage) -> float:
        """Calculate $ cost for given token usage."""
        pass
    
    @abstractmethod
    def estimate_cost(self, input_tokens: int, output_tokens: int) -> float:
        """Estimate cost before execution."""
        pass
    
    async def health_check(self) -> bool:
        """
        Verify provider is responsive.
        Default implementation tries quota check.
        """
        try:
            status = await self.get_quota_status()
            return status.is_healthy
        except Exception:
            return False
```

### Implementing a New Provider: Example (MiniMax)

```python
# src/swarm_mcp/providers/minimax.py
import aiohttp
from typing import AsyncIterator

from swarm_mcp.providers.base import (
    ProviderAdapter, ProviderResponse, TokenUsage, QuotaStatus
)

class MiniMaxAdapter(ProviderAdapter):
    """
    Adapter for MiniMax AI (稀宇科技).
    API Docs: https://www.minimaxi.com/document
    """
    
    def __init__(self, config):
        super().__init__(config)
        self.base_url = config.get("base_url", "https://api.minimaxi.chat/v1")
        self.model = config.get("model", "abab6.5-chat")
        self.group_id = config["group_id"]  # MiniMax uses group IDs
        self.api_key = config["api_key"]
    
    async def connect(self):
        self._client = aiohttp.ClientSession(
            headers={
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            },
            timeout=aiohttp.ClientTimeout(total=self.timeout)
        )
        # Verify connection
        await self.health_check()
    
    async def execute(self, task: str, context=None) -> ProviderResponse:
        payload = {
            "model": self.model,
            "messages": [
                {"role": "system", "content": self._build_system_prompt(context)},
                {"role": "user", "content": task}
            ],
            "tokens_to_generate": 2048,
            "temperature": 0.1,
        }
        
        async with self._client.post(
            f"{self.base_url}/text/chatcompletion_v2",
            json=payload
        ) as resp:
            data = await resp.json()
            
            if data.get("base_resp", {}).get("status_code") != 0:
                raise ProviderError(f"MiniMax error: {data}")
            
            choice = data["reply"]
            usage = data["usage"]
            
            return ProviderResponse(
                content=choice,
                usage=TokenUsage(
                    prompt_tokens=usage["prompt_tokens"],
                    completion_tokens=usage["completion_tokens"],
                    total_tokens=usage["total_tokens"]
                ),
                model=self.model,
                finish_reason="stop"
            )
    
    async def execute_stream(self, task: str, context=None) -> AsyncIterator[str]:
        payload = {
            "model": self.model,
            "messages": [
                {"role": "user", "content": task}
            ],
            "stream": True,
            "tokens_to_generate": 2048,
        }
        
        async with self._client.post(
            f"{self.base_url}/text/chatcompletion_v2",
            json=payload
        ) as resp:
            async for line in resp.content:
                if line.startswith(b"data: "):
                    data = json.loads(line[6:])
                    if "choices" in data:
                        yield data["choices"][0]["delta"]["content"]
    
    async def get_quota_status(self) -> QuotaStatus:
        # MiniMax doesn't have a direct quota endpoint
        # We estimate based on recent usage patterns
        return QuotaStatus(
            remaining_requests=None,  # Unknown
            remaining_tokens=None,    # Unknown
            reset_time=None,
            is_healthy=await self.health_check()
        )
    
    def calculate_cost(self, usage: TokenUsage) -> float:
        # MiniMax pricing (example rates)
        input_cost = (usage.prompt_tokens / 1_000_000) * 0.015
        output_cost = (usage.completion_tokens / 1_000_000) * 0.015
        return round(input_cost + output_cost, 6)
    
    def _build_system_prompt(self, context):
        default = "You are a helpful coding assistant."
        if not context:
            return default
        return context.get("system_prompt", default)
```

### Registration

Add your adapter to the provider registry:

```python
# src/swarm_mcp/providers/__init__.py
from swarm_mcp.providers.kimi import KimiAdapter
from swarm_mcp.providers.deepseek import DeepSeekAdapter
from swarm_mcp.providers.zai import ZaiAdapter
from swarm_mcp.providers.qwen import QwenAdapter
from swarm_mcp.providers.minimax import MiniMaxAdapter  # Your new adapter

PROVIDER_REGISTRY = {
    "kimi": KimiAdapter,
    "deepseek": DeepSeekAdapter,
    "zai": ZaiAdapter,
    "qwen": QwenAdapter,
    "minimax": MiniMaxAdapter,  # Register here
}

def load_provider(name: str, config: dict):
    if name not in PROVIDER_REGISTRY:
        raise ValueError(f"Unknown provider: {name}. "
                        f"Available: {list(PROVIDER_REGISTRY.keys())}")
    return PROVIDER_REGISTRY[name](config)
```

---

## Extending the Router

### Creating Custom Routing Strategies

```python
# src/swarm_mcp/routing/strategies.py
from abc import ABC, abstractmethod
from typing import List
from swarm_mcp.models import Task, Provider, RoutingContext

class RoutingStrategy(ABC):
    @abstractmethod
    def select(
        self,
        tasks: List[Task],
        providers: List[Provider],
        context: RoutingContext
    ) -> Provider:
        pass

class CostOptimizedStrategy(RoutingStrategy):
    """
    Minimize cost while meeting quality bar.
    Default strategy for most operations.
    """
    
    def select(self, tasks, providers, context):
        # Filter providers that can handle this task type
        capable = [
            p for p in providers
            if p.has_capabilities(tasks[0].required_capabilities)
        ]
        
        if not capable:
            raise NoProviderAvailable("No provider has required capabilities")
        
        # Score by cost
        def cost_score(provider):
            estimated = provider.estimate_cost(
                input_tokens=sum(t.estimated_input for t in tasks),
                output_tokens=sum(t.estimated_output for t in tasks)
            )
            # Normalize by capability (cheaper but incapable = bad deal)
            capability_bonus = len(provider.capabilities) * 0.1
            return estimated / (1 + capability_bonus)
        
        return min(capable, key=cost_score)

class SpeedDemonStrategy(RoutingStrategy):
    """
    Maximize parallelization, cost be damned.
    For urgent bulk operations.
    """
    
    def select(self, tasks, providers, context):
        # Prioritize max_parallel over everything
        best_parallel = max(providers, key=lambda p: p.max_parallel)
        
        # But ensure it has basic capability
        if not best_parallel.has_capabilities(["coding"]):
            # Fall back to next best
            candidates = [
                p for p in providers
                if p.has_capabilities(["coding"])
            ]
            return max(candidates, key=lambda p: p.max_parallel)
        
        return best_parallel

class QualityFirstStrategy(RoutingStrategy):
    """
    Select highest quality provider regardless of cost.
    For critical security or architecture decisions.
    """
    
    QUALITY_RANKING = {
        "claude": 10,
        "kimi": 8,
        "deepseek": 7,
        "qwen": 6,
        "zai": 5,
    }
    
    def select(self, tasks, providers, context):
        # Score by quality ranking
        def quality_score(provider):
            base = self.QUALITY_RANKING.get(provider.name, 5)
            
            # Boost for reasoning capability if task needs it
            if tasks[0].needs_reasoning and "reasoning" in provider.capabilities:
                base += 2
            
            # Penalty for low quota
            quota = provider.get_quota_status()
            if quota.remaining_tokens and quota.remaining_tokens < 10000:
                base -= 3
            
            return base
        
        return max(providers, key=quality_score)

# Register in router
STRATEGIES = {
    "cost_optimized": CostOptimizedStrategy,
    "speed_demon": SpeedDemonStrategy,
    "quality_first": QualityFirstStrategy,
}
```

### Adding Task Classification

```python
# src/swarm_mcp/routing/classifier.py
import re
from dataclasses import dataclass
from typing import Set

@dataclass
class TaskProfile:
    complexity: str  # "low", "medium", "high", "critical"
    domain: str      # "coding", "math", "writing", "analysis"
    safety_critical: bool
    required_capabilities: Set[str]
    estimated_tokens: int
    parallelizable: bool

class TaskClassifier:
    """
    Analyze task description to determine routing characteristics.
    """
    
    # Patterns for classification
    COMPLEXITY_PATTERNS = {
        "critical": [
            r"security", r"crypto", r"authentication", r"password",
            r"payment", r"encrypt", r"vulnerability"
        ],
        "high": [
            r"architecture", r"design", r"algorithm", r"optimize",
            r"refactor.*entire", r"migrate.*framework"
        ],
        "medium": [
            r"implement", r"feature", r"api", r"endpoint",
            r"component", r"module"
        ],
        "low": [
            r"fix.*typo", r"update.*doc", r"rename", r"format",
            r"lint", r"style"
        ]
    }
    
    DOMAIN_PATTERNS = {
        "coding": [r"function", r"class", r"def ", r"import", r"refactor"],
        "math": [r"calculate", r"algorithm", r"equation", r"optimize"],
        "writing": [r"document", r"explain", r"describe", r"readme"],
        "analysis": [r"review", r"audit", r"analyze", r"compare"]
    }
    
    def analyze(self, tasks: List[str]) -> TaskProfile:
        combined = " ".join(tasks).lower()
        
        # Determine complexity
        complexity = "medium"  # default
        for level, patterns in self.COMPLEXITY_PATTERNS.items():
            if any(re.search(p, combined) for p in patterns):
                complexity = level
                break  # Take first (most critical) match
        
        # Determine domain
        domain = "coding"  # default
        for d, patterns in self.DOMAIN_PATTERNS.items():
            if any(re.search(p, combined) for p in patterns):
                domain = d
                break
        
        # Safety critical?
        safety_critical = complexity == "critical"
        
        # Required capabilities
        required = {"coding"}  # baseline
        if complexity in ["high", "critical"]:
            required.add("reasoning")
        if len(tasks) > 5:
            required.add("parallel")
        if safety_critical:
            required.add("long_context")  # For thorough analysis
        
        # Estimate tokens (rough heuristic)
        estimated = len(combined) // 4  # ~4 chars per token
        
        # Parallelizable?
        parallelizable = len(tasks) > 1 and not any(
            re.search(r"depend|sequence|order|step.*step", combined)
        )
        
        return TaskProfile(
            complexity=complexity,
            domain=domain,
            safety_critical=safety_critical,
            required_capabilities=required,
            estimated_tokens=estimated,
            parallelizable=parallelizable
        )
```

---

## Testing & Debugging

### Unit Testing

```python
# tests/unit/test_router.py
import pytest
from unittest.mock import AsyncMock, MagicMock

from swarm_mcp.core.router import Router
from swarm_mcp.routing.classifier import TaskClassifier

@pytest.fixture
def mock_config():
    return {
        "providers": {
            "kimi": {
                "max_parallel": 100,
                "capabilities": ["coding", "parallel", "long_context"],
                "cost_per_1m_input": 0.60,
            },
            "deepseek": {
                "max_parallel": 50,
                "capabilities": ["coding", "reasoning"],
                "cost_per_1m_input": 0.28,
            }
        }
    }

@pytest.fixture
def mock_providers(mock_config):
    providers = {}
    for name, conf in mock_config["providers"].items():
        p = AsyncMock()
        p.name = name
        p.max_parallel = conf["max_parallel"]
        p.capabilities = conf["capabilities"]
        p.estimate_cost.return_value = conf["cost_per_1m_input"]
        providers[name] = p
    return providers

@pytest.mark.asyncio
async def test_router_selects_cheaper_for_simple_task(mock_config, mock_providers):
    router = Router(mock_config)
    router.providers = list(mock_providers.values())
    
    # Simple task should prefer DeepSeek (cheaper)
    tasks = ["def add(a, b): return a + b"]
    
    # Mock classifier to return simple profile
    router.classifier.analyze = MagicMock(return_value=MagicMock(
        complexity="low",
        required_capabilities={"coding"},
        task_count=1
    ))
    
    selected = router.select_provider(tasks)
    assert selected.name == "deepseek"

@pytest.mark.asyncio
async def test_router_selects_kimi_for_parallel_bulk(mock_config, mock_providers):
    router = Router(mock_config)
    router.providers = list(mock_providers.values())
    
    # Bulk task should prefer Kimi (more parallel)
    tasks = [f"Task {i}" for i in range(50)]
    
    router.classifier.analyze = MagicMock(return_value=MagicMock(
        complexity="medium",
        required_capabilities={"coding", "parallel"},
        task_count=50
    ))
    
    selected = router.select_provider(tasks)
    assert selected.name == "kimi"
```

### Integration Testing

```python
# tests/integration/test_end_to_end.py
import pytest
import asyncio
from swarm_mcp.server import SwarmMCPServer

@pytest.mark.integration
@pytest.mark.asyncio
async def test_spawn_agents_integration():
    """
    Test full flow: server receives MCP request,
    routes to provider (mocked), returns aggregated result.
    """
    # This would use real config but mock provider HTTP calls
    server = SwarmMCPServer()
    
    # Mock the provider execution
    server.engine.spawn_parallel = AsyncMock(return_value=[
        MagicMock(content="Result 1", status="success"),
        MagicMock(content="Result 2", status="success"),
    ])
    
    result = await server._handle_spawn_agents({
        "tasks": ["Task 1", "Task 2"],
        "provider": "kimi",
        "max_concurrent": 2
    })
    
    assert len(result) == 1
    assert "Result 1" in result[0].text
    assert "Result 2" in result[0].text
```

### Debugging Tools

```python
# src/swarm_mcp/utils/debug.py
import json
import asyncio
from typing import Any
from datetime import datetime

class SwarmDebugger:
    """
    Development utilities for inspecting Swarm behavior.
    """
    
    def __init__(self, server):
        self.server = server
        self.trace_log = []
    
    async def trace_request(self, request_id: str, tool_name: str, arguments: Any):
        """Log request lifecycle for debugging."""
        trace = {
            "request_id": request_id,
            "tool": tool_name,
            "arguments": arguments,
            "start_time": datetime.utcnow().isoformat(),
            "stages": []
        }
        
        # Hook into router decision
        original_select = self.server.router.select_provider
        async def traced_select(tasks, hint):
            provider = await original_select(tasks, hint)
            trace["stages"].append({
                "stage": "routing",
                "selected_provider": provider.name,
                "rationale": getattr(provider, '_selection_rationale', 'unknown')
            })
            return provider
        
        self.server.router.select_provider = traced_select
        
        # Hook into execution
        # ... similar for engine
        
        self.trace_log.append(trace)
        return trace
    
    def dump_traces(self, filename: str = None):
        """Export all traces for analysis."""
        output = json.dumps(self.trace_log, indent=2)
        if filename:
            with open(filename, 'w') as f:
                f.write(output)
        return output
    
    def analyze_performance(self):
        """Calculate statistics from traces."""
        stats = {
            "total_requests": len(self.trace_log),
            "avg_routing_time_ms": 0,
            "provider_distribution": {},
            "error_rate": 0
        }
        
        for trace in self.trace_log:
            provider = trace["stages"][0]["selected_provider"]
            stats["provider_distribution"][provider] = \
                stats["provider_distribution"].get(provider, 0) + 1
        
        return stats
```

### Running the Test Suite

```bash
# Unit tests only (fast, no external calls)
pytest tests/unit -v

# Integration tests (requires API keys in .env.test)
pytest tests/integration -v --timeout=120

# With coverage
pytest --cov=swarm_mcp --cov-report=html

# Specific test
pytest tests/unit/test_router.py::test_router_selects_cheaper -v -s

# Debug mode (verbose logging)
SWARM_LOG_LEVEL=DEBUG pytest tests/ -v -s
```

---

## Performance Tuning

### Benchmarking

```python
# scripts/benchmark.py
import asyncio
import time
import statistics
from dataclasses import dataclass
from typing import List

@dataclass
class BenchmarkResult:
    provider: str
    task_count: int
    total_time: float
    avg_latency: float
    p99_latency: float
    total_cost: float
    success_rate: float

async def benchmark_provider(
    provider_name: str,
    task: str,
    count: int,
    max_concurrent: int
) -> BenchmarkResult:
    """
    Benchmark a provider with specific workload.
    """
    # Load provider
    from swarm_mcp.providers import load_provider
    from swarm_mcp.config import load_config
    
    config = load_config()
    provider = load_provider(provider_name, config.providers[provider_name])
    await provider.connect()
    
    # Prepare tasks
    tasks = [task for _ in range(count)]
    
    # Execute with timing
    start = time.perf_counter()
    latencies = []
    
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def timed_execution(t):
        async with semaphore:
            t0 = time.perf_counter()
            try:
                result = await provider.execute(t)
                latency = time.perf_counter() - t0
                latencies.append(latency)
                return result
            except Exception as e:
                latencies.append(time.perf_counter() - t0)
                raise e
    
    results = await asyncio.gather(
        *[timed_execution(t) for t in tasks],
        return_exceptions=True
    )
    
    total_time = time.perf_counter() - start
    
    # Calculate stats
    successes = sum(1 for r in results if not isinstance(r, Exception))
    success_rate = successes / count
    
    if latencies:
        avg_latency = statistics.mean(latencies)
        p99_latency = sorted(latencies)[int(len(latencies) * 0.99)]
    else:
        avg_latency = p99_latency = 0
    
    # Calculate cost
    total_tokens = sum(
        r.usage.total_tokens for r in results
        if not isinstance(r, Exception)
    )
    total_cost = provider.calculate_cost(
        type('Usage', (), {
            'prompt_tokens': total_tokens // 2,
            'completion_tokens': total_tokens // 2,
            'total_tokens': total_tokens
        })()
    )
    
    return BenchmarkResult(
        provider=provider_name,
        task_count=count,
        total_time=total_time,
        avg_latency=avg_latency,
        p99_latency=p99_latency,
        total_cost=total_cost,
        success_rate=success_rate
    )

# Run comparison
async def main():
    task = "Generate a Python function to calculate fibonacci numbers"
    
    for provider in ["kimi", "deepseek", "zai"]:
        result = await benchmark_provider(provider, task, count=50, max_concurrent=20)
        print(f"\n{provider}:")
        print(f"  50 tasks in {result.total_time:.2f}s")
        print(f"  Avg latency: {result.avg_latency*1000:.0f}ms")
        print(f"  P99 latency: {result.p99_latency*1000:.0f}ms")
        print(f"  Total cost: ${result.total_cost:.4f}")
        print(f"  Success rate: {result.success_rate*100:.1f}%")

if __name__ == "__main__":
    asyncio.run(main())
```

### Optimization Strategies

#### 1. Connection Pooling

```python
# In provider adapter
import aiohttp

class OptimizedProvider(ProviderAdapter):
    def __init__(self, config):
        super().__init__(config)
        # Share session across requests
        self._session = None
    
    async def connect(self):
        # TCP connection pooling
        connector = aiohttp.TCPConnector(
            limit=100,           # Max concurrent connections
            limit_per_host=50,   # Per endpoint
            enable_cleanup_closed=True,
            force_close=False,   # Keep-alive
        )
        
        timeout = aiohttp.ClientTimeout(
            total=self.timeout,
            connect=5,
            sock_read=30
        )
        
        self._session = aiohttp.ClientSession(
            connector=connector,
            timeout=timeout,
            headers=self._default_headers()
        )
    
    async def execute(self, task):
        # Reuse session (connection pooling)
        async with self._session.post(...) as resp:
            # ...
```

#### 2. Batching

```python
# For providers that support batching
async def execute_batch(self, tasks: List[str]) -> List[ProviderResponse]:
    """
    Execute multiple tasks in single request if provider supports it.
    """
    if not self.supports_batching:
        # Fall back to individual requests with gather
        return await asyncio.gather(*[self.execute(t) for t in tasks])
    
    # Batch API call
    payload = {
        "batch": [
            {"custom_id": f"task_{i}", "body": {"messages": [{"role": "user", "content": t}]}}
            for i, t in enumerate(tasks)
        ]
    }
    
    async with self._session.post(f"{self.base_url}/batch", json=payload) as resp:
        data = await resp.json()
        return [self._parse_batch_item(item) for item in data["results"]]
```

#### 3. Caching

```python
# For identical or similar tasks
from functools import lru_cache
import hashlib

class CachingProvider(ProviderAdapter):
    def __init__(self, config):
        super().__init__(config)
        self._cache = {}  # In production, use Redis
    
    def _get_cache_key(self, task: str) -> str:
        return hashlib.sha256(task.encode()).hexdigest()[:16]
    
    async def execute(self, task: str, use_cache=True):
        if use_cache:
            key = self._get_cache_key(task)
            if key in self._cache:
                return self._cache[key]
        
        result = await super().execute(task)
        
        if use_cache:
            self._cache[key] = result
        
        return result
```

#### 4. Adaptive Concurrency

```python
class AdaptiveEngine(ExecutionEngine):
    def __init__(self, config):
        super().__init__(config)
        self.latency_history = []
        self.error_rate = 0.0
    
    def get_optimal_concurrency(self) -> int:
        """
        Adjust concurrency based on recent performance.
        """
        base_limit = self.config.max_concurrent
        
        # If errors are high, reduce concurrency
        if self.error_rate > 0.1:
            return max(1, base_limit // 4)
        
        # If latency is spiking, reduce slightly
        if self.latency_history:
            recent_avg = statistics.mean(self.latency_history[-10:])
            if recent_avg > 5.0:  # 5 seconds
                return max(1, base_limit // 2)
        
        return base_limit
    
    async def spawn_parallel(self, tasks, provider, max_concurrent):
        # Use adaptive limit
        actual_concurrency = min(
            max_concurrent,
            self.get_optimal_concurrency(),
            provider.max_parallel
        )
        
        return await super().spawn_parallel(
            tasks, provider, actual_concurrency
        )
```

---

## Contributing

### Development Workflow

```bash
# 1. Fork and clone
git clone https://github.com/YOUR_USERNAME/swarm-mcp.git
cd swarm-mcp

# 2. Set up development environment
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# 3. Create branch
git checkout -b feature/your-feature-name

# 4. Make changes, run tests
pytest tests/unit -v

# 5. Lint and format
ruff check src/
ruff format src/
mypy src/

# 6. Commit with conventional commit
git commit -m "feat: add MiniMax provider adapter"

# 7. Push and PR
git push origin feature/your-feature-name
# Open PR with detailed description
```

### Commit Convention

We use [Conventional Commits](https://www.conventionalcommits.org/):

| Type | Use For | Example |
|------|---------|---------|
| `feat` | New features | `feat: add swarm_fallback_chain tool` |
| `fix` | Bug fixes | `fix: handle timeout in Kimi adapter` |
| `docs` | Documentation | `docs: add troubleshooting guide` |
| `test` | Tests | `test: add router unit tests` |
| `refactor` | Code restructuring | `refactor: extract base adapter class` |
| `perf` | Performance | `perf: add connection pooling` |
| `chore` | Maintenance | `chore: update dependencies` |

### Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing performed

## Checklist
- [ ] Code follows style guide (ruff, mypy)
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings

## Provider Changes (if applicable)
- [ ] Adapter implements all abstract methods
- [ ] Quota tracking implemented
- [ ] Cost calculation verified
- [ ] Test credentials provided to maintainers
```

### Code Review Process

1. **Automated checks** must pass (CI/CD)
2. **Two approvals** required from maintainers
3. **House-appropriate review**: Router changes need House of Rout, Provider changes need relevant house
4. **Documentation**: Must update docs for user-facing changes

---

## API Reference

### MCP Tools

#### `swarm_spawn_agents`

Spawn multiple agents in parallel to execute independent tasks.

**Input Schema:**
```json
{
  "tasks": ["string"],           // Required: 1-1000 tasks
  "provider": "string",          // Optional: "kimi", "deepseek", "zai", "qwen", "auto"
  "max_concurrent": "integer",   // Optional: 1-100, default 20
  "aggregation_strategy": "string",  // Optional: "concatenate", "summarize", "merge_code", "json_merge"
  "timeout": "integer"           // Optional: seconds per agent, default 60
}
```

**Example Usage (from Claude Code):**
```
Use swarm_spawn_agents to generate unit tests for these functions:
- def add(a, b): return a + b
- def subtract(a, b): return a - b  
- def multiply(a, b): return a * b
- def divide(a, b): return a / b if b != 0 else raise ValueError

Use Kimi provider with max 10 concurrent agents.
```

**Response:**
```
Agent 1 result:
def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
---
Agent 2 result:
def test_subtract():
    assert subtract(5, 3) == 2
    assert subtract(0, 5) == -5
---
[...]
```

#### `swarm_smart_route`

Execute single task with automatic provider selection.

**Input Schema:**
```json
{
  "task": "string",              // Required: task description
  "complexity_hint": "string",   // Optional: "low", "medium", "high", "critical"
  "budget_ceiling": "number",    // Optional: max USD cost
  "latency_priority": "boolean"   // Optional: prioritize speed over cost
}
```

#### `swarm_quota_status`

Check current quota availability across all providers.

**Response:**
```json
{
  "kimi": {
    "remaining_requests": 8500,
    "reset_in": "2h 15m",
    "status": "healthy"
  },
  "deepseek": {
    "remaining_tokens": "unlimited",
    "status": "healthy"
  },
  "zai": {
    "remaining_requests": 1200,
    "reset_in": "45m",
    "status": "warning"
  }
}
```

#### `swarm_fallback_chain`

Execute with explicit fallback providers.

**Input Schema:**
```json
{
  "task": "string",
  "chain": ["kimi", "deepseek", "zai", "claude"],
  "timeout_per_provider": 30
}
```

---

## Troubleshooting

### Common Issues

#### "No provider available"

**Cause**: All providers exhausted quota or failed health checks.

**Resolution**:
```bash
# Check status
claude
> Check swarm quota status

# Add more providers to config
# Or wait for quota reset
```

#### "Timeout waiting for agents"

**Cause**: Tasks too complex for timeout, or provider degraded.

**Resolution**:
- Increase timeout: `{"timeout": 120}`
- Reduce max_concurrent to avoid overwhelming provider
- Check provider status page

#### "Cost higher than expected"

**Cause**: Router selected more expensive provider due to constraints.

**Resolution**:
- Explicitly specify cheaper provider: `{"provider": "deepseek"}`
- Remove constraints that force expensive routing
- Check cost index: `docs/economics/prices.md`

#### "Claude Code doesn't see tools"

**Resolution**:
```bash
# Restart Claude Code
claude config set mcpServers '{}'
claude mcp add swarm-mcp -- python -m swarm_mcp.server

# Verify server starts without errors
python -m swarm_mcp.server --verbose
```

### Debug Mode

Enable detailed logging:

```bash
export SWARM_LOG_LEVEL=DEBUG
export SWARM_LOG_FORMAT=json
export SWARM_LOG_OUTPUT=~/.local/share/swarm-mcp/logs/

python -m swarm_mcp.server
```

### Getting Help

| Channel | Best For |
|---------|----------|
| GitHub Issues | Bug reports, feature requests |
| Discord `#help` | Quick questions, troubleshooting |
| Discord `#dev` | Architecture discussions |
| Email security@swarm-mcp.dev | Security vulnerabilities |

---

<div align="center">

## Build the Swarm

**Documentation is never finished. Code is never perfect. The work continues.**

[Contribute on GitHub](https://github.com/yourusername/swarm-mcp) • [Join the Community](https://discord.gg/swarm-mcp)

</div>

---

This Developer Guide provides comprehensive technical depth while maintaining the purposeful, quality-focused tone from the Guild document. It covers:

1. **Immediate utility**: Getting started, configuration, first tests
2. **Deep technical understanding**: Architecture, router logic, execution engine
3. **Extensibility**: How to add providers, routing strategies, task classifiers
4. **Quality assurance**: Testing at multiple levels, debugging tools
5. **Performance**: Benchmarking, optimization strategies, adaptive systems
6. **Contribution mechanics**: Workflow, conventions, review process
7. **Operational reference**: API docs, troubleshooting, debugging

The structure allows readers to dive deep where needed while providing clear paths for different user types.
