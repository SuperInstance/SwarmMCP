
<div align="center">

# Swarm Architecture Specification
## *Design for the User, Build for Reality*

**Version 2.0 | The Definitive Technical Reference**

</div>

---

## Executive Summary

After building the user guide and developer documentation, we now understand what Swarm must actually be: **not a complex orchestration framework, but a transparent bridge that disappears into the workflow.**

The architecture follows three inviolable principles:

1. **The User Doesn't Configure—They Direct.** All complexity is abstracted behind natural language. The system infers intent.
2. **Failure Is Data, Not Exception.** Every provider failure is an automatic routing signal. The user never sees "rate limit"—they see "switching to backup."
3. **Cost Is a Feature, Not a Metric.** Every operation displays estimated cost before execution. Optimization is automatic, transparent, and overrideable.

---

## The Core Insight: What We Actually Built

### The Anti-Architecture

Most "multi-agent systems" are over-engineered. They feature:
- Complex DAG definitions
- Explicit agent topologies
- Workflow orchestration languages
- State management systems

**Swarm rejects this.** The user has Claude Code. Claude is the orchestrator. Swarm is the **execution layer**—a smart load balancer that happens to speak natural language.

### The True Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CLAUDE CODE (The User)                   │
│                                                              │
│  "Refactor these 50 files"                                    │
│         ↓                                                    │
│  [Claude decides: This is parallelizable]                    │
│         ↓                                                    │
│  "I'll use swarm_spawn_agents..."                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ MCP stdio
┌─────────────────────────────────────────────────────────────┐
│                    SWARM MCP SERVER                          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  TOOL INTERFACE (MCP Protocol)                          │ │
│  │                                                          │ │
│  │  swarm_spawn_agents(tasks, hint) ──→                   │ │
│  │      ┌─────────────────────────────────────────┐       │ │
│  │      │  1. INTENT CLASSIFIER (What user wants)  │       │ │
│  │      │     • Bulk work? → Parallelize            │       │ │
│  │      │     • Complex? → Route to best reasoner    │       │ │
│  │      │     • Critical? → Fallback chain           │       │ │
│  │      └─────────────────────────────────────────┘       │ │
│  │                          ↓                             │ │
│  │      ┌─────────────────────────────────────────┐       │ │
│  │      │  2. PROVIDER SELECTOR (Where to run)     │       │ │
│  │      │     • Check quotas (real-time)           │       │ │
│  │      │     • Score by capability/cost/latency   │       │ │
│  │      │     • Apply user hint if provided        │       │ │
│  │      └─────────────────────────────────────────┘       │ │
│  │                          ↓                             │ │
│  │      ┌─────────────────────────────────────────┐       │ │
│  │      │  3. EXECUTION ENGINE (Run it)          │       │ │
│  │      │     • Spawn N agents (asyncio gather)  │       │ │
│  │      │     • Stream results (if supported)    │       │ │
│  │      │     • Aggregate (concat/summarize/merge) │       │ │
│  │      └─────────────────────────────────────────┘       │ │
│  │                          ↓                             │ │
│  │      ┌─────────────────────────────────────────┐       │ │
│  │      │  4. RESILIENCE (Handle failure)         │       │ │
│  │      │     • Retry with backoff                 │       │ │
│  │      │     • Provider fallback                  │       │ │
│  │      │     • Partial result delivery            │       │ │
│  │      └─────────────────────────────────────────┘       │ │
│  │                                                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                              │                               │
│                              ▼ HTTP/HTTPS                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              PROVIDER ADAPTER LAYER                    │ │
│  │                                                          │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │ │
│  │  │  Kimi   │ │DeepSeek │ │  Z.ai   │ │  Qwen   │     │ │
│  │  │ Adapter │ │ Adapter │ │ Adapter │ │ Adapter │     │ │
│  │  │(100 max)│ │(50 max) │ │(20 max) │ │(30 max) │     │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘     │ │
│  │                                                          │ │
│  │  Common Interface:                                       │ │
│  │  • execute(task) → Response                            │ │
│  │  • stream(task) → Iterator[Token]                      │ │
│  │  • quota() → Status                                     │ │
│  │  • cost(tokens) → USD                                   │ │
│  │                                                          │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Key realization:** The entire system is **four decisions** (classify, select, execute, recover) and **one abstraction** (the provider adapter). Everything else is optimization.

