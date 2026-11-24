# Week 03 课堂实验：设计人在环路的综述生成 skill（review-gen-lab）

> 本文档配合 `plan.md` 第3节使用。前置：`zotero-lab.md` 第1–4步已完成（Zotero MCP 已连库）——本实验只读 Zotero 库，不写库。

## 实验目标

基于 LiRA（AAAI 2026）参考架构，**自己设计**一个综述生成 skill：不是复刻 LiRA 的全自动管道，而是把关键阶段性决策交还给用户——**AI 执行机械活，你在闸口做判断**。课堂产出设计 + skill 骨架，完整实现课后完成，用它生成自己选题方向的综述初稿。

你扮演**设计者与裁决者**，AI 按你设计的规程扮演 Researcher / Outline Drafter / Subsection Writer / Editor / Reviewer 等角色。

## 背景概念

- **综述是结构化综合**：按主题组织而非逐篇罗列（呼应第2节 Webster & Watson 概念中心矩阵）
- **LiRA 参考架构**：Researcher 文献分析 → Outline Drafter 大纲起草 → Subsection Writer 分节写作（逐节检索，≤文献池 25%）→ Editor 润色（不改事实）→ Reviewer 四准则审查（最多 3 轮）；引用用完整标题作"语义锚点"，终稿再转编号；LangGraph 全自动编排
- **课堂版的核心偏离**：LiRA 里 Reviewer 循环、大纲定稿等判断由机器自动完成；你的 skill 要把这些裁决权**拿回来**——在关键阶段设人工闸口，AI 必停下等你决策。全自动 vs 人在环路不是对错问题，而是取舍：全自动效率高但判断不可追责，人在环路慢但每步可审计
- **检索底座**：Zotero MCP 检索工具——关键词检索 + 向量语义检索（`[semantic]`，课前已 `update-db --fulltext` 建索引）。设计时明确每个阶段用哪种检索、检索什么（文献池构建 vs 逐节检索）

## 设计任务

综述写作流程拆解为六个子任务：**检索 → 筛选 → 提取 → 组织 → 生成 → 审查**。你的设计要回答四个问题：

1. **每个子任务谁来做**：标注"AI 可自动 / 必须人裁决"。判断标准：机械转换（格式化、润色、按模板提取）可自动；涉及研究判断（信什么文献、按什么逻辑组织、引用是否成立、终稿是否合格）必须人做
2. **闸口放在哪**：最少 3 个（对照 LiRA 想想它自动化了哪些本该人做的判断）：
   - **G-纳入**：文献池清单审查——哪几篇跑题？缺哪个主题分支？（这是第1节"纳入排除标准"在生成工作流中的延续：Zotero 管"信什么"，AI 管道管"怎么写"）
   - **G-大纲**：候选大纲裁决——聚类对不对？节该合并/拆分吗？通常是最重的闸口
   - **G-终审**：Reviewer 打分后裁决——接受本轮还是再来一轮？
   - 建议再加：**G-引用**（写完一节抽 3 条引用核对正文论点真的被该文献支撑——防幻觉训练点）
3. **留痕怎么做**：每次闸口决策记到 `review/memory/decisions.md`（或你设计的等价物）——这是反思日志的现成素材，也是"可审计"的落点
4. **输出文件怎么组织**：建议 `review/` 目录分阶段产出（文献池 / 提取笔记 / 大纲 / 分节草稿 / 润色稿 / 终稿 + `PROGRESS.md` 支持续跑）

### skill 工程要点

