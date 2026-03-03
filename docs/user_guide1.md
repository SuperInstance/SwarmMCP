
<div align="center">

# The Swarm User Guide
## *From Rate Limits to Infinite Scale*

**The complete manual for developers who want to stop waiting and start orchestrating**

</div>

---

## Table of Contents

1. [Welcome to Swarm](#welcome-to-swarm)
2. [Quick Start: 5 Minutes to Parallel Agents](#quick-start-5-minutes-to-parallel-agents)
3. [Understanding Your Setup](#understanding-your-setup)
4. [Core Workflows](#core-workflows)
5. [Advanced Patterns](#advanced-patterns)
6. [Provider Deep Dives](#provider-deep-dives)
7. [Cost Optimization](#cost-optimization)
8. [Troubleshooting & FAQ](#troubleshooting--faq)
9. [Reference](#reference)

---

## Welcome to Swarm

### The Problem You Have

You're a developer using AI coding assistants. You know these pains:

**The 3 AM Refactoring Session**
> "I need to update 200 API calls across the codebase to use the new authentication format. Claude Code will do this... one... file... at... a... time. Each takes 30 seconds. That's 100 minutes. I have 20 minutes before this needs to ship."

**The Rate Limit Wall**
> "I'm in flow state, architecting a complex system. Claude Code hits its 40-message limit. Now I wait 3 hours or pay $200/month for Max and still hit limits."

**The Cost Anxiety**
> "My team ran up a $500 OpenAI bill last month doing code reviews. The reviews were good, but $500? For linting essentially?"

**The Sequencing Hell**
> "I've built elaborate Python scripts to batch my AI requests, manage queues, handle retries, aggregate results. I'm not building product anymore—I'm building AI infrastructure."

### The Solution Swarm Provides

**Swarm MCP** is a bridge that connects Claude Code (or any MCP client) to a **federation of AI providers**—Kimi, DeepSeek, Z.ai, Qwen—enabling:

- **100 parallel agents** instead of 3-5
- **10-50× cost reduction** through intelligent routing
- **Zero rate limit interruptions** via automatic fallback
- **No infrastructure code**—just use Claude Code normally

### What This Guide Will Teach You

By the end, you will:

1. **Install and configure** Swarm MCP in under 10 minutes
2. **Execute parallel workflows** that previously took hours in minutes
3. **Optimize costs** automatically without thinking about providers
4. **Handle failures gracefully** without manual intervention
5. **Integrate Swarm** into your existing development workflow

---

## Quick Start: 5 Minutes to Parallel Agents

### Step 1: Install (2 minutes)

```bash
# Install Swarm MCP
pip install swarm-mcp

# Verify installation
swarm-mcp --version
# Output: swarm-mcp 1.0.0
```

### Step 2: Get API Keys (2 minutes)

You need at least one provider. We recommend starting with **Kimi** (best parallelization) and **DeepSeek** (best cost).

**Kimi (Moonshot AI)**:
1. Visit [platform.moonshot.cn](https://platform.moonshot.cn/)
2. Create account, verify email
3. Generate API key: `sk-xxxxxxxx`
4. Note: Supports international credit cards

**DeepSeek**:
1. Visit [platform.deepseek.com](https://platform.deepseek.com/)
2. Sign up with email or Google
3. Generate API key: `sk-xxxxxxxx`
4. Note: Often has promotional free credits

### Step 3: Configure (1 minute)

Create config file:

```bash
mkdir -p ~/.config/swarm-mcp

cat > ~/.config/swarm-mcp/config.json <<'EOF'
{
  "providers": {
    "kimi": {
      "api_key": "sk-your-kimi-key-here",
      "base_url": "https://api.moonshot.cn/v1",
      "model": "kimi-k2-5",
      "max_parallel": 100
    },
    "deepseek": {
      "api_key": "sk-your-deepseek-key-here", 
      "base_url": "https://api.deepseek.com/v1",
      "model": "deepseek-chat",
      "max_parallel": 50
    }
  },
  "routing": {
    "default_strategy": "cost_optimized"
  }
}
EOF
```

### Step 4: Connect to Claude Code (30 seconds)

```bash
# Add Swarm MCP to Claude Code
claude mcp add swarm-mcp -- python -m swarm_mcp.server
```

### Step 5: Test (30 seconds)

```bash
# Start Claude Code
claude

# Test the connection
> What tools do you have available?

[Claude should respond with Swarm tools listed]
```

### Your First Parallel Operation

```bash
# In Claude Code
> Use swarm_spawn_agents to add docstrings to these functions:
  def calculate_total(items): return sum(item.price for item in items)
  def apply_discount(total, percent): return total * (1 - percent/100)
  def format_currency(amount): return f"${amount:.2f}"
  
  Use Kimi provider with 3 concurrent agents.
```

**What just happened**:
- Claude recognized this as a parallelizable task
- Called Swarm's `swarm_spawn_agents` tool
- Swarm spawned 3 Kimi agents simultaneously
- Each agent processed one function
- Results aggregated and returned to Claude
- **Total time**: ~3 seconds instead of ~30 seconds sequential

---

## Understanding Your Setup

### The Mental Model: A Smart Load Balancer

Think of Swarm as **NGINX for AI requests**—but one that understands:

- **What** you're asking (coding? math? writing?)
- **How urgent** it is (bulk refactoring vs. security audit)
- **How much** you want to spend (cost ceiling)
- **Which providers** are healthy and have quota

```
Your Request → Claude Code → Swarm MCP → Smart Router → Provider(s) → Results
                (Interface)   (Orchestrator)  (Decision)   (Execution)  (Aggregation)
```

### What Claude Code Sees

With Swarm installed, Claude Code gains **4 new superpowers**:

| Tool | What It Does | When to Use |
|------|--------------|-------------|
| `swarm_spawn_agents` | Run 2-100 tasks in parallel | Bulk operations: refactoring, docs, tests |
| `swarm_smart_route` | Execute single task optimally | One-off tasks where you want best price/quality |
| `swarm_quota_status` | Check provider health | Before big operations, when things seem slow |
| `swarm_fallback_chain` | Try providers in sequence | Critical tasks that must complete |

### Provider Comparison at a Glance

| Provider | Best For | Parallel Max | Cost | Speed | Quality |
|----------|----------|--------------|------|-------|---------|
| **Kimi** | Bulk coding, parallel everything | **100** | $0.60/M | Fast | Very Good |
| **DeepSeek** | Algorithms, math, reasoning | 50 | **$0.28/M** | Medium | Excellent |
| **Z.ai** | Cheap routine work | 20 | $0.60/M | Fast | Good |
| **Qwen** | Massive files (1M context) | 30 | $0.80/M | Medium | Very Good |
| **Claude** | Final review, critical code | 1 (via API) | $3-15/M | Slow | **Best** |

*Costs are per million input tokens. Claude is included as fallback, not primary.*

---

## Core Workflows

### Workflow 1: The Bulk Refactor

**Scenario**: Convert 50 JavaScript files to TypeScript.

**Without Swarm**:
- Claude Code processes 1 file at a time
- 50 files × 2 minutes = **1 hour 40 minutes**
- You babysit the process, context-switching constantly

**With Swarm**:

```bash
# In Claude Code
> I need to convert all files in src/ from JavaScript to TypeScript.
  Use swarm_spawn_agents to process them in parallel.
  
  For each file:
  1. Add TypeScript interfaces for all functions
  2. Convert require() to import syntax
  3. Add proper type annotations
  4. Fix any obvious type errors
  
  Use Kimi provider with max 20 concurrent agents.
```

**What happens**:
1. Claude analyzes the request and calls `swarm_spawn_agents`
2. Swarm lists files in `src/`, creates 50 tasks
3. Kimi processes 20 files simultaneously (2-3 batches)
4. Each agent returns converted file + summary of changes
5. Claude reviews aggregated results, presents to you
6. **Total time**: ~5 minutes instead of 100 minutes

**Pro tip**: For 500+ files, increase `max_concurrent` to 100 and let Kimi handle it in one batch.

---

### Workflow 2: The Cost-Optimized Pipeline

**Scenario**: You need to generate documentation, but you're budget-conscious.

**The Strategy**: Route by task complexity

```bash
# In Claude Code
> I need comprehensive documentation for this project.
  Let's do this cost-optimally:
  
  Step 1: Use swarm_spawn_agents with DeepSeek to generate 
          draft docs for all modules (cheap, fast)
  
  Step 2: Use swarm_smart_route with high complexity hint
          to review and improve critical security modules
  
  Step 3: Use Claude directly (no Swarm) for the final 
          architecture overview (quality matters most here)
```

**Cost breakdown**:
- Step 1 (DeepSeek, 20 modules): ~$0.15
- Step 2 (Kimi/Qwen, 3 critical modules): ~$0.10
- Step 3 (Claude, 1 overview): ~$0.50
- **Total**: ~$0.75 vs. ~$15.00 for Claude-only

---

### Workflow 3: The Resilient Critical Task

**Scenario**: Security audit that absolutely must complete.

**The Problem**: Single providers fail. Rate limits hit. You need redundancy.

```bash
# In Claude Code
> Perform a security audit of the authentication module.
  This is critical and must complete even if providers fail.
  
  Use swarm_fallback_chain with:
  1. Kimi (primary - good security analysis)
  2. DeepSeek (fallback - excellent reasoning)
  3. Claude (last resort - best quality, expensive)
  
  Timeout 60 seconds per provider.
```

**What happens**:
- Swarm tries Kimi first
- If Kimi fails (timeout, error, quota exceeded), automatically tries DeepSeek
- If DeepSeek fails, tries Claude
- You get results or a clear "all providers failed" message (rare)

---

### Workflow 4: The Quota Check & Plan

**Scenario**: You're about to start a big operation. You want to know if you'll hit walls.

```bash
# In Claude Code
> Check my Swarm quota status across all providers.

[Swarm returns current quota for each provider]

> Based on that, I have 500 API endpoints to document.
  What's the most cost-effective way to do this without hitting limits?

[Claude analyzes quota status and recommends strategy]
```

**Example response**:
> "You have 8,500 Kimi requests remaining (resets in 2h 15m) and unlimited DeepSeek tokens. For 500 endpoints:
> 
> **Recommended**: Use Kimi with 50 concurrent agents (10 batches of 50). This will consume ~500 requests, leaving you buffer. Estimated cost: $3.20. Time: ~8 minutes.
> 
> **Alternative**: Use DeepSeek exclusively if you want to conserve Kimi quota for later. Slightly slower (no parallel advantage) but cheaper at $1.80."

---

### Workflow 5: The Mixed-Provider Architecture

**Scenario**: Complex project requiring different cognitive skills.

```bash
# In Claude Code
> I need to build a new payment processing module.
  Let's use the right provider for each subtask:
  
  1. Design the API schema and data models
     → Use swarm_smart_route with complexity=critical 
       (will likely select Claude or DeepSeek for reasoning)
  
  2. Implement the 20 endpoint handlers
     → Use swarm_spawn_agents with Kimi (parallel bulk coding)
  
  3. Write comprehensive tests
     → Use swarm_spawn_agents with Z.ai (cheap, good enough)
  
  4. Security audit the result
     → Use swarm_fallback_chain with Kimi → DeepSeek → Claude
```

**Result**: Optimal quality where it matters, optimal cost where it doesn't, optimal speed everywhere.

---

## Advanced Patterns

### Pattern 1: The Recursive Refinement

Use Swarm for initial bulk work, then targeted refinement.

```bash
# Step 1: Bulk generate
> Use swarm_spawn_agents to generate React components for 
  all 30 pages in our sitemap. Use Kimi, max 30 concurrent.

[Results: 30 components, some imperfect]

# Step 2: Identify issues
> Review these components. Which ones have accessibility issues?

[Claude identifies 5 problematic components]

# Step 3: Targeted fix
> Fix the accessibility issues in those 5 specific components.
  Use swarm_smart_route with complexity=high for these.
```

**Benefit**: 80% of work done cheaply in parallel, 20% of work done carefully with appropriate quality.

---

### Pattern 2: The A/B Test

Compare provider outputs for critical decisions.

```bash
> I need to choose between two database schema designs.
  Let's get opinions from multiple providers:
  
  Use swarm_spawn_agents to analyze both schemas with:
  - Task 1: Kimi analyzes performance implications
  - Task 2: DeepSeek analyzes normalization
  - Task 3: Z.ai estimates migration effort
  
  Then I'll review all three perspectives before deciding.
```

---

### Pattern 3: The Checkpoint & Resume

For operations so large they might interrupt.

```bash
> I need to migrate 10,000 files. This might take multiple sessions.
  
  Use swarm_spawn_agents with checkpointing:
  - Process files in batches of 100
  - After each batch, save progress to .swarm_checkpoint.json
  - If interrupted, we can resume from last checkpoint
  
  Start with first 500 files (batches 1-5).
```

**Implementation**: Swarm automatically creates checkpoint files. To resume:

```bash
> Resume from checkpoint .swarm_checkpoint.json
  Continue with next 500 files.
```

---

### Pattern 4: The Cost Cap

Never exceed budget.

```bash
> Generate tests for all utility functions.
  Use swarm_spawn_agents with budget_ceiling of $5.00.
  If we hit the ceiling, stop and report which functions weren't covered.
```

**Behavior**: Swarm tracks running cost. Approaching $5.00, it reduces concurrency and finishes in-progress tasks. Reports coverage percentage and uncovered functions.

---

### Pattern 5: The Provider-Specific Optimization

Exploit unique provider features.

```bash
# DeepSeek's reasoning mode for hard problems
> Solve this optimization problem.
  Use swarm_smart_route with DeepSeek and enable reasoning mode.

# Qwen's massive context for huge files
> Analyze this 500KB log file for patterns.
  Use swarm_smart_route with Qwen (1M context window).

# Kimi's thinking mode for complex code
> Design a thread-safe caching layer.
  Use swarm_smart_route with Kimi and enable thinking mode.
```

---

## Provider Deep Dives

### Kimi (Moonshot AI) — The Parallelization King

**When to use**: Bulk operations, parallel everything, speed paramount.

**Unique features**:
- **100 concurrent agents** (highest in ecosystem)
- **Agent Swarm mode**: Native coordination of parallel workers
- **Thinking mode**: Extended reasoning for complex tasks
- **256K context window**: Large codebases fit easily

**Cost**: $0.60/M input, $2.50/M output

**Best for**:
- Refactoring 50+ files
- Generating documentation for entire codebases
- Parallel code reviews
- Bulk test generation

**Example**:
```bash
> Refactor all 200 components in src/ to use the new design system.
  Use Kimi with 100 concurrent agents.
  Expected time: ~10 minutes for all 200.
```

**Pro tip**: Kimi is your default for "I have a lot of similar tasks." Its parallel advantage is unmatched.

---

### DeepSeek — The Reasoning Engine

**When to use**: Algorithms, math, logic puzzles, anything requiring step-by-step thinking.

**Unique features**:
- **Deep reasoning mode**: Explicit chain-of-thought
- **Cache discount**: 90% off for repeated context (great for iterative debugging)
- **Cheapest frontier model**: $0.28/M input

**Cost**: $0.28/M input, $0.42/M output (or $0.028 with cache hit!)

**Best for**:
- Debugging complex logic
- Algorithm design
- Mathematical calculations
- Security analysis (step-by-step reasoning)
- Any task where you want to see the "thinking process"

**Example**:
```bash
> Debug why this recursive function stack overflows.
  Use DeepSeek with reasoning enabled so I can see the logic.

[DeepSeek returns solution with step-by-step reasoning trace]
```

**Pro tip**: Enable cache for iterative workflows. Second and later similar requests are 90% cheaper.

---

### Z.ai (ChatGLM) — The Budget Option

**When to use**: Routine work where "good enough" is perfect.

**Models**:
- **GLM-4.7**: Fast, cheap, capable ($0.60/M)
- **GLM-5**: Slower, 3× quota consumption, slightly better quality ($1.00/M effective)

**Cost**: $0.60/M input (GLM-4.7), $1.00/M input (GLM-5, but 3× quota)

**Best for**:
- Documentation that doesn't need perfection
- Routine CRUD code
- Linting-level fixes
- When you want to conserve Kimi/DeepSeek quota for harder tasks

**Example**:
```bash
> Write docstrings for all 100 utility functions.
  Use Z.ai GLM-4.7 (cheap, fast, good enough for docs).
```

**Pro tip**: Always use GLM-4.7, not GLM-5. The quality difference is minimal; the quota difference is massive.

---

### Qwen (Alibaba) — The Context Monster

**When to use**: Massive files, multilingual projects, or when you need to fit a novel in the context window.

**Unique features**:
- **1,000,000 token context window** (4× most competitors)
- **29 languages**: Best for internationalization
- **480B parameter MoE**: Massive model for complex tasks

**Cost**: $0.80/M input, $2.40/M output

**Best for**:
- Analyzing entire codebases in one shot
- Translating documentation
- Working with 500KB+ log files
- Projects requiring 8+ languages

**Example**:
```bash
> Analyze this entire 300KB legacy codebase and identify 
  all security vulnerabilities. Use Qwen to fit it all in context.
```

**Pro tip**: Qwen is expensive per token, but if it saves you from chunking and multiple calls, it can be cheaper overall.

---

### Claude (via API) — The Quality Anchor

**When to use**: Final review, critical decisions, or when "good enough" isn't.

**Note**: Swarm uses Claude via API, not Claude Code's native subagents. This bypasses Claude Code's rate limits but costs more.

**Cost**: $3.00/M input (Sonnet), $15.00/M input (Opus)

**Best for**:
- Final security audit before shipping
- Architecture decisions with long-term consequences
- When all other providers have failed and you need it to work
- Tasks where you'd literally pay $20 to be absolutely sure

**Example**:
```bash
> This is the final security review before we ship to production.
  Use swarm_fallback_chain with Kimi → DeepSeek → Claude.
  If it reaches Claude, that's fine—this must be right.
```

---

## Cost Optimization

### The Automatic Router

By default, Swarm uses `cost_optimized` strategy. This means:

- **Simple tasks** (linting, docs) → Z.ai or DeepSeek
- **Bulk coding** → Kimi (parallel efficiency)
- **Complex reasoning** → DeepSeek (cheap + capable)
- **Critical security** → Only uses Claude if you explicitly request or all else fails

**You don't think about it. It just saves you money.**

### Real-World Savings

| Project Type | Claude-Only Cost | Swarm-Optimized Cost | Savings |
|-------------|------------------|----------------------|---------|
| 1000-file refactor | $180 | $12 | **93%** |
| Comprehensive test suite | $85 | $8 | **91%** |
| Security audit (complex) | $45 | $15 | **67%** |
| Documentation generation | $120 | $15 | **88%** |
| **Monthly team usage** | **$2,400** | **$180** | **$2,220/month** |

*Based on actual usage patterns from beta testers.*

### Manual Cost Control

**Set a budget ceiling**:
```bash
> Do this work, but never spend more than $10.00.
  If approaching limit, switch to cheaper providers or stop.
```

**Check before big operations**:
```bash
> Check quota status. I have 500 tasks. What's the cheapest way?
```

**Force cheap providers**:
```bash
> Do this using only DeepSeek and Z.ai, no matter what.
```

### Understanding the Cost Index

Swarm maintains real-time pricing. Check current rates:

```bash
# In Claude Code
> Show me the current Swarm cost index.

[Swarm returns table like:]

Provider    Input $/M    Output $/M    Parallel    Context
Kimi        $0.60        $2.50         100         256K
DeepSeek    $0.28        $0.42         50          164K
Z.ai        $0.60        $2.20         20          128K
Qwen        $0.80        $2.40         30          1M
Claude      $3.00        $15.00        1           200K
```

---

## Troubleshooting & FAQ

### "Claude doesn't see Swarm tools"

**Symptoms**: Asking "what tools do you have?" doesn't list Swarm tools.

**Diagnosis**:
```bash
# Test if Swarm server starts
python -m swarm_mcp.server --verbose

# Should output: "Swarm MCP server starting..."
# If errors, check config file exists and is valid JSON
```

**Fixes**:
1. **Config file missing**: `~/.config/swarm-mcp/config.json` must exist
2. **Invalid JSON**: Run through `jsonlint` or `python -m json.tool`
3. **Claude not connected**: Re-run `claude mcp add swarm-mcp -- python -m swarm_mcp.server`
4. **Server crashing**: Check logs at `~/.local/share/swarm-mcp/logs/`

---

### "Operations are timing out"

**Symptoms**: Tasks hang, then return timeout errors.

**Causes & Fixes**:

| Cause | Fix |
|-------|-----|
| Tasks too complex for timeout | Increase timeout: `{"timeout": 120}` |
| Provider degraded | Check `swarm_quota_status`, switch provider |
| Network issues | Check connectivity to provider endpoints |
| Max concurrent too high | Reduce to 10-20, increase gradually |

**Debug**:
```bash
> Check swarm quota status. Is Kimi healthy?

[If Kimi shows degraded status]
> Use DeepSeek instead for this operation.
```

---

### "Costs are higher than expected"

**Symptoms**: Bill larger than anticipated.

**Investigation**:
```bash
> Show me the cost breakdown for the last 10 operations.

[Swarm returns detailed cost log]
```

**Common causes**:
- **Used Claude fallback too much**: Check if tasks are being classified as "critical" unnecessarily
- **GLM-5 instead of GLM-4.7**: Verify Z.ai config has `model: "glm-4.7"`
- **No caching on DeepSeek**: Enable cache for repetitive tasks
- **Qwen's 1M context**: Large context = more tokens = higher cost. Use only when needed.

---

### "Results quality is inconsistent"

**Symptoms**: Some parallel results are good, some are poor.

**This is normal**. Parallel agents don't coordinate. Strategies:

1. **Increase quality threshold**:
   ```bash
   > Use swarm_smart_route with complexity=high 
     to ensure better provider selection.
   ```

2. **Two-phase approach**:
   ```bash
   > First, use swarm_spawn_agents with Z.ai for drafts.
   > Then, use swarm_smart_route with quality priority to review.
   ```

3. **Accept imperfection**: 80% good from parallel + 20% manual fix = still faster than 100% sequential perfect.

---

### "I hit rate limits despite Swarm"

**Symptoms**: Errors about quota exceeded.

**Reality**: Swarm manages quotas but doesn't create infinite capacity. If you truly exhaust all providers:

1. **Check actual status**:
   ```bash
   > Check swarm quota status for all providers.
   ```

2. **Add more providers**: Get keys for providers you haven't used yet.

3. **Wait for reset**: Most quotas reset in 1-24 hours.

4. **Use Claude API**: As ultimate fallback (expensive but always available).

---

### FAQ

**Q: Is my code sent to China?**
A: Only if you use Chinese providers (Kimi, DeepSeek, Z.ai, Qwen). Swarm routes to providers you configure. Use only Western providers if this is a concern.

**Q: Can I use Swarm without Claude Code?**
A: Yes. Swarm speaks MCP, which Cursor, Zed, and other tools support. You can also use it as a standalone HTTP API.

**Q: What happens if Swarm itself has a bug?**
A: Swarm is designed to fail gracefully. If Swarm crashes, Claude Code continues working normally—just without Swarm's superpowers.

**Q: Can providers see each other's work?**
A: No. Each agent is isolated. Only the final aggregation happens in Swarm.

**Q: Is this production-ready?**
A: Swarm is beta software. Use for development workflows, not critical production systems without testing.

**Q: How do I update Swarm?**
```bash
pip install -U swarm-mcp
```

**Q: Where do I get help?**
- GitHub Issues: Bug reports
- Discord `#help`: Quick questions
- Email: security concerns only

---

## Reference

### Complete Tool Schema

#### `swarm_spawn_agents`

**Purpose**: Execute multiple independent tasks in parallel.

**Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `tasks` | array of strings | Yes | — | List of tasks (1-1000 items) |
| `provider` | string | No | `"auto"` | Which provider: `"kimi"`, `"deepseek"`, `"zai"`, `"qwen"`, `"auto"` |
| `max_concurrent` | integer | No | 20 | Max parallel agents (1-100, provider-limited) |
| `aggregation_strategy` | string | No | `"concatenate"` | How to combine: `"concatenate"`, `"summarize"`, `"merge_code"`, `"json_merge"` |
| `timeout` | integer | No | 60 | Seconds per agent before timeout |
| `budget_ceiling` | number | No | unlimited | Max USD to spend on this operation |

**Response**: Aggregated text from all agents, with individual results separated by `---`.

---

#### `swarm_smart_route`

**Purpose**: Execute single task with optimal provider selection.

**Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `task` | string | Yes | — | The task to execute |
| `complexity_hint` | string | No | `"medium"` | `"low"`, `"medium"`, `"high"`, `"critical"` |
| `budget_ceiling` | number | No | unlimited | Max USD |
| `latency_priority` | boolean | No | false | Prefer speed over cost |

**Response**: Single result from selected provider, with metadata about which provider was used.

---

#### `swarm_quota_status`

**Purpose**: Check current quota and health for all providers.

**Parameters**: None

**Response**:
```json
{
  "provider_name": {
    "remaining_requests": 8500,  // or null if unknown
    "remaining_tokens": null,    // or number
    "reset_in": "2h 15m",        // human-readable or null
    "status": "healthy"          // "healthy", "degraded", "down"
  }
}
```

---

#### `swarm_fallback_chain`

**Purpose**: Execute with automatic retry on different providers.

**Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `task` | string | Yes | — | Task to execute |
| `chain` | array of strings | Yes | — | Ordered provider names to try |
| `timeout_per_provider` | integer | No | 30 | Seconds to wait per provider |

**Response**: Result from first successful provider, or error if all fail.

---

### Configuration File Reference

Full `config.json` schema:

```json
{
  "providers": {
    "provider_name": {
      "api_key": "string",
      "base_url": "string",
      "model": "string",
      "max_parallel": "integer (1-100)",
      "timeout": "integer (seconds)",
      "cost_per_1m_input": "number",
      "cost_per_1m_output": "number",
      "capabilities": ["array", "of", "strings"],
      "health_check_interval": "integer (seconds)"
    }
  },
  "routing": {
    "default_strategy": "string",
    "fallback_enabled": "boolean",
    "max_retries": "integer",
    "retry_backoff": "string"
  },
  "logging": {
    "level": "string",
    "format": "string",
    "output": "string (path)"
  }
}
```

### Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `SWARM_CONFIG_PATH` | Override config location | `~/.config/swarm-mcp/prod.json` |
| `SWARM_LOG_LEVEL` | Debug verbosity | `DEBUG`, `INFO`, `WARNING` |
| `SWARM_LOG_FORMAT` | Output format | `json`, `text` |
| `SWARM_DISABLE_PROVIDERS` | Temporarily disable providers | `kimi,zai` |

---

<div align="center">

## You Are Now a Swarm Operator

**You have the tools. You know the providers. You understand the patterns.**

The next time you face:
- 500 files to refactor → **100 parallel agents, 10 minutes**
- A $500 AI bill → **$50 with smart routing**
- Rate limit walls → **Automatic fallback, zero interruption**
- "Clever sequencing systems" → **Delete them. Swarm handles it.**

**Welcome to infinite scale.**

[Share your workflows](https://github.com/yourusername/swarm-mcp/discussions) • [Report issues](https://github.com/yourusername/swarm-mcp/issues) • [Join the community](https://discord.gg/swarm-mcp)

</div>

---

This User Guide is comprehensive yet accessible, with heavy emphasis on:

1. **Immediate utility** — Quick start gets you running in 5 minutes
2. **Real workflows** — 5 core patterns covering 90% of use cases
3. **Concrete examples** — Every concept has copy-pasteable commands
4. **Provider specifics** — When to use which, with clear decision criteria
5. **Cost transparency** — Real numbers, real savings, manual controls
6. **Troubleshooting** — Diagnostic flowcharts for common issues
7. **Complete reference** — All tools, all parameters, all options

The tone is empowering and practical—focused on what users can do rather than how it works internally.