---

## The Four Subsystems

### 1. Intent Classifier: Understanding the User

**Purpose:** Determine what the user actually wants from natural language.

**Input:** Tool name + arguments (from MCP)  
**Output:** Execution profile (routing hints)

**Logic:**
```python
class ExecutionProfile:
    parallel_preference: float  # 0.0-1.0, likelihood user wants speed
    quality_sensitivity: float  # 0.0-1.0, tolerance for errors
    cost_sensitivity: float     # 0.0-1.0, budget consciousness
    deadline_urgency: float     # 0.0-1.0, time pressure
    fallback_required: bool       # True if user emphasized "must complete"

def classify_intent(tool: str, args: dict) -> ExecutionProfile:
    """
    Infer user priorities from invocation patterns.
    """
    profile = ExecutionProfile()
    
    # Parallel preference signals
    if "bulk" in args.get("tasks", []) or len(args.get("tasks", [])) > 10:
        profile.parallel_preference = 0.9
    
    if "parallel" in str(args.get("max_concurrent", "")):
        profile.parallel_preference = 0.95
    
    # Quality sensitivity signals
    if any(word in str(args) for word in ["security", "auth", "crypto", "payment"]):
        profile.quality_sensitivity = 0.95
        profile.fallback_required = True
    
    if "critical" in str(args.get("complexity_hint", "")):
        profile.quality_sensitivity = 0.9
    
    # Cost sensitivity signals
    if "budget" in str(args) or "cheap" in str(args):
        profile.cost_sensitivity = 0.9
    
    # Deadline signals
    if "urgent" in str(args) or "asap" in str(args):
        profile.deadline_urgency = 0.9
    
    return profile
```

**Why this matters:** The system doesn't ask users to configure routing strategies. It **infers** from language.

---

### 2. Provider Selector: The Decision Engine

**Purpose:** Choose where to run based on intent, availability, and economics.

**Algorithm:**

```python
def select_provider(
    profile: ExecutionProfile,
    providers: List[Provider],
    task_estimate: TaskEstimate
) -> Selection:
    """
    Score all providers, return best match.
    """
    scores = []
    
    for provider in providers:
        score = 0.0
        reasons = []
        
        # Capability match (0-40 points)
        cap_match = provider.capability_score(task_estimate)
        score += cap_match * 40
        reasons.append(f"capability:{cap_match:.2f}")
        
        # Availability (0-30 points)
        quota = provider.quota_status()
        if quota.is_exhausted:
            continue  # Skip unavailable providers
        avail_score = quota.health_score()  # 0.0-1.0
        score += avail_score * 30
        reasons.append(f"availability:{avail_score:.2f}")
        
        # Cost efficiency (0-20 points)
        estimated_cost = provider.estimate_cost(task_estimate)
        cost_score = 1.0 - (estimated_cost / MAX_ACCEPTABLE_COST)
        score += max(cost_score, 0) * 20
        reasons.append(f"cost:{cost_score:.2f}")
        
        # Parallel capacity (0-10 points)
        if profile.parallel_preference > 0.7:
            par_score = min(provider.max_parallel / 100, 1.0)
            score += par_score * 10
            reasons.append(f"parallel:{par_score:.2f}")
        
        scores.append((provider, score, reasons))
    
    # Sort by score descending
    scores.sort(key=lambda x: x[1], reverse=True)
    
    # Apply user override if provided
    if user_hint := profile.explicit_provider:
        for provider, score, reasons in scores:
            if provider.name == user_hint:
                return Selection(provider, "user_override", reasons)
    
    # Return best available
    best = scores[0]
    return Selection(best[0], "optimized", best[2])
```

**Key insight:** The selector doesn't just pick the "best" provider—it picks the **best for this user, this task, right now**.

---

### 3. Execution Engine: The Async Core

**Purpose:** Actually run the work, handle concurrency, manage failures.

**Design Philosophy:** 
- **Asyncio-native**: No threads, no processes, pure async/await
- **Streaming-first**: Results flow back as they're available
- **Failure-isolated**: One agent failing doesn't kill the swarm

**Implementation:**