- skill 放自己项目的 `.opencode/skills/<你的skill名>/`，含 `SKILL.md`（触发条件 + 全流程规程）与 `references/` 分阶段提示词模板——结构参考 [OpenCode Skills 文档](https://opencode.ai/docs/skills/)
- **为什么用 skill 而不是 command（提示词模板）或 agent（会话人格/后台子代理）**：skill 能带 references/ 支撑文件、在当前会话内加载规程、闸口可中途暂停等人；subagent 后台运行无法中途提问。三种机制的取舍是 agent 工程的常见判断
- 规程里明确写死闸口行为：到闸口必停、向用户呈现什么信息、等待什么决策、决策记录到哪

## 实验步骤（45 分钟，课堂只做设计 + 骨架）

### 第1步 · 工作流设计（10 分钟，对应 plan 第9步）

先自己拆：综述写作流程怎么拆成子任务？每步输入输出是什么？每个子任务标注"AI 可自动 / 必须人裁决"。再对照 LiRA 四智能体检查你的拆解——它把哪些判断自动化了？你为什么（不）拿回来？

### 第2步 · 个人文献智能检索实操（15 分钟，对应 plan 第10步）

用自己的文献库跑检索——关键词 + 语义检索各试一遍，体会两种检索的差异（精确匹配 vs 按意思找）；提取综述所需的关键信息（研究问题 | 理论框架 | 方法 | 核心发现 | 局限）。这一步同时是在为你的 skill 验证检索底座：哪些检索词有效、语义检索质量如何。💡 课堂互动：抽取 2-3 位同学投屏演示智能检索过程，全班交流检索策略与效果（3-4 分钟）

### 第3步 · skill 骨架搭建与试跑演练（15 分钟，对应 plan 第11步）

把第1步的设计落成 `.opencode/skills/` 下的骨架：`SKILL.md`（写清流程、闸口、留痕、输出文件）+ `references/` 各阶段提示词的空骨架或要点。**完整实现课后完成**。

课堂内做一次单节试跑演练：不走 skill，直接用一段 Prompt 走通"语义检索 → 提取 → 写一节"——这段实测就是你的 Subsection Writer 提示词的原型，写进 skill 前先验证它有效。💡 课堂互动：抽 2-3 位投屏讲自己的闸口设计——闸口放在哪、为什么

### 第4步 · 阶段收束（5 分钟，对应 plan 第12步）

回顾设计要点；明确课后任务：实现 skill → 跑通全流程生成综述初稿 → W4 课堂可能被抽到分享。教师参考实现将在 W4 学生分享后发布——先有自己的设计，再对照差距，学习效果远好于先看答案

### 课后 · 实现、生成与迭代

1. 补全 skill：把课堂验证过的提示词写进 `references/`，闸口行为写死在 `SKILL.md`
2. 跑通全流程，产出综述初稿（`review/` 目录 + 终稿 + `decisions.md`）
3. 第二轮起迭代：互评/导师意见/自读发现 → 对 OpenCode 说"修改综述：<反馈>"——你的 skill 如何响应修改请求（从 Reviewer 重评重入？只改相关节？）也是设计决策之一，留痕记录。真实投稿就是 review→revise 的多轮循环
4. 同伴互评：两人交换终稿，用 Snyder 2019 四准则（完整性 / 透明性 / 清晰性 / 贡献，见 LiRA 论文 Reviewer 设计）互评打分 + 一条具体改进建议——rubric 是否内置进你的 skill，同样是设计决策

## 与 LiRA 的对照表（自查用）

| LiRA 机制 | 你的设计可对照 | 教学点 |
|---|---|---|
| 文档检索器（RQ4 真实场景） | Zotero MCP 语义检索 | 论文自设场景 → 个人文献库 |
| 逐节 FAISS 检索（≤25% 池） | 每节一次 Zotero 语义检索 | 换索引引擎，机制保留 |
| Researcher 分析文献 | 提取笔记阶段 | 一致 |
| Outline Drafter 候选→合并 | 大纲阶段 + G-大纲闸口 | **LiRA 自动定稿 → 你裁决** |
| Reviewer 四准则 ≤3 轮 | 审查阶段 + G-终审闸口 | **LiRA 自动循环 → 你裁决** |
| Citation Behavior 标题锚点→编号 | 生成 + 后处理 | 一致；配 G-引用抽查 |
| Editor 不改事实 | 润色阶段 | 一致 |
| agent 记忆（Reflexion） | `review/memory/` 文件 | 形式替换 |
| LangGraph 自动编排 | 学生=编排者，skill=编排纪律 | **编排者换成学生——这正是教学点** |

设计自查：你的 skill 与 LiRA 的偏离度在哪几行？每处偏离是特性（人在环路）还是缺陷（实现偷懒）？

## 卡住了

- 语义检索提示 not available → 课前没装 `[semantic]`，退回关键词检索，课后补装并 `update-db`
- 不知道闸口怎么设 → 回到判断标准：机械转换可自动，研究判断必须人做；最少三闸口（G-纳入 / G-大纲 / G-终审）
- AI 引用了清单外的文献 → 规程里写明"只用文献池与提取笔记里的文献"，并跑 G-引用抽查
- 一次读太多全文变慢 → 摘要先行，全文只给核心 3–4 篇——"检索是核心瓶颈"的体感
- 综述像逐篇罗列 → 回到 G-大纲闸口，检查是否按主题组织
- skill 写出来了但不触发 → 检查 `SKILL.md` 的 description 是否清楚描述使用场景；也可显式 `@<skill名>` 点名

## 实验交付

- 自己设计的 review-gen skill（`.opencode/skills/` 目录：SKILL.md + references/，**结课作业组成部分**）
- `review/` 目录（终稿综述 + `PROGRESS.md` + `memory/decisions.md` 闸口判断记录）
- 阶段反思日志：`decisions.md` 里闸口判断是现成素材——围绕"AI 辅助综述流程中的人工判断点"与"闸口设计取舍（对照 LiRA 的偏离度）"展开
- 汇总提交要求见 `lab.md` 实验交付与 `plan.md` 交付物

## 参考资料

- LiRA：Go, G. H. T., Ly, K., Søgaard, A., & Tabatabaei, S. A. (2026). "LiRA: A Multi-Agent Framework for Reliable and Readable Literature Review Generation." *AAAI 2026*（课程 Zotero 库；[arXiv:2510.05138](https://arxiv.org/abs/2510.05138)；代码 [github.com/lira-workflow/auto-review-writing](https://www.github.com/lira-workflow/auto-review-writing)）
- PaperQA：Lála, J. et al. (2023). "PaperQA: Retrieval-Augmented Generative Agent for Scientific Research." [arXiv:2312.07559](https://arxiv.org/abs/2312.07559)（对照：问答式检索增强 vs 整篇综述工作流）
- Snyder, F. (2019). Literature Review as a Research Methodology. 四准则出处（见 LiRA 论文 Reviewer 设计）
- [OpenCode Skills 文档](https://opencode.ai/docs/skills/)（SKILL.md 结构与 references/ 用法）
- `knowledge/m1-7-ai-literature-review.md`（LiRA 案例）、`knowledge/m1-8-review-writing.md`（综述写作规范）
