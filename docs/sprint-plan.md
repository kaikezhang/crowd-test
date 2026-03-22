# CrowdTest — Sprint Plan

> Updated: 2026-03-22
> Scope: v0.2 → v0.4 detailed task breakdown

---

## Sprint 1: Scout + Persona Generator (P0)

**Duration**: 2-3 days
**Goal**: 能自动检测页面状态 + 生成多样化 persona

### Task 1.1: Scout — Page Readiness

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 1.1.1 | P0 | 2h | 基础导航: `browser_navigate` + `browser_wait_for(networkIdle)` + `browser_snapshot` |
| 1.1.2 | P0 | 1h | 健康检查: 计算 interactive elements, 阈值 < 5 → retry |
| 1.1.3 | P0 | 2h | Auth 检测: 在 snapshot 中搜索 sign in/log in 模式 |
| 1.1.4 | P0 | 1h | CAPTCHA 检测: 搜索 captcha/recaptcha/hcaptcha 模式 |
| 1.1.5 | P0 | 2h | Cookie banner 处理: 检测 + 自动点击 Accept/Close |
| 1.1.6 | P1 | 2h | 多页探索: 跟踪 top-level nav links, 快速 snapshot 前 3-5 页 |
| 1.1.7 | P1 | 1h | SiteMap 输出: 组装 JSON, 验证 schema |

**Acceptance**: Scout 在 3 个不同站点上正确判断 ready/auth/captcha

### Task 1.2: Persona Generator

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 1.2.1 | P0 | 2h | 产品分析: 从 SiteMap 推断产品类型、目标用户、核心功能 |
| 1.2.2 | P0 | 3h | 维度矩阵: 定义 tech_level × purpose × age × personality × device 采样策略 |
| 1.2.3 | P0 | 3h | 单 persona 生成: LLM prompt → Persona JSON, 包含 behavioral_rules |
| 1.2.4 | P0 | 2h | 累积多样性: 每次生成把已有 persona 摘要放入 prompt |
| 1.2.5 | P1 | 1h | 多样性验证: 确保无两个 persona 共享 (tech_level, purpose, device) |
| 1.2.6 | P1 | 1h | 批量生成: 循环 N 次, 输出 Persona[] JSON array |

**Acceptance**: 生成 10 个 persona, 均有 ≥3 条 IF-THEN 行为规则, 覆盖 ≥3 tech_level + ≥3 age range

### Sprint 1 交付物

- Scout prompt module (可独立运行)
- Persona Generator prompt module (可独立运行)
- 示例输出 JSON (SiteMap + 10 Personas)
- PR (不 merge)

---

## Sprint 2: Single Persona Browser Loop (P0)

**Duration**: 3-4 days
**Goal**: 一个 persona 能完整走完 浏览 → 反馈 流程
**Dependency**: Sprint 1 (需要 SiteMap + Persona)

### Task 2.1: Browser Session Core

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.1.1 | P0 | 2h | INIT: 设置 browser context (device, viewport, locale) |
| 2.1.2 | P0 | 2h | NAVIGATE: 打开 URL + 等待 + cookie banner 处理 (复用 Scout) |
| 2.1.3 | P0 | 3h | ORIENT: snapshot 当前页, 基于 persona purpose 规划下一步 |
| 2.1.4 | P0 | 3h | ACT: 执行 click/type/scroll, 等 500ms, 重新 snapshot |
| 2.1.5 | P0 | 2h | OBSERVE: 对比前后 snapshot, 判断 action 是否生效 |
| 2.1.6 | P1 | 2h | RECOVER: action 失败 → 尝试替代方式 (scroll, parent element) |
| 2.1.7 | P0 | 2h | DECIDE: 继续/停止 逻辑 (goal met / stuck / limit hit) |
| 2.1.8 | P0 | 1h | ActionLog: 记录每步 (action, target, snapshots, success, notes) |

### Task 2.2: Circuit Breakers

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.2.1 | P0 | 1h | Max 20 actions per session |
| 2.2.2 | P0 | 1h | Max 3 minutes per session |
| 2.2.3 | P1 | 1h | Max 5 consecutive failures → force stop |

### Task 2.3: Feedback Writer

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.3.1 | P0 | 3h | Phase 2: LLM 回顾 action log, 写 grounded 反馈 |
| 2.3.2 | P0 | 2h | Phase 3: 结构化提取 → PersonaFeedback JSON |
| 2.3.3 | P1 | 1h | Evidence linking: 每个 issue 必须引用 action log step |
| 2.3.4 | P1 | 2h | Canary check: LLM 自验证 "specific or generic?" |
| 2.3.5 | P1 | 1h | Generic feedback retry: 最多 1 次 regeneration |