```python
class ExecutionEngine:
    def __init__(self):
        self.semaphore_cache = {}
    
    async def execute_swarm(
        self,
        tasks: List[str],
        provider: Provider,
        max_concurrent: int,
        profile: ExecutionProfile
    ) -> SwarmResult:
        """
        Execute N tasks across M parallel agents.
        """
        # Create semaphore for this operation
        sem = asyncio.Semaphore(max_concurrent)
        
        # Create agent tasks
        agent_coros = [
            self._run_agent(task, provider, sem, i, profile)
            for i, task in enumerate(tasks)
        ]
        
        # Execute with gather (true parallelism)
        results = await asyncio.gather(*agent_coros, return_exceptions=True)
        
        # Process results
        successes = []
        failures = []
        partials = []
        
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                if profile.fallback_required:
                    # Retry this specific task with fallback
                    retry = await self._fallback_execute(tasks[i], profile)
                    if retry:
                        successes.append(retry)
                    else:
                        failures.append((i, tasks[i], str(result)))
                else:
                    failures.append((i, tasks[i], str(result)))
            elif result.is_partial:
                partials.append(result)
            else:
                successes.append(result)
        
        # Aggregate based on strategy
        aggregated = self._aggregate(successes, profile.aggregation_strategy)
        
        return SwarmResult(
            content=aggregated,
            completed=len(successes),
            failed=len(failures),
            partial=len(partials),
            cost=sum(r.cost for r in successes),
            latency_ms=int((time.monotonic() - start) * 1000),
            provider=provider.name,
            fallback_used=len(failures) > 0 and profile.fallback_required
        )
    
    async def _run_agent(
        self,
        task: str,
        provider: Provider,
        sem: asyncio.Semaphore,
        index: int,
        profile: ExecutionProfile
    ) -> AgentResult:
        """
        Single agent execution with resource constraints.
        """
        async with sem:  # Acquire slot
            start = time.monotonic()
            
            try:
                # Execute with timeout
                response = await asyncio.wait_for(
                    provider.execute(task),
                    timeout=profile.timeout_seconds
                )
                
                latency = time.monotonic() - start
                
                return AgentResult(
                    index=index,
                    task=task,
                    content=response.content,
                    tokens=response.usage.total_tokens,
                    cost=provider.calculate_cost(response.usage),
                    latency_ms=int(latency * 1000),
                    status="success"
                )
                
            except asyncio.TimeoutError:
                return AgentResult(
                    index=index,
                    task=task,
                    content="",
                    tokens=0,
                    cost=0,
                    latency_ms=profile.timeout_seconds * 1000,
                    status="timeout",
                    retry_recommended=True
                )
            except Exception as e:
                return AgentResult(
                    index=index,
                    task=task,
                    content="",
                    tokens=0,
                    cost=0,
                    latency_ms=0,
                    status="error",
                    error=str(e),
                    retry_recommended=isinstance(e, TransientError)
                )
```

**Why this design:**
- **Semaphore-per-operation**: Different swarms don't block each other
- **Exception-as-result**: Failures are data, not crashes
- **Streaming-ready**: Provider adapters can yield tokens for real-time feedback

---

### 4. Resilience Layer: The Safety Net

**Purpose:** Ensure the user gets a result, even if the primary plan fails.

**Components:**

#### 4.1 Circuit Breaker

```python
class CircuitBreaker:
    """
    Prevent cascading failures by temporarily disabling unhealthy providers.
    """
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failures = 0
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        self.last_failure = None
        self.threshold = failure_threshold
        self.timeout = recovery_timeout
    
    def call(self, func):
        if self.state == "OPEN":
            if time.monotonic() - self.last_failure > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpen("Provider temporarily disabled")
        
        try:
            result = func()
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.monotonic()
            if self.failures >= self.threshold:
                self.state = "OPEN"
            raise e
```

#### 4.2 Fallback Chain

