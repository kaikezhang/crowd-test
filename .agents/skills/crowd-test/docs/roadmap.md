# CrowdTest — Roadmap

> Updated: 2026-03-22

---

## Version History

| Version | Milestone | Status |
|---------|-----------|--------|
| v0.1 | Single persona validation (Alex on Event Radar) | ✅ Done |
| v0.2 | Scout + Persona Generator + Single-persona loop | 🔜 Next |
| v0.3 | Multi-persona + Aggregation + Report | Planned |
| v0.4 | SKILL.md packaging + Self-validation | Planned |
| v1.0 | Public release (ClawHub + GitHub) | Planned |
| v1.1+ | Advanced features | Future |

---

## v0.1 — Proof of Concept ✅

**Goal**: Prove the pipeline works end-to-end

**Delivered**:
- Single persona "Alex" (swing trader) tested Event Radar
- Manual persona definition
- Browser automation via Playwright
- Structured feedback output
- **Validated**: LLM persona + browser automation + product feedback = viable

---

## v0.2 — Core Pipeline (Current) 🔜

**Goal**: Automated persona generation + reliable page testing

**Sprint 1 — Scout + Persona Generator** (P0):
- [ ] Scout: page readiness checker (load, auth, CAPTCHA, cookie banner)
- [ ] Scout: site_map builder (multi-page exploration)
- [ ] Persona Generator: dimension matrix → N unique personas
- [ ] Persona Generator: behavioral rules (IF-THEN, not adjectives)
- [ ] Persona Generator: cumulative diversity (each new persona fills gaps)

**Sprint 2 — Single Persona Browser Loop** (P0):
- [ ] Browser session state machine (INIT → NAVIGATE → ORIENT → ACT → OBSERVE → DECIDE → REFLECT)
- [ ] Action logging (every action + before/after snapshot)
- [ ] Circuit breakers (20 actions, 3 min, 5 consecutive failures)
- [ ] Grounded feedback writer (three-phase: log → reflect → extract)
- [ ] Feedback self-validation canary check

**Exit criteria**: One persona can Scout a URL, generate a persona, browse as that persona, and produce specific grounded feedback — all automatically.

---

## v0.3 — Multi-Persona + Aggregation

**Goal**: Run N personas and produce a useful report

- [ ] Multi-persona sequential execution (5-10 personas)
- [ ] Cumulative feedback diversity ("find different issues")
- [ ] Issue deduplication across personas (semantic merge)
- [ ] Severity scoring with consensus weighting
- [ ] Cross-dimension segment analysis (novice vs expert, mobile vs desktop)
- [ ] Markdown report generation

**Exit criteria**: `crowdtest https://example.com` runs 10 personas and outputs a prioritized issue report with cross-persona consensus.

---

## v0.4 — Packaging + Validation

**Goal**: Ship-ready skill with confidence in quality

- [ ] SKILL.md packaging (Agent Skills 开放标准 — 兼容 Claude Code / Cursor / Gemini CLI / OpenClaw 等 15+ 工具)
- [ ] Dependency check (Playwright MCP availability)
- [ ] 3-5 test fixtures with planted bugs
- [ ] Self-validation suite (≥75% known-issue discovery rate)
- [ ] Error handling polish (graceful degradation on all failure modes)
- [ ] README with installation + quick start
- [ ] `npx skills add kaikezhang/crowd-test` 安装测试

**Exit criteria**: `npx skills add kaikezhang/crowd-test` 后，用户在任意支持 Agent Skills 的工具中运行 `/crowd-test https://their-site.com`。Test fixtures pass at ≥75% discovery rate.

---

## v1.0 — Public Release 🎯

**Goal**: Ready for real users

- [ ] 发布到 skills.sh directory (Agent Skills 目录)
- [ ] ClawHub publication (OpenClaw 渠道)
- [ ] GitHub README with demo video/GIF
- [ ] HTML interactive report (standalone, no server)
- [ ] Parallel persona execution via sub-agents (3-5 concurrent)
- [ ] Configuration file support (custom dimensions, model selection, persona count)
- [ ] Cost estimation before run ("This will cost ~$X")
- [ ] Demo video for Indie Hackers / HN / Reddit

**Exit criteria**: 10+ different products tested, average ≥2 novel issues found per run, feedback specificity ≥80%.

---

## v1.1+ — Advanced Features (Future)

**When validated and adopted**:

| Feature | Description |
|---------|-------------|
| A/B Testing | `--compare url1 url2` — same personas, two versions |
| Return User Simulation | Multi-session: first visit → return visit (memory) |
| Embedding-based Diversity | Scale to 50-100 personas without context overflow |
| Custom Evaluation Metrics | User-defined scoring criteria |
| CI/CD Integration | GitHub Action / pre-commit hook |
| Accessibility Personas | Screen reader, color blind, motor impaired simulation |
| Locale/i18n Testing | Test RTL, date formats, translations |
| Visual Regression | Screenshot comparison between runs |
| API Testing Mode | Test API endpoints, not just web UI |
| Team Dashboard | Web UI for viewing historical runs |

---

## Timeline Estimate

| Milestone | Estimated Effort | Dependencies |
|-----------|-----------------|--------------|
| v0.2 Sprint 1 | 2-3 days | None |
| v0.2 Sprint 2 | 3-4 days | Sprint 1 |
| v0.3 | 3-4 days | v0.2 |
| v0.4 | 2-3 days | v0.3 |
| v1.0 | 5-7 days | v0.4 + user feedback |

**Total to v1.0**: ~3-4 weeks of focused development

---

*Ship small, validate often. Every version must prove its value before the next begins.*
