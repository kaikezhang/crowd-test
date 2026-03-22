# CrowdTest — Technical Design Document

> Version: 1.0
> Date: 2026-03-22
> Status: Pre-implementation

---

## 1. 系统架构

### 1.1 总览

```
USER INPUT                    CROWDTEST PIPELINE                           OUTPUT
─────────                     ──────────────────                           ──────

                         ┌─────────────────────────┐
  Product URL ──────────►│  1. SCOUT               │
  (可选) 产品描述         │     Navigate URL         │
  (可选) 目标用户提示     │     Check page health    │
                         │     Detect auth/CAPTCHA   │
                         │     Build site_map        │
                         └──────────┬──────────────┘
                                    │
                                    ▼
                         ┌─────────────────────────┐
                         │  2. PERSONA GENERATOR    │
                         │     Input: site_map +    │
                         │       audience hint      │
                         │     Dimension matrix     │
                         │     Diversity check      │
                         │     Output: personas[]   │
                         └──────────┬──────────────┘
                                    │
                         ┌──────────┴──────────────┐
                         │  For each persona:       │
                         │                          │
                         │  ┌─────────────────────┐ │
                         │  │ 3. BROWSER SESSION   │ │
                         │  │    Init context      │ │
                         │  │    Navigate product   │ │
                         │  │    Explore as persona │ │──► action_log
                         │  │    Log all actions    │ │
                         │  └─────────────────────┘ │
                         │                          │
                         │  ┌─────────────────────┐ │
                         │  │ 4. FEEDBACK WRITER   │ │──► feedback{}
                         │  │    Review action log │ │
                         │  │    Write grounded    │ │
                         │  │    feedback          │ │
                         │  └─────────────────────┘ │
                         └──────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────────┐
                         │ 5. AGGREGATOR            │
                         │    Dedup issues          │
                         │    Score severity        │──► Report
                         │    Cross-persona patterns │   (MD / HTML)
                         │    Generate report       │
                         └─────────────────────────┘
```

### 1.2 技术栈

| 层 | 技术 | 原因 |
|----|------|------|
| 浏览器自动化 | Playwright MCP (`@playwright/mcp`) | 成熟、21 个浏览器工具、accessibility snapshot |
| 页面观察 | Accessibility Snapshots | Token 高效 (~200-500 tokens vs 1500+ screenshot)、确定性 |
| LLM — Persona 行为 | Claude Sonnet 4.6 | 性价比最优，$3/1M input |
| LLM — 反馈综合 | Claude Sonnet 4.6 | 够用，除非用户选 Opus |
| LLM — 聚合去重 | Claude Opus 4.6 (可选) | 跨 persona 综合推理更强 |
| 执行模式 | 串行 (v1) | 可靠、简单、10 personas < 10min |
| 输出格式 | Markdown (v1) / HTML (v2) | LLM 天然擅长生成 MD |
| 打包形式 | OpenClaw Skill (SKILL.md) | 零依赖安装，prompt-as-code |

### 1.3 不构建什么

- ❌ 自建浏览器自动化层（用 Playwright MCP）
- ❌ Persona 数据库（每次按产品即时生成）
- ❌ Web 服务器/Dashboard（输出是文件）
- ❌ 自建评估框架（借鉴 ArkSim 模式）
- ❌ 多机编排（单机 10-20 personas 够用）

---

## 2. 核心数据结构

### 2.1 SiteMap (Scout 输出)

```typescript
interface SiteMap {
  url: string
  title: string
  status: "ready" | "auth_required" | "captcha_detected" | "load_failed"
  pages: {
    url: string
    type: "landing" | "form" | "dashboard" | "content" | "pricing" | "other"
    interactive_elements: number
    key_elements: string[]  // 主要交互元素描述
  }[]
  auth_type?: "login_form" | "oauth" | "unknown"
  cookie_banner: boolean
  tech_stack?: string[]  // 检测到的框架 (React, Vue, etc.)
}
```

### 2.2 Persona