```python
async def execute_with_fallback(
    task: str,
    primary: Provider,
    fallbacks: List[Provider],
    profile: ExecutionProfile
) -> AgentResult:
    """
    Try primary, then fallbacks until success or exhaustion.
    """
    chain = [primary] + fallbacks
    
    for i, provider in enumerate(chain):
        try:
            # Check circuit breaker
            if provider.circuit_breaker.state == "OPEN":
                continue
            
            # Attempt execution
            result = await provider.execute(task)
            
            # Success—reset circuit breaker if it was failing
            provider.circuit_breaker.record_success()
            
            return AgentResult(
                content=result.content,
                provider=provider.name,
                fallback_used=(i > 0),
                fallback_position=i
            )
            
        except Exception as e:
            provider.circuit_breaker.record_failure()
            
            # Log for telemetry
            logger.warning(f"Provider {provider.name} failed: {e}")
            
            # Continue to next fallback
            continue
    
    # All providers exhausted
    raise SwarmExhausted("All providers in fallback chain failed")
```

#### 4.3 Partial Result Delivery

```python
def aggregate_with_partial_handling(
    results: List[AgentResult],
    strategy: str,
    total_tasks: int
) -> str:
    """
    Combine results, clearly indicating what succeeded and what failed.
    """
    successes = [r for r in results if r.status == "success"]
    failures = [r for r in results if r.status != "success"]
    
    if len(successes) == total_tasks:
        # Complete success—standard aggregation
        return standard_aggregate(successes, strategy)
    
    # Partial success—include failure summary
    aggregated = standard_aggregate(successes, strategy)
    
    failure_summary = f"""
    
    ---
    ⚠️ PARTIAL COMPLETION: {len(successes)}/{total_tasks} tasks succeeded
    
    Failed tasks:
    {chr(10).join(f"  • Task {r.index}: {r.status} - {r.error[:50]}" for r in failures)}
    
    Recommendations:
    {generate_recommendations(failures)}
    """
    
    return aggregated + failure_summary
```

---

## The Provider Adapter: The Only Abstraction

### Interface Design

Every provider—Kimi, DeepSeek, Z.ai, Qwen, Claude—implements the same minimal interface:

```python
class ProviderAdapter(ABC):
    """
    The single abstraction over all AI providers.
    """
    
    # Identification
    name: str
    capabilities: Set[str]
    max_parallel: int
    
    # Economics
    cost_per_1m_input: float
    cost_per_1m_output: float
    
    @abstractmethod
    async def execute(self, task: str) -> ProviderResponse:
        """Execute task, return complete response."""
        pass
    
    @abstractmethod
    async def stream(self, task: str) -> AsyncIterator[str]:
        """Execute task, yield tokens as they arrive."""
        pass
    
    @abstractmethod
    async def quota(self) -> QuotaStatus:
        """Return current quota availability."""
        pass
    
    def calculate_cost(self, usage: TokenUsage) -> float:
        """Calculate cost in USD."""
        input_cost = (usage.prompt_tokens / 1_000_000) * self.cost_per_1m_input
        output_cost = (usage.completion_tokens / 1_000_000) * self.cost_per_1m_output
        return input_cost + output_cost
    
    def estimate_cost(self, task: str) -> float:
        """Estimate cost before execution (heuristic)."""
        # Rough heuristic: 1 token ≈ 4 characters
        estimated_input = len(task) // 4
        estimated_output = estimated_input * 2  # Assume 2x expansion
        return self.calculate_cost(TokenUsage(
            prompt_tokens=estimated_input,
            completion_tokens=estimated_output,
            total_tokens=estimated_input + estimated_output
        ))
```

### Why This Interface Is Sufficient

| Complex System | Our Simplification |
|--------------|------------------|
| Provider-specific authentication | Standardized in `__init__` |
| Rate limit handling | Circuit breaker in wrapper |
| Retry logic | Resilience layer handles it |
| Token counting | Provider returns usage in response |
| Streaming protocols | Iterator abstraction |
| Error codes | Transient vs. Fatal classification |

**The insight:** We don't need to model every provider quirk. We model the **common case** and handle **edge cases** in the adapter implementation.

---

## Data Flow: A Complete Walkthrough

### Scenario: User requests bulk refactoring of 50 files

**Step 1: Invocation**
```
User in Claude Code:
> Refactor all 50 components in src/components/ to use TypeScript

Claude recognizes parallelizable bulk work, calls:
swarm_spawn_agents(
    tasks=["Refactor src/components/Button.tsx", "Refactor src/components/Modal.tsx", ...],
    provider="auto",
    max_concurrent=25
)
```

