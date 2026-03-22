# CrowdTest — Product Requirements Document (PRD)

> Version: 1.0
> Date: 2026-03-22
> Status: Pre-implementation

---

## 1. 产品概述

### 1.1 一句话描述

**CrowdTest** 是一个开源 AI 工具，用一条命令让 N 个仿真用户测试你的网站，找出你看不到的 UX 问题。

```
crowdtest https://myapp.com
```

### 1.2 核心价值主张

> 当你没有用户、没有预算、没有时间时，CrowdTest 是唯一可用的 UX sanity check。

- **60-70%** 的真人反馈质量
- **1%** 的成本
- **100x** 的速度

### 1.3 不是什么

- ❌ 不是替代真人用户测试（承认 15% 的 fidelity gap）
- ❌ 不是视觉设计评审工具（看不到颜色/布局/动画）
- ❌ 不是性能测试工具（不测 Core Web Vitals）
- ❌ 不是安全测试工具
- ❌ 不是无障碍审计工具（无法模拟辅助技术）

---

## 2. 目标用户

### 2.1 Primary — Solo Founder (pre-PMF)

**画像**: 独立开发者，刚上线一个 B2C Web 产品，DAU < 50

**场景**: 晚上 11 点重做了 onboarding flow，没信心能用。想要 fresh eyes 但身边没人可问。

**痛点**:
- Curse-of-knowledge blindness — 自己太了解产品，无法模拟新手
- 没预算请 UX consultant ($150-300/session)
- 没用户给 Hotjar/FullStory 跑数据
- Product Hunt 上线后 95% Day-1 churn，知道是 UI 问题但不知道具体是什么

**触发时刻**: "我需要别人看看我的产品到底哪里有问题"

### 2.2 Secondary — 小团队 (2-5 人)

**画像**: 小型 SaaS 团队，没有专职 QA 或设计师

**场景**: 即将发布大功能更新，需要 sanity check

**痛点**:
- 4 个工程师 dogfood = 4 个带有 curse of knowledge 的视角
- 招真人测试要协调 1-3 天，功能今晚要发

### 2.3 NOT the Customer (Yet)

| 用户类型 | 为什么不是 |
|---------|-----------|
| 企业团队 | 有 UX 研究员、UserTesting.com 预算、Figma prototyping |
| Mobile 开发者 | 浏览器自动化帮不上忙 |
| Agency | 他们靠 UX 工作收费，不需要自动化 |

---

## 3. 竞争格局

### 3.1 替代方案对比

| 替代方案 | 成本 | 质量 | 速度 | CrowdTest 优势 |
|---------|------|------|------|---------------|
| 找 5 个朋友看 | 免费 | 低（客气偏差） | 1-3 天 | 即时、无社交债务、多样 persona |
| Reddit/HN 发帖 | 免费 | 不稳定 | 不可预测 | 结构化、可重复、私密 |
| UserTesting.com | $49/人 × 5 = $245 | 高 | 2-24 小时 | 10x 便宜，即时，但质量稍低 |
| UX Consultant | $150-300/session | 很高 | 数天 | 不是替代品，是预筛 |
| Hotjar/FullStory | 免费 tier | 真实数据 | 需要流量 | 零用户也能用 |

### 3.2 最近竞争者

- **Blok** — $7.5M 融资，做 AI persona 测试，但面向 PM 不面向开发者，SaaS 不开源
- **CrowdTest 差异化**: 开源 + CLI/Skill + 本地运行 + 开发者工具

### 3.3 无人覆盖的空白

没有任何项目同时做到：**persona 生成 + 浏览器自动化 + 结构化产品反馈**

---

## 4. 核心功能

### 4.1 MVP (v0.2) — "The Demo That Sells"

**一条命令，零配置**:

```bash
crowdtest https://myapp.com
```

**输出**: 终端报告

```
╔══════════════════════════════════════════════════╗
║  CrowdTest Report — myapp.com                    ║
║  5 personas · 47 pages visited · 12 issues found ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  🔴 CRITICAL (2)                                  ║
║  ┌─────────────────────────────────────────────┐ ║
║  │ Signup form has no error message on invalid  │ ║
║  │ email — 4/5 personas were confused           │ ║
║  └─────────────────────────────────────────────┘ ║
║                                                  ║
║  🟡 MODERATE (4)                                  ║
║  🟢 MINOR (6)                                     ║
╚══════════════════════════════════════════════════╝
```