```typescript
interface Persona {
  id: string                    // "persona_001"
  name: string                  // "Maria Chen"
  tech_level: "novice" | "intermediate" | "advanced" | "developer"
  purpose: string               // 使用产品的目的
  age: number
  industry: string
  personality: {
    patience: number            // 0.0-1.0
    exploration_tendency: number
    attention_to_detail: number
  }
  behavioral_rules: string[]    // 硬约束（如"3次失败后放弃"）
  device: "desktop" | "mobile" | "tablet"
  narrative: string             // 完整的 prompt 段落
}
```

### 2.3 ActionLog

```typescript
interface ActionLog {
  persona_id: string
  actions: {
    step: number
    type: "navigate" | "click" | "type" | "scroll" | "wait" | "back"
    target: string              // 元素描述
    snapshot_before: string     // accessibility snapshot
    snapshot_after: string
    success: boolean
    notes: string               // LLM 的 in-character 观察
  }[]
  duration_seconds: number
  pages_visited: string[]
  abandoned: boolean            // 是否中途放弃
  abandon_reason?: string
}
```

### 2.4 PersonaFeedback

```typescript
interface PersonaFeedback {
  persona_id: string
  first_impression: string      // 前 30 秒感受
  journey_narrative: string     // 使用过程叙述
  issues: {
    page: string
    element: string
    issue: string
    severity: "critical" | "high" | "medium" | "low"
    evidence: string            // 引用 action log step
  }[]
  positives: { feature: string, why: string }[]
  nps_score: number             // 1-10
  would_pay: boolean
  would_return: boolean
  one_line_verdict: string
}
```

### 2.5 AggregatedReport

```typescript
interface AggregatedReport {
  product_url: string
  test_date: string
  personas_count: number
  avg_nps: number
  pay_rate: number              // would_pay 的百分比
  return_rate: number           // would_return 的百分比
  unique_issues: {
    id: string
    description: string
    severity: "critical" | "high" | "medium" | "low"
    personas_affected: number
    total_personas: number
    evidence: string[]          // 各 persona 的证据
  }[]
  consensus_positives: {
    feature: string
    mention_count: number
  }[]
  segment_insights: {           // 按 persona 维度分组的洞察
    segment: string             // "novice users" / "mobile users"
    key_finding: string
  }[]
}
```

---

## 3. 模块详细设计

### 3.1 Scout (页面预检)

**目的**: 在跑任何 persona 之前，确认页面可用。不做这步 = 30%+ 的运行产出垃圾。

**流程**:

```
1. browser_navigate(url)
2. browser_wait_for(networkIdle, 10s timeout)
3. browser_snapshot()
4. 检查:
   a. interactive_elements > 5? (否则 → 页面可能没加载完)
   b. 有 "sign in" / "log in" 等元素? (→ auth_required)
   c. 有 reCAPTCHA / hCaptcha? (→ captcha_detected)
   d. 有 cookie banner? (→ 自动关闭)
5. 如果状态不是 "ready" → 提示用户并终止
6. 快速探索 top-level 导航 → 建 site_map
```

**Auth 处理**:
- 检测到 auth → 提示用户提供 `--storage-state` JSON 文件
- Playwright `--storage-state` 格式包含 cookies + localStorage

**CAPTCHA 处理**:
- 检测到 → 告知用户 CrowdTest 无法绕过 CAPTCHA
- 建议：在 staging 环境测试，或提供 pre-auth cookies

### 3.2 Persona Generator

**输入**: site_map + (可选) audience_hint
**输出**: Persona[]

**生成策略**:

```
Step 1: 分析产品
  - 从 site_map 推断产品类型、目标用户、核心功能
  - 如果有 audience_hint，作为额外输入

Step 2: 构建维度矩阵
  - tech_level: 从4个等级各取样
  - purpose: 从产品分析推断 3-5 种使用目的
  - age: 覆盖至少 3 个年龄段
  - personality: 随机采样，确保 variance

Step 3: 生成 Personas
  - 每个 persona 一次 LLM call
  - 累积多样性: 把已生成的 persona 摘要放入 prompt
  - 指令: "Fill gaps in the existing collection"
  
Step 4: 多样性检查
  - < 20 personas: 累积 prompt 足够
  - > 20 personas: 用 embedding cosine similarity，reject > 0.85
```