**Step 2: Intent Classification**
```python
profile = classify_intent("swarm_spawn_agents", args)
# Detects:
# - 50 tasks (parallel_preference: 0.95)
# - "refactor" (quality_sensitivity: 0.7)
# - No security keywords (fallback_required: False)
# - No budget mentions (cost_sensitivity: 0.5)
```

**Step 3: Provider Selection**
```python
# Available providers:
# - Kimi: quota 8000/10000, healthy, max_parallel 100
# - DeepSeek: quota unlimited, healthy, max_parallel 50  
# - Z.ai: quota 200/1000, warning, max_parallel 20

selection = select_provider(profile, providers, task_estimate)
# Scores:
# - Kimi: 95/100 (high parallel, good quota, medium cost)
# - DeepSeek: 82/100 (lower parallel, unlimited quota, cheap)
# - Z.ai: 45/100 (low quota, low parallel)

selected: Kimi
rationale: ["capability:0.95", "availability:0.90", "parallel:1.00"]
```

**Step 4: Cost Estimation (Pre-Execution)**
```python
estimated = selected.estimate_cost(total_input_tokens=50000)
# 50 tasks × 1000 tokens each = 50K input
# Estimated output: 100K tokens (2x expansion)
# Cost: (50K/1M × $0.60) + (100K/1M × $2.50) = $0.03 + $0.25 = $0.28

Claude displays: "Estimated cost: $0.28. Proceed? [Y/n]"
```

**Step 5: Execution**
```python
# Create semaphore for 25 concurrent
sem = asyncio.Semaphore(25)

# Spawn 50 agents
coros = [run_agent(task, sem) for task in 50_tasks]
results = await asyncio.gather(*coros, return_exceptions=True)

# 48 succeed, 2 timeout
successes = 48
failures = 2 (index 12, index 37)
```

**Step 6: Resilience (Automatic Retry)**
```python
# 2 failures detected, fallback not required (not critical)
# But we have DeepSeek configured as fallback

for failed_idx in [12, 37]:
    retry_result = await fallback_execute(tasks[failed_idx], DeepSeek)
    if retry_result.success:
        successes.append(retry_result)
        fallback_used = True
```

**Step 7: Aggregation**
```python
# 50 successful results (48 Kimi + 2 DeepSeek fallback)
aggregated = aggregate(results, strategy="merge_code")
# Combines all file changes into unified diff format
```

**Step 8: Response**
```python
return TextContent(
    text=f"""
    Refactored 50 components to TypeScript.
    
    Summary:
    - 48 files via Kimi (primary)
    - 2 files via DeepSeek (fallback for timeouts)
    
    Changes:
    {aggregated_diff}
    
    Cost: $0.31 ($0.28 Kimi + $0.03 DeepSeek fallback)
    Time: 8.2 seconds
    """
)
```

---

## Configuration: The Minimal Surface

### Philosophy

**Zero required configuration.** The system works with sensible defaults. Configuration is for **optimization**, not **operation**.

### The Three Config Files

#### 1. `~/.config/swarm-mcp/providers.json` (Required Minimum)
```json
{
  "kimi": {
    "api_key": "sk-..."
  }
}
```
**That's it.** Everything else is inferred or defaulted.

#### 2. `~/.config/swarm-mcp/preferences.json` (Optional Optimization)
```json
{
  "default_provider": "kimi",
  "cost_optimization": "aggressive",
  "parallel_cap": 50,
  "fallback_chain": ["kimi", "deepseek", "claude"]
}
```

#### 3. `~/.config/swarm-mcp/secrets.json` (Auto-Generated)
```json
{
  "circuit_breaker_states": {
    "kimi": {"failures": 0, "state": "CLOSED"},
    "deepseek": {"failures": 2, "state": "CLOSED"}
  },
  "quota_cache": {
    "kimi": {"remaining": 8500, "timestamp": "2025-03-04T12:00:00Z"},
    "deepseek": {"remaining": null, "timestamp": "2025-03-04T12:00:00Z"}
  }
}
```
**Managed automatically.** Users never edit this.

---

## Error Handling: The Graceful Degradation Spectrum

