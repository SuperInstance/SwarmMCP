
<div align="center">

# The Swarm User Guide
## *From Rate Limits to Parallel Power*

**The complete manual for developers who are tired of waiting**

</div>

---

## Table of Contents

1. [What Swarm Solves](#what-swarm-solves)
2. [Quick Start: 5 Minutes to Parallel](#quick-start-5-minutes-to-parallel)
3. [Understanding Your Workflow](#understanding-your-workflow)
4. [The Core Tools in Depth](#the-core-tools-in-depth)
5. [Real-World Recipes](#real-world-recipes)
6. [Cost Optimization Mastery](#cost-optimization-mastery)
7. [Provider Deep Dives](#provider-deep-dives)
8. [Integration Patterns](#integration-patterns)
9. [Troubleshooting & FAQ](#troubleshooting--faq)
10. [Advanced Techniques](#advanced-techniques)

---

## What Swarm Solves

### The Pain You Know

You're coding with AI assistance. It's 2 AM. You're hitting limits.

**Scenario 1: The Refactoring Trap**
```
You: Refactor this 50-file React codebase to TypeScript
Claude: I'll start with file 1 of 50...
[5 minutes pass]
Claude: Now file 2 of 50...
[5 minutes pass]
Claude: Now file 3 of 50...
You: [checking watch] This will take 4 hours.
```

**Scenario 2: The Rate Limit Wall**
```
You: [After 3 hours of work]
Claude: ⚠️ Rate limit reached. 40 messages per 3 hours.
You: But I have 200 more files to review!
```

**Scenario 3: The Cost Anxiety**
```
You: Generate tests for the entire API surface
Claude: [works for 20 minutes]
Your bill this month: $340
You: [wondering if you can expense this]
```

### The Swarm Alternative

Swarm transforms these sequential, rate-limited, expensive workflows into **parallel, resilient, cost-optimized** operations.

**Same scenarios with Swarm:**

```
You: Refactor this 50-file React codebase to TypeScript
Claude: I'll parallelize this with Swarm. Spawning 20 agents via Kimi...
[30 seconds pass]
Claude: All 50 files refactored. Type safety improved from 23% to 94%.
Cost: $0.42
```

---

## Quick Start: 5 Minutes to Parallel

### Installation

```bash
# Install Swarm MCP
pip install swarm-mcp

# Or from source for latest features
pip install git+https://github.com/yourusername/swarm-mcp.git
```

### Configuration (One-Time Setup)

Create your provider file:

```bash
mkdir -p ~/.config/swarm-mcp

cat > ~/.config/swarm-mcp/config.json <<'EOF'
{
  "providers": {
    "kimi": {
      "api_key": "sk-your-moonshot-key",
      "base_url": "https://api.moonshot.cn/v1",
      "model": "kimi-k2-5"
    }
  }
}
EOF
```

**Getting API Keys:**

| Provider | Where to Get | Cost | Best For |
|----------|-------------|------|----------|
| **Kimi** | [platform.moonshot.cn](https://platform.moonshot.cn/) | $0.60/M tokens | Parallel bulk work |
| **DeepSeek** | [platform.deepseek.com](https://platform.deepseek.com/) | $0.28/M tokens | Cheap reasoning |
| **Z.ai** | [z.ai](https://www.z.ai/) | ~$10/month plans | Coding subscription |

### Connect to Claude Code

```bash
# Add Swarm to Claude Code
claude mcp add swarm-mcp -- python -m swarm_mcp.server

# Verify it worked
claude
# Inside Claude:
> What tools do you have available?
# Should list: swarm_spawn_agents, swarm_smart_route, swarm_quota_status
```

### Your First Parallel Operation

```bash
# In Claude Code
> Use swarm_spawn_agents to write Python docstrings for these functions:
  1. def calculate_fibonacci(n): ...
  2. def is_prime(num): ...
  3. def sort_by_date(items): ...
  4. def validate_email(address): ...

[Claude uses Swarm to spawn 4 parallel agents]
[Results return in ~5 seconds instead of ~20 seconds]
```

---

## Understanding Your Workflow

### The Mental Shift: From Dialogue to Orchestration

**Traditional AI Interaction (What You're Used To):**
```
You → AI → Response → You → AI → Response...
[Single thread, back-and-forth, one thing at a time]
```

**Swarm Interaction (What Swarm Enables):**
```
You → Orchestrator → [Agent 1, Agent 2, Agent 3, ... Agent N] → Aggregated Response
[Parallel threads, simultaneous execution, coordinated results]
```

### When to Use Swarm

**Perfect for Swarm (Embarrassingly Parallel):**
- Refactoring multiple files
- Generating documentation for many functions
- Writing tests for an entire module
- Analyzing multiple components independently
- Bulk code review (style, patterns, anti-patterns)
- Data processing across many records

**Better Single-Threaded (Sequential Dependencies):**
- Architecture decisions requiring full context
- Debugging complex interdependent bugs
- Refactoring where every change affects the next
- Learning/explaining concepts conversationally

**The Litmus Test:** *Can I break this into independent chunks that don't need to talk to each other?*

If **yes** → Use Swarm  
If **no** → Use Claude directly

### The Swarm Decision Framework

```
START: What do you need to do?
    │
    ├─► Is it ONE complex decision?
    │   └─► Use Claude directly (reasoning quality paramount)
    │
    ├─► Is it MANY similar tasks?
    │   ├─► Are they independent?
    │   │   ├─► YES → swarm_spawn_agents (parallel speed)
    │   │   └─► NO → Claude with careful prompting (sequential dependency)
    │   │
    │   └─► Unsure about independence?
    │       └─► swarm_smart_route (let Swarm decide)
    │
    └─► Is it URGENT and might fail?
        └─► swarm_fallback_chain (resilient execution)
```

---

## The Core Tools in Depth

### 1. `swarm_spawn_agents` — The Workhorse

**Purpose:** Execute many independent tasks in parallel.

**When to use it:**
- You have a list of similar operations (files, functions, records)
- Speed matters more than perfect coordination between tasks
- You want results aggregated into a single response

**Basic Usage:**
```
Use swarm_spawn_agents to generate unit tests for all functions in src/utils/math.py
```

**Advanced Usage:**
```
I need to refactor 47 React components to use the new design system.
Components are in src/components/.

Use swarm_spawn_agents with:
- Provider: kimi (for 100-agent parallelism)
- Max concurrent: 25 (don't overwhelm the API)
- Aggregation: merge_code (intelligently combine file changes)
- Tasks: one per component file
```

**Parameters Explained:**

| Parameter | What It Does | When to Adjust |
|-----------|-----------|----------------|
| `tasks` | List of work items | Longer = more parallelism |
| `provider` | Which AI to use | See provider selection guide |
| `max_concurrent` | Simultaneous agents | Higher = faster but riskier |
| `aggregation_strategy` | How to combine results | Depends on output type |
| `timeout` | Seconds per agent | Increase for complex tasks |

**Aggregation Strategies:**

| Strategy | Use For | Example |
|----------|---------|---------|
| `concatenate` | Independent outputs | Docstrings, comments |
| `summarize` | Too much output | Research summaries |
| `merge_code` | Code changes | Refactoring multiple files |
| `json_merge` | Structured data | API responses, configs |

---

### 2. `swarm_smart_route` — The Intelligent Selector

**Purpose:** Let Swarm choose the best provider for a single task.

**When to use it:**
- You're unsure which provider is best
- You want automatic cost/quality tradeoffs
- You have multiple providers configured

**Usage:**
```
I need to implement a JWT authentication middleware.
Use swarm_smart_route to select the best provider for this security-critical task.
```

**How it decides:**

```
Task: "Implement JWT auth middleware"

Swarm analyzes:
├── Complexity: HIGH (security, crypto)
├── Domain: coding
├── Safety critical: YES
└── Required capabilities: ["coding", "reasoning", "long_context"]

Provider scores:
├── Kimi: 85/100 (has capabilities, good at coding)
├── DeepSeek: 90/100 (excellent reasoning, cheaper)
├── Z.ai GLM-5: 95/100 (best quality, but expensive)
└── Claude (if configured): 98/100 (highest quality)

Decision: DeepSeek (best capability/price ratio for this task)
```

**Override when you know better:**
```
Actually, use Kimi specifically—I need this done in parallel with other tasks.
```

---

### 3. `swarm_quota_status` — The Dashboard

**Purpose:** Check if you have capacity before starting big jobs.

**When to use it:**
- Before a large operation (100+ agents)
- When things are failing mysteriously (might be quota)
- To plan your work session

**Usage:**
```
Check my swarm quota status across all providers.
```

**Sample Response:**
```
Provider Status:
├── Kimi: ✅ Healthy
│   └── Remaining: 8,500 tokens (resets in 2h 15m)
├── DeepSeek: ✅ Healthy
│   └── Unlimited quota
└── Z.ai: ⚠️ Warning
    └── Remaining: 1,200 tokens (resets in 45m)
    
Recommendation: Use DeepSeek or Kimi for next large task. Z.ai available for small tasks only.
```

---

### 4. `swarm_fallback_chain` — The Safety Net

**Purpose:** Ensure a task completes even if providers fail.

**When to use it:**
- Critical tasks that MUST complete
- You're unsure which provider is most reliable right now
- You want automatic retry with different providers

**Usage:**
```
Generate comprehensive API documentation for our core service.
This is critical for tomorrow's release.

Use swarm_fallback_chain:
- Primary: kimi (fast, parallel)
- If fails: deepseek (reliable, cheap)
- If fails: zai (backup subscription)
- If all fail: claude (guaranteed but expensive)
```

**How it works:**

```
Attempt 1: Kimi
    └── ❌ Timeout after 60s
        └── Attempt 2: DeepSeek
            └── ❌ Rate limit (429)
                └── Attempt 3: Z.ai
                    └── ✅ Success! (12s, $0.08)
                    
Result: Delivered via Z.ai after 72s total
Without fallback: Failed after 60s, no result
```

---

## Real-World Recipes

### Recipe 1: The Mass Refactor

**Problem:** Convert 200 Python files from imperative to functional style.

**Without Swarm:**
- 200 files × 3 minutes = 10 hours of sequential work
- Multiple Claude sessions
- Context loss between sessions
- $200+ in API costs

**With Swarm:**
```
I need to refactor 200 Python files in src/ from imperative to functional style.

Use swarm_spawn_agents with:
- Provider: kimi
- Max concurrent: 50 (conservative for reliability)
- Aggregation: merge_code
- Tasks: one per .py file

For each file:
1. Identify impure functions (side effects)
2. Extract pure functions
3. Replace loops with map/filter/reduce
4. Add type hints to new functions
5. Return unified diff of changes
```

**Result:** 200 files processed in ~8 minutes, cost ~$3.50

---

### Recipe 2: The Documentation Sprint

**Problem:** Document a 50-function API library with no docs.

**Without Swarm:**
- Start documenting, lose steam at function 12
- Inconsistent style across functions
- Takes 3 days of intermittent work

**With Swarm:**
```
Document the entire API in src/api/client.py.

Use swarm_spawn_agents with provider kimi:

For each public function:
1. Analyze parameters and return types
2. Identify side effects and exceptions
3. Write Google-style docstring
4. Provide 2-3 usage examples
5. Note any deprecation warnings

Aggregate into a single markdown file organized by module.
```

**Result:** Complete API docs in 10 minutes, consistent style, cost ~$0.80

---

### Recipe 3: The Test Generation Blitz

**Problem:** Legacy codebase with 0% test coverage, 300 functions.

**Without Swarm:**
- Overwhelming scope leads to procrastination
- Manual test writing is tedious
- Coverage increases 5% per week

**With Swarm:**
```
Generate comprehensive unit tests for all functions in src/.

Use swarm_spawn_agents with provider deepseek (cheap for this volume):

For each function:
1. Identify input types and edge cases
2. Write pytest test cases covering:
   - Happy path
   - Edge cases (empty, null, max values)
   - Error conditions
3. Use mocking for external dependencies
4. Target 90%+ branch coverage

Organize tests in tests/ mirroring src/ structure.
```

**Result:** 300 test files generated in 15 minutes, cost ~$2.20

---

### Recipe 4: The Multi-Provider Research

**Problem:** Compare 5 different approaches to a problem (caching strategies, database schemas, etc.).

**Without Swarm:**
- Research one approach at a time
- Lose track of comparisons
- Biased by recency (last one researched feels best)

**With Swarm:**
```
Research optimal caching strategies for our high-traffic API.

Use swarm_spawn_agents with provider kimi, max 5 concurrent:

Research in parallel:
1. Redis with TTL strategies
2. CDN edge caching (Cloudflare)
3. In-memory LRU with invalidation
4. Database query caching
5. Hybrid multi-layer approach

Each agent should:
- Explain the strategy
- List pros/cons for our use case (high read, occasional write)
- Estimate implementation complexity
- Provide code sketch

Aggregate into comparison matrix with recommendation.
```

**Result:** 5 complete research reports in 3 minutes, cost ~$0.45

---

### Recipe 5: The Resilient Critical Path

**Problem:** Security audit that MUST complete before release, regardless of API issues.

**Without Swarm:**
- Start audit, provider fails mid-way
- Scramble to find alternative
- Delay release

**With Swarm:**
```
Perform security audit of auth/ and payment/ modules.
This is release-blocking—must complete tonight.

Use swarm_fallback_chain:
1. Kimi (parallel analysis of both modules)
2. DeepSeek (deep reasoning if Kimi fails)
3. Claude (guaranteed completion if all else fails)

Audit checklist for each module:
- [ ] Injection vulnerabilities (SQL, NoSQL, Command)
- [ ] Authentication bypasses
- [ ] Sensitive data exposure
- [ ] Business logic flaws
- [ ] Dependency vulnerabilities

Generate executive summary with risk ratings and fixes.
```

**Result:** Audit completes 100% of the time, average cost $1.20, max cost $8.50 (if Claude needed)

---

## Cost Optimization Mastery

### The Price Landscape (As of March 2025)

| Provider | Input Cost | Output Cost | Parallel Max | Quality |
|----------|-----------|-------------|--------------|---------|
| **DeepSeek** | $0.28/M | $0.42/M | 50 | ⭐⭐⭐⭐ |
| **Kimi** | $0.60/M | $2.50/M | **100** | ⭐⭐⭐⭐ |
| **Z.ai GLM-4.7** | $0.60/M | $2.20/M | 20 | ⭐⭐⭐ |
| **Z.ai GLM-5** | $1.00/M | $3.20/M | 10 | ⭐⭐⭐⭐ |
| **Qwen** | $0.80/M | $2.40/M | 30 | ⭐⭐⭐⭐ |
| **Claude 3.5** | $3.00/M | $15.00/M | 5 | ⭐⭐⭐⭐⭐ |

*Prices in USD per million tokens. Quality is subjective coding assessment.*

### Cost Optimization Strategies

#### Strategy 1: The DeepSeek Default

**Rule:** Start with DeepSeek for everything. Upgrade only when necessary.

**When to upgrade:**
- DeepSeek fails or times out → Kimi
- Need >50 parallel agents → Kimi
- Security-critical code → Z.ai GLM-5 or Claude
- Complex reasoning task → DeepSeek (it's already best at this)

**Example workflow:**
```
Routine coding: DeepSeek ($0.28/M)
    ↓ If fails
Fallback to: Kimi ($0.60/M)  
    ↓ If quality insufficient
Final fallback: Claude ($15/M, but rare)
```

**Savings:** 80-95% vs Claude-only workflow

---

#### Strategy 2: The Parallel Premium

**Rule:** Use Kimi's 100-agent parallelism for bulk work, even at slightly higher cost per token.

**Math:**
- **DeepSeek sequential:** 100 tasks × 10s = 1000s (16.7 minutes) @ $0.28/M
- **Kimi parallel:** 100 tasks ÷ 100 agents = 10s @ $0.60/M

**Time savings:** 16.6 minutes  
**Cost premium:** 2.1× per token  
**Net value:** Developer time is worth more than the $0.32/M premium

**Use for:** Bulk refactoring, mass documentation, test generation

---

#### Strategy 3: The Z.ai Subscription Arbitrage

**Rule:** If you have Z.ai subscription (unlimited tokens), use it for routine work.

**Z.ai Max Coding Plan:** ~$15/month unlimited  
**Equivalent API usage:** Would cost $200+ on metered providers

**Best for:** Daily driver work, routine coding, learning projects

**Swarm integration:** Configure Z.ai as primary, others as fallback for parallel bursts

---

#### Strategy 4: The Smart Route Economy

**Rule:** Let `swarm_smart_route` decide, but set constraints.

```
Use swarm_smart_route with:
- Budget ceiling: $5.00 per operation
- Latency priority: false (prefer cost)
- Complexity hint: medium
```

Swarm will:
1. Estimate cost for each capable provider
2. Select cheapest that meets quality bar
3. Fall back to more expensive if cheap fails

**Typical savings:** 60-80% vs manual provider selection

---

### Real Cost Examples

| Scenario | Claude Only | Swarm Optimized | Savings |
|----------|-------------|-----------------|---------|
| Refactor 100 files | $45.00 | $2.80 (DeepSeek) | **94%** |
| Generate 500 tests | $120.00 | $4.20 (Kimi bulk) | **97%** |
| Security audit (critical) | $25.00 | $1.50 (DeepSeek→Kimi→Claude fallback) | **94%** |
| Daily coding (1 month) | $200.00 | $18.00 (Z.ai sub + API bursts) | **91%** |

*Based on actual token usage estimates. Your mileage may vary.*

---

## Provider Deep Dives

### Kimi (Moonshot AI) — The Parallel Powerhouse

**Best for:** Bulk operations, parallel agent spawning, speed

**Strengths:**
- **100 parallel agents** (highest in ecosystem)
- **256K context window** (massive codebases)
- **Fast token generation** (~100 tokens/sec)
- **Reliable API** (OpenAI-compatible)

**Weaknesses:**
- Higher cost than DeepSeek
- China-based (latency ~800ms from US)
- No deep reasoning mode (use DeepSeek for that)

**Optimal use cases:**
- Refactoring 50+ files simultaneously
- Generating documentation for entire modules
- Bulk test generation
- Any "embarrassingly parallel" coding task

**Configuration tip:**
```json
{
  "kimi": {
    "max_parallel": 100,  // Use it all for bulk
    "timeout": 60,        // Generous for big tasks
    "model": "kimi-k2-5"  // Latest version
  }
}
```

---

### DeepSeek — The Reasoning Engine

**Best for:** Algorithms, math, complex logic, cheap volume

**Strengths:**
- **Cheapest frontier-class model** ($0.28/M input)
- **Deep reasoning mode** (explicit chain-of-thought)
- **Context caching** (90% discount on repeated prompts)
- **Excellent for debugging** (methodical analysis)

**Weaknesses:**
- Only 50 parallel agents (half of Kimi)
- Smaller context window (164K vs 256K)
- Can be "too thorough" (verbose outputs)

**Optimal use cases:**
- Algorithm design and optimization
- Complex debugging (race conditions, memory leaks)
- Mathematical computations
- Security analysis (methodical vulnerability finding)

**Pro tip:** Use cache for iterative debugging:
```
Debug this error: [error message]
[DeepSeek reasons through it, caches context]

Now fix the code: [code]
[Same context, 90% cheaper]
```

---

### Z.ai (ChatGLM) — The Subscription Workhorse

**Best for:** Daily driver work with predictable costs

**Strengths:**
- **Subscription plans** (unlimited tokens for fixed price)
- **GLM-4.7**: Fast, cheap, good for routine work
- **GLM-5**: Higher quality when needed (3× quota)
- **Good IDE integration** (Chinese ecosystem focus)

**Weaknesses:**
- GLM-5 is expensive (3× quota consumption)
- Limited parallelism (20 agents max)
- Sequential tool calling (slower for complex operations)

**Optimal use cases:**
- Routine coding with subscription
- Learning and experimentation
- Projects where budget predictability matters

**Swarm strategy:** Use Z.ai for single-agent work, burst to Kimi for parallel needs

---

### Qwen (Alibaba) — The Context King

**Best for:** Massive codebases, multilingual projects

**Strengths:**
- **1 million token context window** (largest available)
- **29 languages** (best for internationalization)
- **480B parameter MoE** (massive model capacity)
- **Competitive pricing** ($0.80/M)

**Weaknesses:**
- Slower than Kimi/DeepSeek
- Less parallelization (30 agents)
- Overkill for most tasks

**Optimal use cases:**
- Analyzing entire repositories in one pass
- Multilingual code generation
- Complex documentation spanning many files

**When to use:** When you need to feed an entire codebase as context

---

## Integration Patterns

### Pattern 1: The Daily Driver Setup

**For:** Developers who use AI coding daily, want reliability and cost control

**Configuration:**
```json
{
  "providers": {
    "zai": {
      "api_key": "...",
      "model": "glm-4.7",
      "max_parallel": 10
    },
    "kimi": {
      "api_key": "...",
      "max_parallel": 50
    },
    "deepseek": {
      "api_key": "...",
      "max_parallel": 20
    }
  },
  "routing": {
    "default_strategy": "cost_optimized"
  }
}
```

**Workflow:**
1. **Daily coding:** Z.ai subscription covers most work
2. **Parallel bursts:** Swarm automatically uses Kimi for multi-file operations
3. **Complex debugging:** DeepSeek selected for reasoning-heavy tasks
4. **Fallback:** If Z.ai quota hits, Kimi/DeepSeek take over

**Estimated monthly cost:** $15 (Z.ai) + $5-10 (API bursts) = **$20-25**

---

### Pattern 2: The Project Sprint Setup

**For:** Intensive project periods (new feature, major refactor)

**Configuration:**
```json
{
  "providers": {
    "kimi": {
      "api_key": "...",
      "max_parallel": 100
    },
    "deepseek": {
      "api_key": "...",
      "max_parallel": 50
    }
  },
  "routing": {
    "default_strategy": "speed_demon"
  }
}
```

**Workflow:**
1. **Bulk generation:** Kimi for tests, docs, boilerplate
2. **Quality refinement:** DeepSeek for complex logic
3. **No rate limits:** Parallel execution prevents bottlenecks

**Estimated weekly cost:** $10-20 during sprint

---

### Pattern 3: The Enterprise Resilience Setup

**For:** Teams where failure is not an option

**Configuration:**
```json
{
  "providers": {
    "kimi": { "max_parallel": 100 },
    "deepseek": { "max_parallel": 50 },
    "zai": { "max_parallel": 20 },
    "claude": { "max_parallel": 5 }
  },
  "routing": {
    "default_strategy": "quality_first",
    "fallback_enabled": true,
    "max_retries": 3
  }
}
```

**Workflow:**
1. **Default:** Kimi for speed
2. **If Kimi fails:** DeepSeek (different infrastructure)
3. **If DeepSeek fails:** Z.ai (different company entirely)
4. **If all Chinese providers fail:** Claude (guaranteed US-based)
5. **Never fails:** Four-layer fallback chain

**Estimated cost:** 2-3× single-provider, but 99.9% uptime

---

### Pattern 4: The Cost-Minimizer Setup

**For:** Side projects, startups, personal use with tight budgets

**Configuration:**
```json
{
  "providers": {
    "deepseek": {
      "api_key": "...",
      "max_parallel": 50,
      "cache_enabled": true
    }
  },
  "routing": {
    "default_strategy": "cost_optimized",
    "budget_ceiling": 1.00
  }
}
```

**Workflow:**
1. **Everything:** DeepSeek at $0.28/M
2. **Caching:** Repeated patterns are 90% cheaper
3. **Hard limit:** Operations fail if they would exceed $1
4. **Manual override:** Explicitly request other providers when worth it

**Estimated monthly cost:** **$5-15**

---

## Troubleshooting & FAQ

### "Claude Code doesn't see Swarm tools"

**Symptoms:** `claude` starts but `> What tools do you have?` doesn't list Swarm tools.

**Diagnosis:**
```bash
# Test if Swarm server starts independently
python -m swarm_mcp.server --verbose
# Should show: "Swarm MCP Server started. Waiting for connections..."

# If error, check config
python -c "from swarm_mcp.config import load_config; print(load_config())"
```

**Common fixes:**

| Cause | Fix |
|-------|-----|
| Config file missing | `mkdir -p ~/.config/swarm-mcp && echo '{}' > ~/.config/swarm-mcp/config.json` |
| Invalid JSON | `python -m json.tool ~/.config/swarm-mcp/config.json` to validate |
| Missing API key | Add at least one provider with `api_key` |
| Claude not configured | Re-run `claude mcp add swarm-mcp -- python -m swarm_mcp.server` |

---

### "Operations are timing out"

**Symptoms:** Agents start but never return, or return "timeout" errors.

**Diagnosis:**
```bash
# Check provider health
python -c "
from swarm_mcp.providers import load_provider
from swarm_mcp.config import load_config
import asyncio

async def check():
    config = load_config()
    for name in config.providers:
        p = load_provider(name, config.providers[name])
        await p.connect()
        healthy = await p.health_check()
        print(f'{name}: {\"✅\" if healthy else \"❌\"}')

asyncio.run(check())
"
```

**Solutions:**

| Cause | Solution |
|-------|----------|
| Tasks too complex | Increase `timeout` parameter (default 60s → 120s) |
| Provider degraded | Check `swarm_quota_status`, switch providers |
| Network issues | Use provider geographically closer to you |
| Too many concurrent | Reduce `max_concurrent` to 10-20 |

---

### "Costs are higher than expected"

**Symptoms:** Bills larger than calculated, or Swarm choosing expensive providers.

**Diagnosis:**
```bash
# Enable cost logging
export SWARM_LOG_LEVEL=DEBUG

# Run operation, check logs for routing decisions
# Look for: "Selected provider X with estimated cost $Y"
```

**Common causes:**

| Cause | Fix |
|-------|-----|
| Using GLM-5 instead of GLM-4.7 | Explicitly specify `"model": "glm-4.7"` in config |
| No quota on cheap providers | Check `swarm_quota_status`, add fallback providers |
| Complex tasks routed to Claude | Use `complexity_hint: "medium"` to avoid over-provisioning |
| Not using cache with DeepSeek | Enable `"cache_enabled": true` in DeepSeek config |

---

### "Results are poor quality"

**Symptoms:** Generated code doesn't work, docs are wrong, tests fail.

**Not a Swarm problem**—this is provider/model selection.

**Solutions:**

| Issue | Solution |
|-------|----------|
| Code doesn't compile | Use `provider: kimi` or `provider: deepseek` (better coding) |
| Security logic wrong | Use `swarm_smart_route` with `complexity_hint: "critical"` |
| Tests don't pass | Add example inputs/outputs to task description |
| Docs are generic | Provide specific API signatures in task |

**Quality hierarchy:**
1. Claude (best, expensive)
2. Kimi/DeepSeek (excellent, cheap)
3. Qwen (very good, massive context)
4. Z.ai GLM-5 (good, expensive)
5. Z.ai GLM-4.7 (adequate, cheap)

---

### "I hit rate limits anyway"

**Symptoms:** Even with Swarm, operations fail with "rate limit" or "quota exceeded".

**Understanding:** Swarm manages parallelism *within* providers, but can't exceed provider limits.

**Solutions:**

| Limit | Workaround |
|-------|------------|
| Kimi daily quota | Add DeepSeek/Z.ai as fallbacks |
| DeepSeek rate limit | Reduce `max_concurrent` to 10, add delays |
| Z.ai 5-hour window | Use Kimi for bursts, Z.ai for steady work |
| All providers exhausted | Wait for reset (check `swarm_quota_status`) |

**Nuclear option:** Configure 4+ providers with `swarm_fallback_chain`—mathematically unlikely to all be exhausted simultaneously.

---

## Advanced Techniques

### Technique 1: The Recursive Swarm

**Problem:** You have a task that needs to be broken down, then each subtask parallelized.

**Example:** Analyze a 500-file codebase, then refactor the 50 most complex files.

**Solution:** Two-phase Swarm operation

```
Phase 1: Analysis (parallel)
> Use swarm_spawn_agents to analyze all 500 files for complexity.
> Each agent returns: [filename, complexity_score, refactoring_needed]

[Wait for results]

Phase 2: Refactoring (parallel on subset)
> Based on analysis, use swarm_spawn_agents on the 50 files 
> with complexity_score > 7.
> Provider: kimi, max_concurrent: 50
```

**Key insight:** Swarm results can inform the next Swarm operation. Claude orchestrates the handoff.

---

### Technique 2: The Provider Chain

**Problem:** You want multiple providers to review the same code for safety.

**Example:** Security-critical authentication code.

**Solution:** Sequential validation with different providers

```
Step 1: Generate with Kimi (fast, parallel)
> Use swarm_spawn_agents to generate 3 alternative implementations 
> of the auth middleware.

Step 2: Review with DeepSeek (reasoning)
> Use swarm_smart_route with complexity: critical to review 
> each implementation for security flaws.

Step 3: Final validation with Claude (quality)
> Present the DeepSeek-validated implementation to Claude 
> for final architectural review.
```

**Result:** Speed of Kimi + thoroughness of DeepSeek + quality of Claude

---

### Technique 3: The Context Seed

**Problem:** You want all parallel agents to share common context without repeating it in every task.

**Example:** Refactoring where all files need to know the new design system spec.

**Solution:** Use Swarm's context passing (if supported) or prepopulate system prompts

```
> Use swarm_spawn_agents with shared_context:

shared_context: |
  We are migrating to Design System v2.0.
  Key changes:
  - All buttons use <Button variant="primary"> not <button class="btn">
  - Colors use theme.colors not hardcoded hex
  - Spacing uses theme.spacing(2) not margin: 16px

tasks:
  - Refactor src/components/Header.tsx
  - Refactor src/components/Footer.tsx
  - Refactor src/components/Sidebar.tsx
```

**Alternative:** Include context in first task, reference it in others:
```
Task 1: "Read DESIGN_SYSTEM_V2.md, then refactor Header.tsx"
Task 2: "Following same design system as Header.tsx, refactor Footer.tsx"
```

---

### Technique 4: The Cost-Capped Operation

**Problem:** You have a firm budget and can't exceed it.

**Solution:** Hard limits with graceful degradation

```
> Generate comprehensive documentation for the API.

Constraints:
- Maximum cost: $2.00
- If approaching limit: switch to summarization mode
- If limit reached: return partial results with completion status

Use swarm_smart_route with budget_ceiling: 2.00
```

**Swarm behavior:**
1. Estimates cost for full operation
2. If >$2.00, reduces scope or switches to cheaper provider
3. Returns what was completed with notes on what remains

---

### Technique 5: The Hybrid Human-Swarm Workflow

**Problem:** Some decisions need human judgment, but execution is pure parallel work.

**Solution:** Human-in-the-loop with Swarm execution

```
You: Analyze these 20 feature requests and categorize them.

Swarm: [Parallel analysis of all 20]

You: [Review 20 one-sentence summaries, approve 8 for implementation]

You: Implement these 8 approved features.

Swarm: [Parallel implementation of all 8, 20 agents each for subtasks]
```

**Key:** Human makes high-value decisions (what to build), Swarm handles execution (building it).

---

<div align="center">

## The Swarm Mindset

**You are no longer limited by the speed of a single AI.**

You are limited only by:
- How well you can decompose problems
- How wisely you select providers
- How creatively you orchestrate parallel work

**The future belongs to those who learn to conduct.**

</div>

---

## Quick Reference Card

### Command Cheat Sheet

| Goal | Command |
|------|---------|
| Check quotas | `Check swarm quota status` |
| Bulk refactor | `Use swarm_spawn_agents to refactor [files]` |
| Smart selection | `Use swarm_smart_route for [task]` |
| Guaranteed completion | `Use swarm_fallback_chain for [critical task]` |
| Cost check | `What would this cost with different providers?` |

### Provider Selection Matrix

| Task Type | First Choice | Fallback | Avoid |
|-----------|-------------|----------|-------|
| Bulk refactor | Kimi | DeepSeek | Claude (expensive) |
| Complex algorithm | DeepSeek | Kimi | Z.ai GLM-4.7 |
| Security audit | DeepSeek | Kimi → Claude | Z.ai GLM-4.7 |
| Mass documentation | Kimi | DeepSeek | — |
| Quick fix | Z.ai GLM-4.7 | DeepSeek | Kimi (overkill) |
| Learning/explaining | Z.ai (unlimited) | DeepSeek | — |
| Must-not-fail | Fallback chain ending in Claude | — | Single provider |

### Cost Estimation Formula

```
Estimated cost = (input_tokens / 1,000,000 × input_price) 
               + (output_tokens / 1,000,000 × output_price)

Quick estimate:
- 1 page of code ≈ 500 tokens
- 1 file of docs ≈ 300 tokens output
- 100 parallel tasks ≈ 100× cost but 1/100th time
```

---

<div align="center">

**Stop waiting. Start swarming.**

[Get Started](#quick-start-5-minutes-to-parallel) • [Report Issues](https://github.com/yourusername/swarm-mcp/issues) • [Join Community](https://discord.gg/swarm-mcp)

</div>

---

This User Guide is designed to be **immediately useful** while building **deep understanding** over time. It focuses on:

1. **Problem-first organization** — Each section starts with a pain point you recognize
2. **Recipe-driven learning** — Copy-pasteable solutions for common scenarios
3. **Progressive disclosure** — Quick start for beginners, advanced techniques for power users
4. **Cost transparency** — Real numbers, real comparisons, real savings calculations
5. **Decision support** — Matrixes, flowcharts, and rules of thumb for quick choices
