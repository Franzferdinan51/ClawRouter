# ClawRouter Development Plan for DuckBot

## 🎯 Project Overview

**Repository:** https://github.com/Franzferdinan51/ClawRouter.git (forked from BlockRunAI)
**Purpose:** Intelligent LLM model router that saves 96% on LLM costs
**Current Features:**
- 14-dimension weighted scoring (70-80% accurate in <1ms)
- 4-tier model selection (SIMPLE, MEDIUM, COMPLEX, REASONING)
- 30+ models across 6 providers (OpenAI, Anthropic, Google, DeepSeek, xAI, Moonshot)
- x402 payments (USDC on Base, non-custodial)
- Response deduplication (SHA-256, 30s cache)
- Payment pre-auth (skips x402 round trip)
- SSE heartbeat (prevents upstream timeouts)
- Open source (MIT license)

## 📋 Potential Improvements for DuckBot Use Case

### 1️⃣ DuckBot-Specific Model Integration

**Priority:** HIGH
**Description:** Add LM Studio local models to the model list

**Why:** DuckBot uses local LM Studio models (qwen3-coder-next, gpt-oss-20b, etc.) for zero-cost inference

**Implementation:**
```typescript
// Add to src/models.ts
{
  id: "local/lmstudio-qwen3-coder-next",
  name: "Qwen 3 Coder Next",
  provider: "lmstudio",
  inputPrice: 0,
  outputPrice: 0,
  contextWindow: 128000,
  maxOutput: 32768,
  reasoning: true,
  local: true,
  baseUrl: "http://100.74.88.40:1234/v1",  // Configurable
},
{
  id: "local/lmstudio-gpt-oss-20b",
  name: "GPT-OSS 20B",
  provider: "lmstudio",
  inputPrice: 0,
  outputPrice: 0,
  contextWindow: 64000,
  maxOutput: 8192,
  reasoning: false,
  local: true,
  baseUrl: "http://100.74.88.40:1234/v1",
},
```

**Benefit:** Zero-cost routing for local models!

---

### 2️⃣ Crypto-Mining Aware Routing

**Priority:** HIGH
**Description:** Detect when LM Studio is under heavy load (crypto mining) and route to alternatives

**Why:** Duckets reported that crypto mining causes LM Studio models to fail or respond slowly

**Implementation:**
```typescript
// Add health check to src/router/health.ts
interface LMStudioHealth {
  isHealthy: boolean;
  responseTime: number;
  lastChecked: Date;
}

async function checkLMStudioHealth(baseUrl: string): Promise<LMStudioHealth> {
  const start = Date.now();
  try {
    const res = await fetch(`${baseUrl}/models`, {
      method: 'GET',
      signal: AbortSignal.timeout(5000),  // 5 second timeout
    });
    const responseTime = Date.now() - start;
    return {
      isHealthy: res.ok,
      responseTime,
      lastChecked: new Date(),
    };
  } catch (error) {
    return {
      isHealthy: false,
      responseTime: 9999,
      lastChecked: new Date(),
    };
  }
}

// Add to routing logic
async function routeWithHealthCheck(
  prompt: string,
  config: RoutingConfig,
  health: LMStudioHealth,
): Promise<Decision> {
  const tier = classifyByRules(prompt, /* ... */).tier;
  
  // If local LM Studio is unhealthy, skip to next tier
  if (tier === 'SIMPLE' || tier === 'MEDIUM') {
    if (!health.isHealthy || health.responseTime > 10000) {
      console.log('LM Studio degraded, skipping to cloud tier');
      // Route to next available tier
      return routeToTier('MEDIUM', config);
    }
  }
  
  // Normal routing
  return routeToTier(tier, config);
}
```

**Benefit:** Graceful fallback during crypto mining!

---

### 3️⃣ DuckBot Persona Integration

**Priority:** MEDIUM
**Description:** Add DuckBot personality to routing decisions and responses

**Why:** Consistency with DuckBot's helpful, direct, duck-themed personality

**Implementation:**
```typescript
// Add DuckBot-specific scoring to src/router/duckbot-rules.ts
function scoreDuckPersonality(prompt: string): DimensionScore {
  const duckKeywords = ['duck', 'quack', 'quack', 'waddle', 'webbed foot', 'feathers'];
  const helpfulKeywords = ['help', 'assist', 'guide', 'explain', 'how', 'what'];
  const directKeywords = ['just', 'directly', 'simply', 'precisely'];
  
  const text = prompt.toLowerCase();
  
  // Bonus for DuckBot personality alignment
  if (duckKeywords.some(kw => text.includes(kw))) {
    return { name: 'duckbotPersona', score: 0.5, signal: 'duck-themed' };
  }
  
  if (helpfulKeywords.some(kw => text.includes(kw))) {
    return { name: 'helpfulIntent', score: 0.3, signal: 'helpful' };
  }
  
  if (directKeywords.some(kw => text.includes(kw))) {
    return { name: 'directStyle', score: 0.2, signal: 'direct' };
  }
  
  return { name: 'duckbotPersona', score: 0, signal: null };
}
```