**行为规则生成** (关键！):

不只是性格描述，要生成**可执行的行为约束**:

| 性格维度 | 行为规则示例 |
|----------|------------|
| patience: 0.2 | "After 2 failed clicks, express frustration. After 5 total, abandon." |
| exploration: 0.9 | "Visit at least 4 different pages. Click on non-obvious elements." |
| attention: 0.3 | "Skim headings only. Skip paragraphs longer than 3 lines." |
| tech_level: novice | "Ignore elements with technical jargon. Only use visible buttons/links." |

### 3.3 Browser Session (Per-Persona)

**状态机**:

```
INIT ──► NAVIGATE ──► ORIENT ──► ACT ◄──► OBSERVE ──► DECIDE ──► REFLECT ──► DONE
              │                    │                       │
              └── RETRY (max 2) ◄─┘                       │
                                                          │
                                               RECOVER ◄──┘ (action failed)
```

**INIT**: 设置浏览器上下文
```
- device emulation (desktop/mobile/tablet)
- viewport size
- locale (如果 persona 有地区设定)
```

**NAVIGATE**: 打开目标 URL
```
- browser_navigate(url)
- Wait for network idle
- 如果失败 → RETRY (max 2)
- Dismiss cookie banner if present
```

**ORIENT**: 观察当前页面
```
- browser_snapshot()
- 识别可交互元素
- 基于 persona 的 purpose，计划下一步动作
```

**ACT**: 执行一个动作
```
- click / type / scroll / navigate
- 等待 500ms
- 重新 snapshot
```

**OBSERVE**: 检查结果
```
- snapshot 是否变化？（action 是否生效）
- 如果未变化 → RECOVER（尝试替代方式）
- 记录到 action_log
```

**DECIDE**: 继续还是停止？
```
- actions < 20 AND time < 3min → 继续 (→ ORIENT)
- 目标达成 / 卡住 / 到限制 → REFLECT
```

**REFLECT**: 回顾 action log，产出反馈
```
- 这是 Phase 2（feedback writing）的输入
```

**每个 action 的 LLM prompt 结构**:

```
System: You are {persona.narrative}. 
        Behavioral rules: {persona.behavioral_rules}
        
Context: You are testing {product_url}.
         Your goal: {persona.purpose}
         
Current page snapshot: {accessibility_snapshot}
Action history: {last_5_actions_summary}
Previously covered issues: {already_found_issues}  // 累积多样性

Task: Decide your next action. Choose from:
  - click(element) 
  - type(element, text)
  - scroll(direction)
  - navigate(url)
  - done(reason)

Respond as JSON: { "action": "...", "target": "...", "reasoning": "..." }
```

### 3.4 Feedback Writer

**输入**: Persona + ActionLog
**输出**: PersonaFeedback

**三阶段流程**:

```
Phase 1: action log 已在 Browser Session 中生成 ✓

Phase 2: 反馈撰写
  - LLM 收到完整 action log
  - Prompt: "回顾你的使用过程，作为 {persona}，写出真实反馈"
  - 要求引用具体 step 作为 evidence
  - Temperature: 0.3（保持连贯）

Phase 3: 结构化提取
  - 从 Phase 2 的叙述中提取 JSON 格式
  - 严重度分级
  - Evidence 必须指向 action log step
```

**反馈自验证** (canary check):

```
在生成反馈后，额外一次 LLM call:
"Review this feedback. Is it specific and grounded in the action log,
 or is it generic and could apply to any product?"

如果判定为 generic → regenerate（最多 retry 1 次）
```

### 3.5 Aggregator

**输入**: PersonaFeedback[]
**输出**: AggregatedReport

**去重流程** (借鉴 ArkSim):

```
1. 收集所有 persona 的 issues
2. LLM 去重: 语义相同的 issue 合并为一个 unique_issue
3. 每个 unique_issue 记录:
   - 受影响的 persona 数量
   - 各 persona 的证据
   - 综合严重度（最高严重度 + 受影响人数加权）
4. 按严重度排序

严重度加权公式:
  severity_score = base_severity × (1 + log2(affected_personas))
  
  base_severity: critical=4, high=3, medium=2, low=1
```