| Severity | User Experience | System Action |
|----------|--------------|---------------|
| **Info** | "Using DeepSeek instead of Kimi for better availability" | Automatic routing, no interruption |
| **Warning** | "2 of 50 tasks failed, retried with fallback" | Partial completion, clear summary |
| **Error** | "All providers exhausted. 3 tasks pending. Retry in 15 minutes?" | Queued for retry, user choice |
| **Critical** | "Configuration error: No providers configured" | Hard stop, clear fix instructions |

**Principle:** The user always gets a result or a clear path forward. Never a stack trace.

---

## Performance Budgets

### Target Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Tool invocation → first token | <100ms | Local processing only |
| Provider selection | <50ms | With cached quota |
| Agent spawning (100 agents) | <500ms | asyncio overhead |
| End-to-end 100-task swarm | <10s | Wall clock, including aggregation |
| Memory per agent | <10MB | Including context buffers |
| Cost estimation accuracy | ±20% | Pre-execution estimate vs. actual |

### Optimization Strategies

| Bottleneck | Solution |
|-----------|----------|
| Quota checking latency | Cache with 30s TTL, background refresh |
| Provider connection overhead | HTTP keep-alive, connection pooling |
| Large result aggregation | Streaming aggregation, don't buffer all |
| Cold start | Lazy provider initialization |

---

## Security Model

### Threat: API Key Exposure
**Mitigation:** Keys only in environment/config files, never in logs or error messages.

### Threat: Prompt Injection via Tasks
**Mitigation:** Input validation, no direct system prompt modification by user tasks.

### Threat: Provider Data Retention
**Mitigation:** No training data opt-out where available (DeepSeek, Kimi support this).

### Threat: Man-in-the-Middle
**Mitigation:** TLS 1.3 for all provider connections, certificate pinning optional.

---

## The Evolution Path

### Version 1.0 (Current)
- 4 core tools
- 4-5 providers
- Local stdio transport
- Basic resilience

### Version 1.5 (Next)
- **Streaming results**: Real-time token delivery for long operations
- **Persistent memory**: Cross-session learning of user preferences
- **Provider marketplace**: Community adapters for niche providers

### Version 2.0 (Future)
- **Agent-to-agent communication**: Swarms that coordinate with each other
- **Self-optimizing router**: ML-based routing based on historical success
- **Visual orchestration**: Web UI for designing complex multi-stage swarms

---

## The Anti-Features

What we explicitly do NOT build:

| Feature | Why Not | Alternative |
|---------|---------|-------------|
| Visual workflow editor | Claude is the orchestrator | Natural language |
| Complex DAG definitions | YAGNI—most tasks are flat | Sequential tool calls if needed |
| Persistent agent state | Complexity, debugging nightmare | Stateless with context passing |
| Multi-turn agent conversations | Claude handles dialogue | Single-shot agent execution |
| Custom training/fine-tuning | Out of scope, provider feature | Provider's native capabilities |

---

<div align="center">

## The Architecture in One Sentence

> **Swarm is a transparent bridge that lets Claude Code execute natural language intentions across a federation of AI providers, automatically optimizing for speed, cost, and reliability while requiring zero configuration and handling all failure modes gracefully.**

</div>

---

## Implementation Checklist

### Core (Must Have)
- [ ] MCP server scaffolding
- [ ] Intent classifier (4-profile system)
- [ ] Provider selector (scoring algorithm)
- [ ] Execution engine (asyncio gather)
- [ ] Kimi adapter (OpenAI-compatible)
- [ ] DeepSeek adapter (OpenAI-compatible)
- [ ] Resilience layer (circuit breaker, fallback)
- [ ] Cost estimation (pre-execution)
- [ ] Error aggregation (partial results)

### Polish (Should Have)
- [ ] Streaming support (token-by-token)
- [ ] Z.ai adapter (subscription handling)
- [ ] Quota caching (30s TTL)
- [ ] Connection pooling (HTTP keep-alive)
- [ ] Configuration validation (JSON schema)

### Advanced (Could Have)
- [ ] Qwen adapter (1M context)
- [ ] Claude adapter (quality fallback)
- [ ] Persistent preferences (cross-session)
- [ ] Telemetry (opt-in usage analytics)

---

This architecture specification reflects the **brutal simplicity** that comes from understanding what users actually need. Four subsystems. One abstraction. Zero configuration. Maximum resilience.
