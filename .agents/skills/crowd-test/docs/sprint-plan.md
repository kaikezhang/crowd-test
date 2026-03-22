# CrowdTest — Sprint Plan (v2 Aligned)

> Updated: 2026-03-22
> Scope: Smart v1 (v2 Phase 0-8 pipeline, SKILL.md-only product)
> Canonical vision: `docs/product-design-v2.md`

**核心原则**: CrowdTest 是一个 **SKILL.md prompt**，不是应用代码。所有开发都是写/优化 SKILL.md 的 orchestration prompt。

---

## Sprint 1: Scout + Product Analysis (3 天)

**Goal**: LLM 能自动理解一个网站是什么、做什么、给谁用

### Task 1.1: Scout (Phase 1)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 1.1.1 | P0 | 2h | SKILL.md Phase 1 prompt: 导航 URL + wait + snapshot |
| 1.1.2 | P0 | 1h | 健康检查 prompt: 计数 interactive elements, < 5 → retry |
| 1.1.3 | P0 | 1h | Auth/CAPTCHA/Cookie 检测 prompt + 处理指令 |
| 1.1.4 | P0 | 2h | 多页探索 prompt: top-level nav → 快速 snapshot 3-5 页 |
| 1.1.5 | P1 | 1h | Critical path 识别: 找到核心功能的页面路径 |
| 1.1.6 | P0 | 1h | SiteMap JSON schema + 输出示例 |

### Task 1.2: Product Analysis (Phase 0)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 1.2.1 | P0 | 2h | SKILL.md Phase 0 prompt: 从 snapshot 分析产品 |
| 1.2.2 | P0 | 1h | ProductProfile JSON schema (category, core_tasks, competitors, audience) |
| 1.2.3 | P0 | 1h | User context 合并 (--context, --focus, --competitors flags) |
| 1.2.4 | P1 | 1h | 边界情况: 空白站、纯内容站、需要登录的产品 |

**Acceptance**:
- SKILL.md Phase 0+1 完整可跟随
- 在 cal.com / linear.app / Event Radar 上人工验证 ProductProfile 质量
- SiteMap 含 ≥3 pages + critical_path

### Sprint 1 交付物
- SKILL.md 更新: Phase 0 + Phase 1 完整
- Phase 2-8 保留为 stub
- PR (不 merge)

---

## Sprint 2: Persona Gen + Task Assignment + Browser Loop (4 天)

**Goal**: 一个 persona 能带着具体任务浏览产品并产出 grounded 反馈

### Task 2.1: Persona Generation (Phase 2)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.1.1 | P0 | 3h | SKILL.md Phase 2 prompt: 从 ProductProfile 推导 5 种 archetype |
| 2.1.2 | P0 | 2h | 每个 archetype 生成 2 个 persona (共 10) |
| 2.1.3 | P0 | 2h | 行为规则生成: IF-THEN 格式, ≥3 条/persona |
| 2.1.4 | P0 | 1h | 累积多样性: 已有 persona 摘要放入 prompt |
| 2.1.5 | P1 | 1h | Entry point 多样化: 不是所有人都从首页进入 |

### Task 2.2: Task Assignment (Phase 3)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.2.1 | P0 | 2h | SKILL.md Phase 3 prompt: 基于 persona + ProductProfile 分配具体任务 |
| 2.2.2 | P0 | 1h | 成功标准定义: 每个任务的 COMPLETED/FAILED 判断条件 |

### Task 2.3: Browser Session (Phase 4)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.3.1 | P0 | 3h | SKILL.md Phase 4 prompt: ORIENT → ACT → OBSERVE → DECIDE loop |
| 2.3.2 | P0 | 2h | Action prompt: persona narrative + snapshot + task progress + emotional state |
| 2.3.3 | P0 | 1h | Circuit breakers: 20 actions, 3 min, 5 consecutive failures |
| 2.3.4 | P1 | 1h | 情绪追踪: 每步一个 emotional_state (confident/neutral/confused/frustrated) |

### Task 2.4: Feedback Synthesis (Phase 6)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 2.4.1 | P0 | 2h | SKILL.md Phase 6 prompt: 回顾 action log, 写 grounded 反馈 |
| 2.4.2 | P0 | 1h | Task completion 记录: COMPLETED/FAILED + steps + time |
| 2.4.3 | P0 | 1h | Evidence linking: 每个 issue 必须引用 action log step |
| 2.4.4 | P1 | 1h | Canary check: "Is this specific or generic?" |

