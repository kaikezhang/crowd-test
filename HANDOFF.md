# CrowdTest — Project Handoff

## 项目概述

**CrowdTest** 是一个 AI Agent Skill（开放标准），用于生成多样化仿真用户来测试 Web 产品，产出 grounded UX 反馈。

- **GitHub**: https://github.com/kaikezhang/crowd-test
- **Discord 频道**: #crowd-test (channel ID: `1485244189615722537`)
- **分发**: Agent Skills 开放标准（Claude Code / Cursor / Gemini CLI / Copilot / OpenCode / Goose / 等 15+ 工具原生支持）
- **安装**: `npx skills add kaikezhang/crowd-test`
- **也支持**: OpenClaw Skill（同一份 SKILL.md 双兼容）

## 灵感来源

- **MiroFish** — swarm intelligence with 1M agents
- **Synthetic Users** — AI persona simulation
- **gstack /qa** — browser automation testing

## Roadmap

| 版本 | 内容 | 状态 |
|------|------|------|
| v0.1 | 单 persona（Alex swing trader，已在 Event Radar 验证） | ✅ Done |
| v0.2 | Persona 生成器（维度矩阵 → N 个独特用户） | 🔜 Next |
| v0.3 | 并行测试 + 反馈汇总 | Planned |
| v0.4 | OpenClaw Skill 打包 | Planned |
| v1.0 | ClawHub 发布 | Planned |

## 竞品调研（2026-03-22）

### 核心发现：CrowdTest 的差异化

市场上的项目分三类，**没有一个做到 persona 生成 + 浏览器自动化 + 产品反馈的完整链条**：

1. **对话模拟** (ArkSim, Channel-Labs) — 只测 AI chatbot，不碰 UI
2. **Persona 生成** (PersonaHub, Stanford) — 只造人，不做测试
3. **社会仿真** (MiroFish, AgentSociety) — 太重，不针对产品测试

**CrowdTest = persona 生成 + 浏览器自动化 + 结构化产品反馈** ← 空白地带 🎯

### Tier 1 — 最相关

#### ArkSim ⭐⭐⭐
- GitHub: https://github.com/ArkLexAI/arksim
- 用 LLM 模拟多轮对话测试 AI agent，自动评估性能
- 7 个内置评估指标、自定义 metrics、错误分类、并行执行、HTML 报告
- Python，支持 OpenAI/Anthropic/Google
- **和我们的区别**: 测 AI agent 对话质量，我们测 Web 产品 UI
- **可借鉴**: persona 定义格式、评估框架、报告生成

#### Channel-Labs/synthetic-conversation-generation ⭐⭐⭐
- GitHub: https://github.com/Channel-Labs/synthetic-conversation-generation
- 生成 synthetic user 与 AI 的多轮对话
- **核心创新**: persona 生成与对话生成解耦；自然停止点建模
- **可借鉴**: persona 去重/多样性保证（每个新 persona 要跟已有的不同）

#### Blok ⭐⭐⭐ (商业，$7.5M 融资)
- 网站: https://www.joinblok.co
- AI personas 模拟真实用户行为测试 app
- **启示**: 方向有钱投，市场认可

### Tier 2 — Persona 生成 / 用户模拟

#### Tencent PersonaHub ⭐⭐
- GitHub: https://github.com/tencent-ailab/persona-hub
- **10 亿个 persona** 从 web 数据自动生成
- HuggingFace 有 3.7 亿"精英 persona"可直接用
- **可借鉴**: persona 生成方法论，维度设计，大规模去重

#### Stanford "Generative Agent Simulations of 1,000 People" ⭐⭐
- 论文: https://arxiv.org/abs/2411.10109
- LLM 模拟 1,052 个真实个体，准确率 85%
- **验证了 LLM persona 模拟的可行性**

#### vincentkoc/synthetic-user-research ⭐⭐
- GitHub: https://github.com/vincentkoc/synthetic-user-research
- Jupyter Notebook，AutoGen/BabyAGI/CrewAI 做 synthetic user research
- **可借鉴**: persona prompting 模板、Summary Agent、报告格式

### Tier 3 — 大规模社会模拟

#### MiroFish ⭐
- GitHub: https://github.com/666ghj/MiroFish
- 群体智能引擎，上千 AI agent 有独立人格/记忆/社交关系
- Node.js + Python，Zep 做 agent 记忆，GraphRAG
- **可借鉴**: agent 记忆机制、大规模并行仿真架构