**Acceptance**: 单 persona 在 Event Radar 上完整跑通, 产出 grounded feedback with evidence

### Sprint 2 交付物

- Browser session state machine (完整 ORIENT → ACT → OBSERVE → DECIDE loop)
- Action logging 系统
- Feedback writer (three-phase)
- Canary self-validation
- End-to-end 单 persona demo
- PR (不 merge)

---

## Sprint 3: Multi-Persona + Aggregation (P1)

**Duration**: 3-4 days
**Goal**: N 个 persona 串行运行, 产出去重聚合报告
**Dependency**: Sprint 2

### Task 3.1: Multi-Persona Execution

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.1.1 | P0 | 2h | Sequential runner: for each persona → run browser session |
| 3.1.2 | P0 | 2h | Cumulative diversity: 传递 "already covered issues" 给后续 persona |
| 3.1.3 | P1 | 2h | Progress reporting: 每个 persona 完成后输出进度 |
| 3.1.4 | P1 | 1h | 错误隔离: 单个 persona 失败不影响其他 |

### Task 3.2: Aggregator

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.2.1 | P0 | 3h | Issue 去重: LLM 语义合并相同问题 |
| 3.2.2 | P0 | 2h | Severity 计算: base_severity × (1 + log2(affected_count)) |
| 3.2.3 | P0 | 2h | Consensus 检测: >50% persona 提到 = consensus issue |
| 3.2.4 | P1 | 2h | 跨维度分析: 按 tech_level / device / purpose 分组洞察 |
| 3.2.5 | P1 | 1h | Positives aggregation: 共识优点 |

### Task 3.3: Report Generation

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.3.1 | P0 | 3h | Markdown report: 完整模板 (header, critical/high/medium/low, segments) |
| 3.3.2 | P1 | 2h | 终端 summary: 精简版 (issue count + top 3 critical) |
| 3.3.3 | P2 | 4h | HTML report: standalone 交互式页面 (drill-down) |

### Sprint 3 交付物

- Multi-persona runner
- Aggregator (dedup + severity + consensus)
- Markdown report
- End-to-end: 10 personas on real site → report
- PR (不 merge)

---

## Sprint 4: Packaging + Validation (P2)

**Duration**: 2-3 days
**Goal**: 打包成可发布的 Skill, 有质量保证

### Task 4.1: SKILL.md

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 4.1.1 | P0 | 4h | 主 SKILL.md: 完整的 orchestration prompt |
| 4.1.2 | P0 | 2h | 依赖检查: Playwright MCP 可用性验证 |
| 4.1.3 | P0 | 1h | 配置选项: persona_count, model, verbose |
| 4.1.4 | P1 | 2h | 错误信息: 所有失败场景的 user-friendly 提示 |

### Task 4.2: Test Fixtures

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 4.2.1 | P0 | 3h | Fixture 1: TodoMVC with 4 planted bugs |
| 4.2.2 | P0 | 3h | Fixture 2: Signup flow with 3 UX issues |
| 4.2.3 | P1 | 2h | Fixture 3: E-commerce with pricing confusion |
| 4.2.4 | P1 | 2h | 验证脚本: 自动运行 + 检查发现率 |

### Task 4.3: README + Documentation

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 4.3.1 | P0 | 2h | README.md: 安装 + Quick Start + 示例输出 |
| 4.3.2 | P1 | 1h | 限制说明: 站点兼容性表 + 诚实定位 |
| 4.3.3 | P2 | 2h | Demo GIF/视频 录制 |

### Sprint 4 交付物

- SKILL.md (完整 skill 定义)
- 3 个 test fixtures
- 验证结果: ≥75% 发现率
- 升级版 README.md
- PR (不 merge)

---

## Sprint 跨度总结

```
Sprint 1 (2-3d) ──► Sprint 2 (3-4d) ──► Sprint 3 (3-4d) ──► Sprint 4 (2-3d)
   Scout              Browser Loop         Multi-Persona        Packaging
   Persona Gen        Feedback Writer      Aggregation          Validation
                                           Report               SKILL.md
```

**Total**: ~11-14 天到 v0.4 (可发布状态)

---

## Agent 分配建议

| Sprint | 推荐 Agent | 原因 |
|--------|-----------|------|
| 1 (Scout + Persona) | Codex | 明确 spec, 直接实现 |
| 2 (Browser Loop) | CC | 复杂状态机, 需要架构判断 |
| 3 (Aggregation) | Codex + CC | Codex 做 runner/report, CC 做 aggregation 逻辑 |
| 4 (Packaging) | CC | SKILL.md 需要精心设计 prompt |

**交叉 Review**: 每个 Sprint 的 PR 由另一个 agent review

---

*每个 Sprint 结束时，在真实站点上 demo。如果反馈质量不达标，停下来修而不是继续加功能。*