#### 功能清单

| 功能 | 优先级 | 描述 |
|------|--------|------|
| Scout (页面预检) | P0 | 检测页面是否加载、是否需要登录、是否有 CAPTCHA |
| Persona 生成器 | P0 | 维度矩阵 → N 个独特用户画像 |
| 单 persona 浏览 | P0 | Playwright 驱动浏览器，记录 action log |
| Grounded 反馈提取 | P0 | 基于 action log 的结构化反馈（非 generic slop） |
| 多 persona 串行执行 | P1 | 依次跑 5-10 个 persona |
| 反馈去重聚合 | P1 | 跨 persona 去重、严重度排序、共识检测 |
| Markdown 报告 | P1 | 生成可读的最终报告 |
| SKILL.md 打包 | P2 | OpenClaw / Claude Code Skill 格式 |
| 自验证测试套件 | P2 | 植入已知问题的测试 fixture 验证发现率 |
| HTML 交互报告 | P3 | 带 drill-down 的 standalone HTML |
| 并行 persona 执行 | P3 | Sub-agent 并行跑多个 persona |
| A/B 测试 | P3 | 同一组 persona 测试两个 URL 版本 |

### 4.2 Persona 系统

#### 维度矩阵

| 维度 | 值域 |
|------|------|
| 技术水平 | novice / intermediate / advanced / developer |
| 使用目的 | 从产品 URL 推断 |
| 年龄段 | 18-24 / 25-34 / 35-49 / 50-64 / 65+ |
| 行业 | 从产品类型推断 |
| 性格-耐心度 | 0.0-1.0 |
| 性格-探索倾向 | 0.0-1.0 |
| 性格-细节关注度 | 0.0-1.0 |
| 设备 | desktop / mobile / tablet |

#### Persona 输出格式

```json
{
  "id": "persona_001",
  "name": "Maria Chen",
  "tech_level": "intermediate",
  "purpose": "price comparison shopping",
  "age": 34,
  "industry": "healthcare",
  "personality": {
    "patience": 0.3,
    "exploration_tendency": 0.8,
    "attention_to_detail": 0.6
  },
  "behavioral_rules": [
    "After 3 failed attempts at any task, express frustration and try a different approach",
    "After 5 total failures, abandon the task",
    "Ignore elements with developer jargon"
  ],
  "device": "mobile",
  "narrative": "You are Maria Chen, a 34-year-old nurse who..."
}
```

#### 关键设计决策

- **行为规则 > 性格描述**: "3 次失败后放弃" 比 "你是个不耐烦的人" 效果好得多
- LLM 太聪明了，无法真正"变笨"，所以用**行为约束**代替**认知降级**
- 每个新 persona 的反馈必须标注已有反馈为 "已覆盖"，强制发现新问题

### 4.3 浏览器会话

#### 状态机

```
INIT → NAVIGATE → ORIENT → ACT ⟷ OBSERVE → DECIDE → REFLECT → DONE
                                      ↓
                                   RECOVER (失败时)
```

#### 约束

| 参数 | 值 | 原因 |
|------|-----|------|
| 最大 action 数 | 20 | 防止 runaway session |
| 最大时间 | 3 分钟 | 真人用户形成印象很快 |
| 观察模式 | Accessibility Snapshot | 比截图便宜 3-5x，更确定性 |
| 重试次数 | 2 | 页面加载失败时重试 |

### 4.4 反馈质量保证 — 三阶段流程

```
Phase 1: 浏览 + 记录 action log（ground truth）
Phase 2: 回顾 action log，写 grounded 反馈（不是凭空编）
Phase 3: 结构化提取（JSON + severity + evidence）
```

**跳过 Phase 1 = generic slop**。这是质量的生命线。

### 4.5 反馈数据结构

```json
{
  "persona_id": "persona_001",
  "first_impression": "...",
  "journey_narrative": "...",
  "issues": [{
    "page": "/pricing",
    "element": "CTA button",
    "issue": "Says 'Start Free' but leads to payment",
    "severity": "critical",
    "evidence": "Action log step 7: clicked button → redirected to /checkout"
  }],
  "positives": [{ "feature": "...", "why": "..." }],
  "nps_score": 6,
  "would_pay": false,
  "would_return": true,
  "one_line_verdict": "..."
}
```