**Acceptance**:
- 单 persona 在 Event Radar 上完整跑通
- 产出 grounded feedback with task completion status
- 每个 issue 有 evidence 指向具体 action step

### Sprint 2 交付物
- SKILL.md 更新: Phase 2 + 3 + 4 + 6 完整
- End-to-end 单 persona demo 可运行
- PR (不 merge)

---

## Sprint 3: Multi-Persona + Aggregation + Score + Report (3 天)

**Goal**: 10 personas 串行运行 → Product Score + 去重报告 + "Fix ONE Thing"

### Task 3.1: Multi-Persona Runner

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.1.1 | P0 | 2h | SKILL.md 串行执行 prompt: for each persona, run Phase 4→6 |
| 3.1.2 | P0 | 1h | 累积多样性: "already covered issues" 传递给下一个 persona |
| 3.1.3 | P1 | 1h | 进度输出: persona N/total + status emoji |
| 3.1.4 | P1 | 1h | 错误隔离: 单个 persona 失败不影响其他 |

### Task 3.2: Journey Reconstruction (Phase 5 — 简化版)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.2.1 | P1 | 2h | SKILL.md Phase 5 prompt: 从 action log + emotional states → 关键时刻提取 |
| 3.2.2 | P1 | 1h | 信心曲线: emotional_state → 数值映射 |

### Task 3.3: Aggregation + Scoring (Phase 7)

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.3.1 | P0 | 2h | Issue 去重: LLM 语义合并 |
| 3.3.2 | P0 | 2h | **Product Score**: 6 维度加权计算 |
| 3.3.3 | P0 | 1h | **"Fix ONE Thing"**: 最高影响单一推荐 |
| 3.3.4 | P0 | 1h | **Task Completion Rate**: X/N personas 完成核心任务 |
| 3.3.5 | P1 | 1h | Segment analysis: by tech_level / device / archetype |

### Task 3.4: Report Generation

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 3.4.1 | P0 | 3h | Markdown report: Scorecard + Fix One Thing + Issues + Journeys |
| 3.4.2 | P1 | 1h | 终端 summary: score + task completion + top 3 issues |

**Acceptance**:
- 10 personas 在真实站点上完整运行
- 产出 Product Score (X/10)
- 报告包含 "Fix ONE Thing" 推荐
- Task Completion Rate 显示在报告头部

### Sprint 3 交付物
- SKILL.md 完整 8 阶段
- End-to-end: 10 personas → 完整报告
- PR (不 merge)

---

## Sprint 4: Polish + Validation + Publish (2 天)

**Goal**: 可发布状态

### Task 4.1: 质量验证

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 4.1.1 | P0 | 3h | 在 3 个不同站点跑完整测试 (SaaS + e-commerce + content) |
| 4.1.2 | P0 | 2h | 反馈质量评估: 特异性 ≥80%, 已知问题发现率 ≥75% |
| 4.1.3 | P1 | 1h | 成本验证: 确认 10 personas < $1.50 |

### Task 4.2: 文档 + 打包

| Task | Priority | Est. | Description |
|------|----------|------|-------------|
| 4.2.1 | P0 | 2h | README.md 最终版: 含真实示例输出 |
| 4.2.2 | P0 | 1h | SKILL.md 最终 polish |
| 4.2.3 | P1 | 1h | ClawHub 发布准备 |

### Sprint 4 交付物
- 验证通过的 SKILL.md
- 3 个真实站点的测试报告
- README.md 含示例
- ClawHub 发布就绪

---

## 总览

```
Sprint 1 (3d) ──► Sprint 2 (4d) ──► Sprint 3 (3d) ──► Sprint 4 (2d)
   Scout              Persona Gen        Multi-Persona       Validation
   Product Analysis   Task Assignment    Aggregation         Polish
                      Browser Loop       Product Score       Publish
                      Feedback           Report
```

**Total**: ~12 天到可发布

## Agent 分配

| Sprint | 推荐 Agent | 原因 |
|--------|-----------|------|
| 1 (Scout + Analysis) | **CC** | Prompt 设计需要精心措辞 |
| 2 (Persona + Browser) | **CC** | 复杂 prompt 链，需要架构判断 |
| 3 (Aggregation + Score) | **CC** | 评分体系设计 |
| 4 (Validation) | **晚晚手动** | 需要在真实站点上 human-in-the-loop 验证 |

> 这个项目是 prompt engineering，不是 coding。CC 比 Codex 更适合。

---

*每个 Sprint 在真实站点上 demo。反馈质量不达标就停下来修 prompt，不继续加功能。*