**跨维度分析**:

```
按 tech_level 分组 → novice vs developer 看到不同的问题？
按 device 分组 → mobile 用户有额外问题？
按 purpose 分组 → 不同使用目的的满意度差异？
```

---

## 4. Prompt 设计

### 4.1 Persona 生成 Prompt

```
You are a user research expert creating diverse test personas for a web product.

Product: {site_map.title} ({site_map.url})
Product type: {inferred_type}
Key features: {site_map.pages[].key_elements}
Target audience hint: {audience_hint || "general web users"}

Already generated personas:
{previous_personas_summaries}

Generate 1 new persona that FILLS GAPS in the existing collection.
The persona must be meaningfully different from all existing ones
across at least 2 dimensions (tech_level, age, purpose, personality).

Output as JSON:
{persona_schema}

CRITICAL: Include behavioral_rules — concrete, testable constraints
on how this persona acts when testing software. NOT personality
adjectives, but IF-THEN rules.
```

### 4.2 Browser Action Prompt

```
You are {persona.name}. {persona.narrative}

BEHAVIORAL RULES (you MUST follow these):
{persona.behavioral_rules}

You are exploring {product_url} for the first time.
Your goal: {persona.purpose}

Current page:
{accessibility_snapshot}

Your action history (last 5):
{recent_actions}

Issues already found by other testers (find DIFFERENT ones):
{covered_issues}

What do you do next? Think in character, then choose ONE action:
- click("element description")
- type("element", "text to type")  
- scroll("up" | "down")
- navigate_back()
- done("reason for stopping")

Respond as:
{
  "thinking": "your in-character thought process",
  "action": "click",
  "target": "element description from snapshot",
  "observation": "what you expect to happen"
}
```

### 4.3 Feedback Writing Prompt

```
You just finished testing {product_url} as {persona.name}.

Here is your complete action log:
{action_log}

Now write your honest feedback AS THIS PERSONA. Requirements:
1. Your first_impression is about the FIRST 30 seconds only
2. Every issue MUST reference a specific action log step as evidence
3. Be specific — "button X on page Y" not "the UI could be better"
4. Include at least 1 positive (if any exist)
5. NPS score must reflect your overall experience
6. would_pay and would_return are YOUR honest assessment as this persona

Output as JSON:
{feedback_schema}
```

### 4.4 Aggregation Prompt

```
You are analyzing feedback from {N} different user personas 
who tested {product_url}.

All persona feedbacks:
{all_feedbacks}

Tasks:
1. DEDUPLICATE: Group semantically identical issues.
   Different personas may describe the same problem differently.
   
2. SEVERITY: Rate each unique issue as critical/high/medium/low.
   Critical = blocks core functionality
   High = significantly degrades experience
   Medium = noticeable but workaround exists
   Low = minor annoyance
   
3. CONSENSUS: How many personas mentioned each issue?
   Issues mentioned by >50% are CONSENSUS issues.
   
4. SEGMENTS: Are there patterns by persona type?
   e.g., "all novice users struggled with X"
   
5. POSITIVES: What do multiple personas agree is good?

Output as JSON:
{aggregated_report_schema}
```

---

## 5. 性能与资源

### 5.1 资源预算

| 资源 | 5 personas | 10 personas | 20 personas |
|------|-----------|-------------|-------------|
| RAM | ~500MB | ~800MB | ~1.5GB |
| 浏览器实例 | 1 (串行) | 1 (串行) | 1 (串行) |
| LLM 调用 | ~80 | ~160 | ~320 |
| 时间 | ~5 min | ~10 min | ~20 min |
| 成本 (Sonnet) | ~$0.30 | ~$0.60 | ~$1.20 |
| 成本 (Opus) | ~$1.50 | ~$3.00 | ~$6.00 |

### 5.2 并行策略 (v2)

```
v1: 串行执行（1 browser, 1 persona at a time）
v2: Sub-agent 并行（3-5 parallel browsers）

并行约束:
- 每个 browser ~200MB RAM
- 5 并行 = ~1GB 额外 RAM
- 10 并行 = 风险超出单机能力
- 目标网站可能 rate limit → max 5 并行
```