---

## 5. 成功指标

### 5.1 产品质量指标

| 指标 | 目标 | 测量方式 |
|------|------|---------|
| 已知问题发现率 | ≥ 75% (4/4) | 在植入已知 bug 的 test fixture 上运行 |
| 反馈特异性 | ≥ 80% | LLM 自验证："这条反馈是 specific 还是 generic？" |
| 跨 persona 去重准确率 | ≥ 70% | 手动抽查 |
| Scout 成功率 | ≥ 90% | 公开静态站成功加载 |

### 5.2 用户体验指标

| 指标 | 目标 |
|------|------|
| 首次使用 time-to-value | < 15 分钟（从安装到看到报告） |
| 10 persona 运行时间 | < 10 分钟 |
| 10 persona 运行成本 | < $2（Sonnet） |
| "发现了我不知道的问题" 比率 | ≥ 50%（至少一半用户报告有新发现） |

---

## 6. 成本模型

### 6.1 单次运行成本（Sonnet 4.6）

| 组件 | Token | 成本 |
|------|-------|------|
| Persona 生成 (10个) | ~20K | ~$0.06 |
| 浏览器会话 (10 personas × ~12 actions) | ~150K | ~$0.45 |
| 反馈生成 (10个) | ~20K | ~$0.06 |
| 聚合去重 | ~10K | ~$0.03 |
| **Total** | **~200K** | **~$0.60** |

### 6.2 Opus 升级

用 Opus 4.6 提升质量：~$3-6/次（5x Sonnet）

### 6.3 定价锚点

- UserTesting.com: $49/人 × 5 = **$245**
- CrowdTest: 10 AI personas = **~$1**
- **25x 成本优势**

---

## 7. 分发策略

### 核心渠道：Agent Skills 生态

CrowdTest 遵循 [Agent Skills](https://agentskills.io) 开放标准，一份 SKILL.md 兼容 15+ AI 工具：
- Claude Code, Cursor, Gemini CLI, VS Code Copilot, OpenCode, Goose, Amp, Junie, OpenHands, Firebender, Letta, Mux...
- 安装: `npx skills add kaikezhang/crowd-test`
- 也兼容 OpenClaw Skill（同一份文件）

### Phase 1: Demo 驱动（前 50 用户）

- 录 2 分钟 demo 视频："我让 20 个 AI 用户测试我的 landing page，发现了 4 个我从没注意的问题"
- 发到 Indie Hackers, r/SideProject, r/webdev, HN Show
- **Demo 就是营销** — 让人看到报告输出，不是技术架构
- 提交到 skills.sh directory（Agent Skills 目录）

### Phase 2: 工作流集成（50-200 用户）

- GitHub Action: CI/CD 集成（PR 自动跑 CrowdTest）
- Pre-commit hook: 每次部署前自动检查

### Phase 3: 口碑传播（200+ 用户）

- "CrowdTest found this bug" 截图是天然传播素材

### 不会有效的

- SEO（被 UserTesting.com 等大站淹没）
- Cold outreach（市场太分散）
- 只靠 Skill marketplace（流量不够，但是必要的基础设施）

---

## 8. 风险矩阵

| 风险 | 可能性 | 影响 | 缓解 |
|------|--------|------|------|
| 反馈是 generic slop | 中 | 致命 | 三阶段 grounded 反馈 + 自验证 |
| 浏览器在复杂 SPA 上崩溃 | 高 | 高 | Scout 预检 + 优雅降级 |
| 所有 persona 给同样反馈 | 中 | 高 | 累积多样性 prompt |
| Blok ($7.5M) 做同样的事 | 高 | 高 | 开源 + CLI + 本地运行 差异化 |
| LLM 太聪明无法模拟笨用户 | 高 | 中 | 行为规则代替性格描述 |

---

## 9. 成功标准

**MVP 的 bar**: 一个 founder 在自己的产品上跑 CrowdTest，说"它发现了至少 1 个我之前没注意到的问题"。

**v1.0 的 bar**: 10 个不同产品上运行，平均每次发现 ≥ 2 个 founder 不知道的问题，反馈特异性 ≥ 80%。

---

*Product is truth. Ship the Scout, validate the feedback, then scale up.*