#### AgentSociety (清华) ⭐
- GitHub: https://github.com/tsinghua-fib-lab/AgentSociety
- 大规模城市社会模拟，Mind-Behavior Coupling
- **可借鉴**: agent 心理模型设计、交互式可视化

### 商业闭源竞品

| 产品 | 核心卖点 |
|------|---------|
| SyntheticUsers.com | AI 合成访谈，persona 调研 |
| Uxia.app | synthetic user 工具测评集合 |
| LangWatch | AI agent 测试+评估平台 |
| Playwright Test Agents | 微软官方 AI 测试 agent（免费） |

## 可复用的组件

- **PersonaHub** 的 persona 生成方法论
- **ArkSim** 的评估框架和报告格式
- **Channel-Labs** 的 persona 多样性保证
- **Playwright** 的浏览器自动化（已有 AI test agents）
- **MiroFish** 的 agent 记忆机制（"回头用户"模拟）

## 技术栈

- **Persona 生成**: LLM（维度矩阵驱动 + 行为规则，非性格描述）
- **浏览器自动化**: Playwright MCP（accessibility snapshots，非截图）
- **反馈收集**: 三阶段 grounded feedback（action log → 回顾 → 结构化提取）
- **报告**: Markdown (v1) → HTML 交互式 (v2)
- **打包**: Agent Skills 标准 (SKILL.md) → `npx skills add` 安装

## 核心设计决策（CEO + Eng Review 2026-03-22）

详见 `docs/ceo-review.md` 和 `docs/eng-review.md`。

### 定位
- **不是替代真人测试**，是"当你没人、没钱、没时间时唯一的 UX sanity check"
- 产出是"AI 生成的 UX 假设"，不是"合成用户研究"
- 60-70% 真人反馈质量，1% 成本，100x 速度

### 核心客户
- **Primary**: Solo founder (pre-PMF)，晚上 11 点需要 fresh eyes 但无人可问
- **Secondary**: 2-5 人小团队，发版前 sanity check

### 最大风险
1. 反馈是 generic slop（三阶段 grounded feedback 解决）
2. LLM 太聪明无法模拟笨用户（行为规则 > 性格描述）
3. 浏览器在 SPA 上不稳定（Scout 预检 + 优雅降级）
4. Blok ($7.5M) 做同样的事（差异化：开源 + CLI + 本地运行）

### 成本
- 10 persona × Sonnet = ~$1/次
- 比 UserTesting.com ($49/人) 便宜 25 倍

## v0.1 已验证

单 persona "Alex"（swing trader）在 Event Radar 上做了完整的用户测试流程：
- 用浏览器打开产品
- 按 persona 特征操作 UI
- 给出结构化反馈

证明了 **LLM persona + 浏览器自动化 + 产品反馈** 这条链路是通的。

## 项目文档

| 文件 | 内容 |
|------|------|
| `SKILL.md` | Skill 定义（完整 orchestration prompt） |
| `CLAUDE.md` | CC 开发指南 |
| `AGENTS.md` | Codex 开发指南 |
| `TASK.md` | 当前 Sprint 任务 |
| `docs/PRD.md` | 产品需求文档 |
| `docs/technical-design.md` | 架构、数据结构、Prompt 设计 |
| `docs/competitive-research.md` | 竞品深度技术调研 |
| `docs/ceo-review.md` | CEO/产品视角 Review |
| `docs/eng-review.md` | VP Eng 技术可行性 Review |
| `docs/roadmap.md` | 版本里程碑 |
| `docs/sprint-plan.md` | Sprint 级任务分解 |

## 下一步 (v0.2)

Scout + Persona Generator + 单 persona 浏览 loop：
- **Scout 预检**: 检测 auth/CAPTCHA/cookie banner，建 site_map（**table stakes，不做这个 30% 跑出垃圾**）
- **Persona 生成器**: 维度矩阵 + **行为规则** (IF-THEN, 非性格描述)
- **Browser session**: ORIENT → ACT → OBSERVE → DECIDE 状态机，action log
- **Grounded feedback**: 三阶段（action log → 回顾 → 结构化提取 with evidence）

详细任务分解见 `docs/sprint-plan.md`