**Benefit:** DuckBot personality preserved in routing!

---

### 4️⃣ OpenClaw Plugin Enhancement

**Priority:** MEDIUM
**Description:** Enhance OpenClaw integration with better logging and metrics

**Why:** Duckets uses OpenClaw as their primary AI platform

**Implementation:**
```typescript
// Add to src/provider.ts
interface OpenClawMetrics {
  requests: number;
  savings: number;
  modelUsage: Record<string, number>;
}

let metrics: OpenClawMetrics = {
  requests: 0,
  savings: 0,
  modelUsage: {},
};

export function trackRoute(decision: Decision): void {
  metrics.requests++;
  if (decision.savings !== undefined) {
    metrics.savings += decision.savings;
  }
  
  metrics.modelUsage[decision.model] = (metrics.modelUsage[decision.model] || 0) + 1;
  
  // Log to OpenClaw
  console.log(`📊 Route: ${decision.model} | Tier: ${decision.tier} | Savings: ${((decision.savings || 0) * 100).toFixed(0)}%`);
}

export function getMetrics(): OpenClawMetrics {
  return { ...metrics };
}

// Add periodic metrics export
setInterval(() => {
  console.log('📊 Daily Metrics:', JSON.stringify(metrics, null, 2));
  metrics = { requests: 0, savings: 0, modelUsage: {} };
}, 86400000);  // Every 24 hours
```

**Benefit:** Better cost tracking and savings visibility!

---

### 5️⃣ Multi-Model Parallel Testing

**Priority:** LOW
**Description:** Add capability to test multiple models simultaneously for quality comparison

**Why:** Ensure DuckBot gets the best model for each request type

**Implementation:**
```typescript
// Add to src/router/parallel-test.ts
interface ParallelTestResult {
  model: string;
  response: string;
  latency: number;
  quality: number;
}

async function parallelTest(
  prompt: string,
  models: string[],
  config: RoutingConfig,
): Promise<ParallelTestResult[]> {
  const results = await Promise.all(
    models.map(async (model) => {
      const start = Date.now();
      const response = await callModel(model, prompt, config);
      const latency = Date.now() - start;
      const quality = await evaluateQuality(response, prompt);
      
      return { model, response, latency, quality };
    })
  );
  
  // Sort by quality then latency
  return results.sort((a, b) => {
    if (a.quality !== b.quality) return b.quality - a.quality;
    return a.latency - b.latency;
  });
}

// Add auto-quality evaluation
async function evaluateQuality(response: string, prompt: string): Promise<number> {
  // Simple heuristic evaluation
  const length = response.length;
  const hasCode = response.includes('```');
  const relevant = isRelevant(response, prompt);
  
  let score = 0;
  if (length > 50 && length < 500) score += 1;
  if (hasCode) score += 1;
  if (relevant) score += 2;
  
  return Math.min(score, 5);  // Score out of 5
}
```

**Benefit:** Better model selection for DuckBot's queries!

---

### 6️⃣ Configuration Management UI

**Priority:** LOW
**Description:** Add web UI for configuring ClawRouter settings

**Why:** Easy configuration without editing YAML files

**Implementation:**
```typescript
// Add to src/config-ui.ts
interface ClawRouterConfig {
  lmStudioBaseUrl: string;
  localModels: string[];
  enableHealthCheck: boolean;
  healthCheckInterval: number;
  fallbackOnDegradation: boolean;
}