### 5.3 LLM Rate Limit

```
Anthropic Sonnet rate limits (standard tier):
- ~4,000 RPM
- ~400K input TPM

10 personas 串行: ~1 req/5sec = 12 RPM → 远低于限制
50 personas 5 并行: ~60 RPM → 安全
```

---

## 6. 错误处理

### 6.1 页面加载失败

```
Retry 1: 等 3s → 重新 navigate
Retry 2: 等 5s → 重新 navigate
仍然失败 → 报告 "Page failed to load" + 终止该 persona
```

### 6.2 Action 失败

```
Snapshot 未变化 → RECOVER:
  1. 尝试替代 selector (如 parent element)
  2. 尝试 scroll 后重试
  3. 仍然失败 → 记录为 "action failed" + 继续下一步
```

### 6.3 LLM 输出不合规

```
JSON 解析失败 → retry 一次 (temperature 降到 0.1)
仍然失败 → 跳过该步骤 + 记录 warning
```

### 6.4 Circuit Breaker

```
单个 persona session:
  - > 20 actions → 强制停止
  - > 3 min → 强制停止
  - > 5 consecutive failures → 强制停止 + 标记 "session_unstable"
```

---

## 7. 可扩展性设计

### 7.1 自定义维度

用户可以在配置中添加领域特定的 persona 维度:

```json
{
  "custom_dimensions": {
    "trading_experience": ["beginner", "1-3yr", "3-10yr", "10yr+"],
    "account_size": ["<10K", "10-50K", "50-200K", "200K+"]
  }
}
```

### 7.2 自定义评估指标

借鉴 ArkSim 的 Metric ABC 模式:

```
用户可以定义:
- 额外的量化指标 (1-5 scale)
- 额外的定性指标 (category labels)
- 自定义 severity 规则
```

### 7.3 报告模板

```
v1: 内置 Markdown 模板
v2: 用户提供自定义报告模板 (Jinja2/Handlebars)
v3: 内置 HTML 模板 (参考 ArkSim standalone HTML)
```

### 7.4 A/B 测试 (v2+)

```
crowdtest --compare https://v1.myapp.com https://v2.myapp.com
```

同一组 personas 分别测试两个版本，输出对比报告。

---

## 8. 站点类型兼容性

| 站点类型 | 预期成功率 | 问题 |
|---------|-----------|------|
| 静态营销站 | 90%+ | 最佳场景 |
| Server-rendered (Rails, Django) | 80%+ | 通常 OK |
| SPA (React, Vue, Angular) | 50-70% | hydration 延迟，client routing |
| 需要登录 | 0% (无 setup) | 需要 `--storage-state` |
| Localhost / 开发服务器 | 70%+ | 需要服务器运行中 |
| 有 CAPTCHA | 0% | 无法绕过 |
| VPN/Firewall 后 | 0% | 浏览器无法访问 |

---

## 9. 测试策略

### 9.1 自验证 Test Fixtures

创建 3-5 个植入已知 bug 的测试页面:

```
Fixture 1: TodoMVC (broken)
  - Bug 1: Add 按钮白色背景上白色字 (critical)
  - Bug 2: Delete 只在 hover 显示 (medium)
  - Bug 3: Filter 没有标签 (low)
  - Bug 4: Clear completed 没有确认 (medium)
  
  Pass: 发现 ≥ 3/4

Fixture 2: Signup Flow (confusing)
  - Bug 1: 邮箱验证拒绝 .co TLD (high)
  - Bug 2: 密码规则不展示 (medium)
  - Bug 3: "Free Trial" 按钮跳转到付费页 (critical)
  
  Pass: 发现 ≥ 2/3
```

### 9.2 反馈质量 Canary

每次运行自动执行:

```
对每条反馈问 LLM:
"Is this feedback specific and grounded, or generic?"

Threshold: ≥ 80% 被判为 specific
低于阈值 → 报告中标注 warning
```

### 9.3 回归测试

每次代码变更后在 test fixtures 上运行，确保发现率不退化。

---

*Architecture is trade-offs. This doc prioritizes reliability over speed, specificity over coverage, and honest limitations over false promises.*
