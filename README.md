
# Swarm MCP 🐝

**Multi-Agent Orchestration Bridge for Claude Code**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![MCP Protocol](https://img.shields.io/badge/MCP-Protocol-green.svg)](https://modelcontextprotocol.io/)

> **Stop sequencing. Start swarming.**  
> Route Claude Code's reasoning to 100+ parallel agents across Kimi, DeepSeek, Qwen, and Z.ai—without hitting rate limits or paying Claude Max prices.

---

## The Problem

You're using **Claude Code** or **Z.ai** for serious development work. You've built "clever sequencing systems" to multitask. You know the pain:

- **Claude Code subagents**: Hard limit of 3-5 concurrent tasks, and they can't spawn subagents 
- **Z.ai GLM-5**: 3× quota consumption, sequential tool calling, rate limits on agent spawn 
- **Claude Max ($200/month)**: Still rate-limited when you need 50 agents *right now*
- **The "Good Enough" Trap**: Single-agent tools force you to choose between speed, cost, and quality—never all three

**You shouldn't need a PhD in queue management to refactor a codebase.**

---

## The Solution

Swarm MCP is a **Model Context Protocol (MCP) server** that turns Claude Code into a multi-model orchestrator. Keep Claude's reasoning for architecture. Burst to Kimi's 100-agent swarm for implementation. Fallback to DeepSeek for cheap reasoning. All through Claude Code's native interface.

### Killer Features

| Feature | What It Does | Why You Care |
|---------|--------------|--------------|
| **🚀 100-Agent Parallel Burst** | Spawn up to 100 Kimi K2.5 agents simultaneously via API | Refactor 50 files in parallel, not sequence. No more "clever workarounds." |
| **🧠 Smart Model Routing** | Auto-route tasks: Kimi for speed, DeepSeek for reasoning, Claude for quality | Stop overpaying for simple tasks. Use Opus for architecture, Kimi for boilerplate. |
| **💰 Unified Quota Management** | Track rate limits across Z.ai, Kimi, DeepSeek in one place | Never hit a wall mid-refactor. Automatic fallback when quotas exhaust. |
| **⚡ Zero Context Waste** | Externalize swarm results; Claude only sees aggregated summaries | Claude's 200K context window is for *your* code, not agent chatter. |
| **🔌 Native Claude Code Integration** | Works via MCP—no patches, no forks, no browser extensions | Install in 60 seconds. Use `claude` CLI normally. Tools appear when needed. |

---

## Real-World Impact

**Before Swarm MCP:**
```bash
# You: Refactor this React app to TypeScript
Claude: I'll start with src/components/Button.tsx...
[5 minutes later]
Claude: Now src/components/Modal.tsx...
[5 minutes later]
Claude: Now src/hooks/useAuth.ts...
# 47 files × 5 minutes = 4 hours of your life
```

**After Swarm MCP:**
```bash
# You: Refactor this React app to TypeScript
Claude: I'll parallelize this. Spawning 20 agents...
[30 seconds later]
Claude: All 47 files refactored. Here's the summary of type safety improvements...
```

**Cost comparison** for 100 parallel agent tasks:
- **Claude Max plan**: $200/month, still rate-limited
- **Swarm MCP + Kimi API**: ~$0.60 (yes, 60 cents) for the burst 

---

## Architecture

```
┌─────────────────────────────────────────┐
│         Claude Code (Your Terminal)      │
│         - Natural language commands        │
│         - Architecture decisions           │
│         - Final code review                │
└─────────────────┬───────────────────────┘
                  │ MCP Protocol (stdio/sse)
                  ▼
┌─────────────────────────────────────────┐
│         Swarm MCP Server                 │
│  ┌─────────────┐    ┌─────────────┐     │
│  │ Task Router │───→│ Load Balancer│    │
│  │ (Claude     │    │ (Quota-aware)│    │
│  │  decides)   │    └─────────────┘     │
│  └─────────────┘           ↓             │
│         ↓                  ↓             │
│  ┌─────────────┐    ┌─────────────┐     │
│  │ Kimi Swarm  │    │ DeepSeek    │     │
│  │ (100 agents)│    │ (Reasoning) │     │
│  │ $0.60/M     │    │ $0.28/M     │     │
│  └─────────────┘    └─────────────┘     │
│         ↓                  ↓             │
│  ┌─────────────┐    ┌─────────────┐     │
│  │ Z.ai Backup │    │ Qwen Context│     │
│  │ (GLM-4.7)   │    │ (1M tokens) │     │
│  └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────┘
```

**Key insight**: Claude Code remains the orchestrator—you don't lose its reasoning. But the *execution* happens outside Claude's rate limits, in parallel, at 1/10th the cost.

---

## Installation

### Prerequisites

- **Python 3.10+**
- **Claude Code** (install: `npm install -g @anthropic-ai/claude-code`)
- **API Keys**: At least one of Kimi, DeepSeek, Z.ai, or Qwen

### Step 1: Install Swarm MCP

```bash
# Via pip (recommended)
pip install swarm-mcp

# Or from source
git clone https://github.com/yourusername/swarm-mcp.git
cd swarm-mcp
pip install -e .
```

### Step 2: Configure API Keys

```bash
# Create config directory
mkdir -p ~/.config/swarm-mcp

# Add your keys (at least one required)
cat > ~/.config/swarm-mcp/config.json <<EOF
{
  "providers": {
    "kimi": {
      "api_key": "sk-your-moonshot-key",
      "base_url": "https://api.moonshot.cn/v1",
      "model": "kimi-k2-5",
      "max_parallel": 100,
      "cost_per_1m_tokens": 0.60
    },
    "deepseek": {
      "api_key": "sk-your-deepseek-key",
      "base_url": "https://api.deepseek.com/v1",
      "model": "deepseek-chat",
      "max_parallel": 50,
      "cost_per_1m_tokens": 0.28
    },
    "zai": {
      "api_key": "your-zai-key",
      "base_url": "https://api.z.ai/v1",
      "model": "glm-4.7",
      "max_parallel": 20,
      "cost_per_1m_tokens": 0.60
    },
    "qwen": {
      "api_key": "sk-your-qwen-key",
      "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
      "model": "qwen3-235b-a22b",
      "max_parallel": 30,
      "cost_per_1m_tokens": 0.80
    }
  },
  "routing": {
    "default_strategy": "cost_optimized",
    "fallback_on_quota_exceeded": true
  }
}
EOF
```

### Step 3: Connect to Claude Code

```bash
# Add Swarm MCP to Claude Code
claude mcp add swarm-mcp -- python -m swarm_mcp.server

# Or manually edit ~/.claude.json:
{
  "mcpServers": {
    "swarm-mcp": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "swarm_mcp.server"],
      "env": {
        "SWARM_CONFIG_PATH": "~/.config/swarm-mcp/config.json"
      }
    }
  }
}
```

### Step 4: Verify

```bash
claude
# Inside Claude Code:
/tools  # You should see swarm_spawn_agents, swarm_smart_route, etc.
```

---

## Usage

Swarm MCP exposes **four core tools** that Claude Code discovers automatically:

### 1. `swarm_spawn_agents` — Parallel Execution

**When to use**: You need the same task applied to many files/components.

```json
{
  "tool": "swarm_spawn_agents",
  "arguments": {
    "tasks": [
      "Convert src/components/Button.tsx to TypeScript with strict types",
      "Convert src/components/Modal.tsx to TypeScript with strict types",
      "Convert src/components/Card.tsx to TypeScript with strict types"
    ],
    "provider": "kimi",
    "max_concurrent": 20,
    "aggregation_strategy": "concatenate"
  }
}
```

**Claude Code usage** (natural language):
> "Spawn agents to convert all files in src/components to TypeScript using Kimi"

### 2. `swarm_smart_route` — Intelligent Model Selection

**When to use**: You want optimal cost/quality tradeoffs without thinking about it.

```json
{
  "tool": "swarm_smart_route",
  "arguments": {
    "task": "Implement JWT authentication middleware",
    "complexity": "high",
    "budget_priority": "balanced"
  }
}
```

**Routing logic**:
- **Low complexity** (linting, formatting) → Z.ai GLM-4.7 ($0.60/M, fastest)
- **Medium complexity** (CRUD, APIs) → Kimi K2.5 ($0.60/M, 100 parallel)
- **High complexity** (algorithms, security) → DeepSeek V3.2 ($0.28/M, reasoning mode)
- **Critical quality** (architecture, review) → Claude (your existing setup)

### 3. `swarm_quota_status` — Unified Rate Limit Tracking

**When to use**: Before a big operation, check if you'll hit walls.

```json
{
  "tool": "swarm_quota_status",
  "arguments": {}
}
```

**Returns**:
```json
{
  "kimi": {"remaining": 8500, "reset_in": "2h 15m", "status": "healthy"},
  "deepseek": {"remaining": "unlimited", "status": "healthy"},
  "zai": {"remaining": 1200, "reset_in": "45m", "status": "warning"}
}
```

### 4. `swarm_fallback_chain` — Resilient Execution

**When to use**: Critical tasks that *must* complete even if primary provider fails.

```json
{
  "tool": "swarm_fallback_chain",
  "arguments": {
    "task": "Generate comprehensive API documentation",
    "chain": ["kimi", "deepseek", "qwen", "claude"],
    "timeout_per_provider": 30
  }
}
```

---

## Advanced Configuration

### Custom Routing Strategies

Edit `~/.config/swarm-mcp/routing.yaml`:

```yaml
strategies:
  # Maximize parallelization for speed
  speed_demon:
    default_provider: kimi
    max_agents: 100
    cost_ceiling: 5.00  # USD per operation
    
  # Minimize cost for routine work
  penny_pincher:
    default_provider: deepseek
    max_agents: 50
    cost_ceiling: 0.50
    
  # Best quality regardless of cost
  quality_or_die:
    default_provider: claude  # Fallback to your existing Claude API
    max_agents: 5
    cost_ceiling: 50.00

  # Your custom strategy
  my_workflow:
    task_patterns:
      - pattern: "refactor|convert|migrate"
        provider: kimi
        agents: 50
      - pattern: "debug|fix|optimize"
        provider: deepseek
        agents: 10
      - pattern: "design|architect"
        provider: claude
        agents: 1
```

### Claude Code Integration Patterns

**Pattern A: Burst Then Review**
```bash
# 1. Spawn parallel implementation
> "Use Kimi to generate unit tests for all utils/ files in parallel"

# 2. Claude reviews and integrates
> "Review these test files for edge cases and integrate into the test suite"
```

**Pattern B: Hybrid Architecture**
```bash
# 1. Claude designs
> "Design a caching layer architecture"

# 2. Kimi implements in parallel
> "Spawn agents to implement this caching layer across all service files"

# 3. DeepSeek verifies
> "Use DeepSeek reasoning mode to verify thread safety in these implementations"
```

**Pattern C: Quota-Aware Fallback**
```bash
# Automatic when Z.ai hits limits
> "Refactor all API calls to use async/await"
[Swarm MCP detects Z.ai quota low, auto-switches to Kimi mid-operation]
```

---

## Cost Transparency

Swarm MCP logs every operation with cost breakdown:

```bash
$ claude
> "Generate documentation for 20 modules"

[Swarm MCP] Spawning 20 agents via Kimi K2.5...
[Swarm MCP] Completed in 12.3s
[Swarm MCP] Cost breakdown:
  - Input tokens: 45,230 ($0.027)
  - Output tokens: 128,450 ($0.321)
  - Total: $0.35 USD
  - vs Claude-only equivalent: ~$4.20 (12x savings)
```

---

## Provider Comparison

| Provider | Best For | Parallel Max | Cost/M Tokens | Context | Reasoning |
|----------|----------|--------------|---------------|---------|-----------|
| **Kimi K2.5** | Speed, parallel bulk work | **100** | $0.60/$2.50 | 256K | Fast |
| **DeepSeek V3.2** | Algorithms, math, logic | 50 | **$0.28/$0.42** | 164K | Deep |
| **Z.ai GLM-4.7** | Cheap routine tasks | 20 | $0.60/$2.20 | 128K | Standard |
| **Z.ai GLM-5** | Quality (but expensive) | 10 | $1.00/$3.20 | 128K | Slow |
| **Qwen 3** | Massive context needs | 30 | $0.80/$2.40 | **1M** | Standard |
| **Claude** | Final review, critical code | 1 | $3.00/$15.00 | 200K | **Best** |

*Prices and limits subject to provider changes. Swarm MCP auto-updates from provider APIs.*

---

## Troubleshooting

### "Claude doesn't see the tools"
```bash
# Restart Claude Code
claude config set mcpServers '{}'
claude mcp add swarm-mcp -- python -m swarm_mcp.server
```

### "Kimi API returns 429 (rate limit)"
- Swarm MCP auto-retries with exponential backoff
- Check `swarm_quota_status` to see reset times
- Consider reducing `max_concurrent` in config

### "Results are too large for Claude's context"
- Swarm MCP auto-summarizes results >100K tokens
- Or use `aggregation_strategy: "summary"` in tool calls

### "I want to use this without Claude Code"
You can use Swarm MCP as a standalone API server:
```bash
python -m swarm_mcp.server --transport sse --port 3000
# Now call via HTTP SSE protocol from any client
```

---

## Roadmap

- [ ] **v0.2.0**: Automatic cost optimization based on historical task performance
- [ ] **v0.3.0**: Persistent agent memory across sessions (vector DB integration)
- [ ] **v0.4.0**: Visual dashboard for swarm monitoring (real-time agent status)
- [ ] **v0.5.0**: Cross-provider agent collaboration (Kimi + DeepSeek working together)
- [ ] **v1.0.0**: Self-healing swarms (auto-retry failed agents with different providers)

---

## Contributing

**This is a vacuum in the ecosystem.** The MCP server registry has 1,000+ servers, but zero for multi-model agent swarms . Your use case—hitting Claude Code limits and needing Chinese AI parallelization—is shared by thousands of developers.

### How to contribute:
1. **Use it**: Report bugs, suggest features, share cost savings data
2. **Add providers**: Implement adapters for MiniMax, Baichuan, etc.
3. **Improve routing**: Better heuristics for task→model matching
4. **Documentation**: Share your Claude Code + Swarm MCP workflows

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

---

## License

MIT License - See [LICENSE](LICENSE)

---

## Acknowledgments

- **Moonshot AI** for Kimi K2.5's 100-agent capability and Agent Swarm mode 
- **Anthropic** for Claude Code and the MCP protocol 
- **DeepSeek** for the cheapest frontier-class reasoning 
- **Model Context Protocol** community for the interoperability standard 

---

**Built by developers who were tired of "clever sequencing systems."**

**Stop waiting. Start swarming.**
```

---

## Why This README Works

1. **Immediate hook**: "Stop sequencing. Start swarming."—speaks directly to your pain point
2. **Credibility through specifics**: Real cost numbers ($0.60 vs $200), real provider limits (100 vs 3-5 agents), real architecture diagrams
3. **No overstatement**: Doesn't claim to replace Claude, claims to *augment* it—more believable
4. **Actionable in 60 seconds**: Clear install path, immediate verification step
5. **Shows the math**: Cost comparison table proves value quantitatively
6. **Acknowledges the ecosystem**: Credits MCP, Anthropic, Moonshot—shows this is a bridge, not a replacement
7. **Vacuum statement**: Explicitly notes this is the first multi-model swarm MCP server—true based on my research