function renderConfigUI(): string {
  return `
<!DOCTYPE html>
<html>
<head>
  <title>ClawRouter Configuration</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    .config-group { margin-bottom: 20px; padding: 15px; border: 1px solid #ddd; border-radius: 8px; }
    label { display: block; margin-bottom: 5px; font-weight: bold; }
    input, select { width: 100%; padding: 8px; margin-bottom: 10px; }
    button { background: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 5px; cursor: pointer; }
    button:hover { background: #0056b3; }
  </style>
</head>
<body>
  <h1>🦆 ClawRouter Configuration</h1>
  <p>Configure DuckBot's intelligent routing</p>
  
  <div class="config-group">
    <h2>LM Studio Settings</h2>
    <label>Base URL:</label>
    <input type="text" id="lmStudioUrl" value="http://100.74.88.40:1234/v1" placeholder="http://localhost:1234/v1">
    
    <label>Local Models (comma-separated):</label>
    <input type="text" id="localModels" value="qwen3-coder-next,gpt-oss-20b" placeholder="model1,model2">
  </div>
  
  <div class="config-group">
    <h2>Health Monitoring</h2>
    <label>
      <input type="checkbox" id="enableHealthCheck" checked>
      Enable LM Studio health monitoring
    </label>
    
    <label>Fallback on degradation:</label>
    <select id="fallbackMode">
      <option value="cloud">Use cloud models</option>
      <option value="claudode">Use Claude Code</option>
      <option value="glm">Use GLM 4.7 API</option>
    </select>
  </div>
  
  <button onclick="saveConfig()">Save Configuration</button>
  
  <script>
    function saveConfig() {
      const config = {
        lmStudioBaseUrl: document.getElementById('lmStudioUrl').value,
        localModels: document.getElementById('localModels').value.split(','),
        enableHealthCheck: document.getElementById('enableHealthCheck').checked,
        fallbackOnDegradation: document.getElementById('fallbackMode').value,
      };
      
      fetch('/api/config', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(config),
      }).then(() => alert('Configuration saved!'));
    }
  </script>
</body>
</html>
  `;
}
```

**Benefit:** Easy configuration for DuckBot!

---

## 🚀 Implementation Priority

### Phase 1: Critical (Week 1)
1. ✅ Clone and explore repository
2. ⏳ Add LM Studio local models
3. ⏳ Add crypto-mining aware routing
4. ⏳ Test with DuckBot avatar system
5. ⏳ Commit initial changes

### Phase 2: Enhancements (Week 2)
1. ⏳ Add DuckBot personality integration
2. ⏳ Add OpenClaw metrics and logging
3. ⏳ Add parallel model testing
4. ⏳ Create configuration UI
5. ⏳ Document DuckBot-specific modifications

### Phase 3: Production (Week 3)
1. ⏳ Comprehensive testing
2. ⏳ Performance optimization
3. ⏳ Create Docker image
4. ⏳ Publish to DuckBot's fork
5. ⏳ Create migration guide

---

## 📊 Success Metrics

### What Success Looks Like:
- ✅ All LM Studio models route correctly with zero cost
- ✅ Crypto mining detected automatically, fallback triggered
- ✅ DuckBot personality preserved in routing decisions
- ✅ Cost savings tracked and reported
- ✅ OpenClaw integration seamless
- ✅ 95%+ requests routed optimally
- ✅ Average latency <2 seconds

### KPIs to Track:
- Cost savings per month
- LM Studio health percentage
- Average routing confidence
- DuckBot satisfaction (subjective)
- Number of fallbacks triggered

---

## 🤝 Integration with DuckBot Systems

### Open-LLM-VTuber (Avatar):
```typescript
// Modify characters/duckbot.yaml to use ClawRouter
character_config:
  llm_provider: 'clawrouter_proxy'  // New proxy provider
  clawrouter_config:
    endpoint: 'http://localhost:3000'  // ClawRouter proxy
    enable_local_first: true  // Prioritize local models
```

### DuckBot (Main Assistant):
```typescript
// Integrate ClawRouter as default LLM router
const defaultLLMRouter = '@blockrun/clawrouter';
```

### Cross-Device Coordination:
- DuckBot (Linux) uses ClawRouter with LM Studio on Windows (100.74.88.40)
- AgentSmithsbot (Windows) can also use ClawRouter for consistency

---

## 📝 Development Notes

### Current State:
- Repository cloned successfully
- Codebase reviewed (TypeScript, modular architecture)
- 14-dimension scoring system understood
- Payment system (x402) understood
- Testing requires funded wallet (skip for now)

### Technical Decisions:
- Use local LM Studio models first (zero cost)
- Fallback to cloud models when local is degraded
- Preserve DuckBot personality throughout routing
- Health check interval: 30 seconds
- Configuration via environment variables for security

### Dependencies:
- TypeScript 5.7+
- Node.js 18+
- (Optional) LM Studio running on Windows
- (Optional) Funded wallet for x402 testing

---

**Created:** 2026-02-06
**Author:** DuckBot Autonomous Development 🦆
**Status:** 📋 Plan ready for implementation
