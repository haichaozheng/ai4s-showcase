# Week 03 教学计划

## 本周主题

文献综述 — 文献管理 + AI 综述

> Phase 1: Foundation 第3周，文献综述（计入阶段成果一）。文献库整理是检索到综述的桥梁；第三方智能检索工具改变文献阅读范式——从"读完记住"到"检索即用"；综述是结构化综合而非摘抄堆砌。

## 学习目标

- 能用 Zotero 管理文献库：导入、去重、标签、规范化、纳入排除标准筛选
- 能在课堂跟做中跑通最小 RAG（切分→嵌入→存库→检索，`rag_demo.py`），理解"检索"这一步为什么是核心瓶颈；进阶扩展（加分）：Streamlit 检索测试台、换中文嵌入模型、接入 LLM 生成
- 能用 Zotero MCP 进行文献智能检索与问答（关键词 + 语义检索），了解 AMiner 等第三方智能检索工具的定位（提及，不作为本周核心）
- 能用 Prompt 驱动 LLM 生成有逻辑框架的文献综述
- 能说明综述写作规范：综述有逻辑框架（按主题/方法论/时间线组织），而非逐篇罗列
- 能设计并实现**人在环路**的 AI 辅助综述生成 skill：拆解综述写作流程（检索→筛选→提取→组织→生成→审查），对照 LiRA 多智能体架构定位**哪些决策必须人做**（纳入排除、大纲、引用核查、终审），把人工闸口嵌入 skill 而非全自动；检索层充分利用 Zotero MCP 检索工具（关键词 + 向量语义检索）；能说明全自动（LiRA）与人机协作（人工闸口）两种模式的取舍

## 教学步骤（3节课）

> **课前准备**（三节各自对应一份 lab 文档，课前只做安装配置，课上只做核心体验）：
>
> - **第1节（`zotero-lab.md`）**：按第1–3步完成 Zotero MCP 安装与 OpenCode 配置，按 6.1 节完成写入授权（`zotero-mcp authorize-local`，弹窗选 Always Allow——课上要批量打标签）；确认 W2 文献已导入（≥20 篇）。验收：`/mcps` 里 zotero 显示 connected
> - **第2节（`RAG-lab.md`）**：按第1步装好 `chromadb` + `sentence-transformers`，按"课前分发"把教师发的模型文件夹解压到位——课上不装依赖、不下载模型。验收：`import chromadb` 不报错
> - **第3节（`review-gen-lab.md`）**：课前跑 `zotero-mcp update-db --fulltext` 建好语义索引（课上现跑会超时）；课前浏览课程 Zotero 库中 LiRA 论文（AAAI 2026）的工作流设计部分，思考"哪些环节你不放心全交给 AI"
> - 配置指南由教师提前一周发布

**第1节 · 文献库整理与 Zotero MCP 直连（45 分钟，完成 `zotero-lab.md` 核心部分）**

1. 文献库整理与去重（12 分钟）：AI 查重（只读，`zotero-lab.md` 第6步——无专用查重工具，AI 拉全库按标题/DOI 分组输出疑似重复）→ 回 Zotero 界面合并（Duplicate Items 视图，人工核对哪份带 PDF 和标注）→ 批量打标签（先 1 篇试水，再 `zotero_batch_update` 批量）——目标不是"管理文献"，而是构造后续生成工作流的参考文献池
2. 纳入排除标准确认（8 分钟）：用 `zotero_advanced_search` 按 `dateAdded` 圈出 W2 导入批次，再按时间范围、方法论类型筛选——点明分工：Zotero 管"信什么"（人工守住的 screening 环节），AI 工作流管"怎么写"
3. MCP 直连：找→读→标注（18 分钟）：验证连接后按 `zotero-lab.md` 第5步走完整流水线——关键词检索（子串匹配三规则：多词收窄、默认只搜元数据、没命中自动降级）→ 读元数据与摘要 → 全文与续读 → 高亮标注提取；期间提及 AMiner 等第三方智能检索工具（学者图谱、对话式检索）作为对照路线，不展开演示；末尾 2 分钟语义检索初尝（`zotero_semantic_search`，为第2节 RAG 埋点）。💡 课堂互动：请 1 位同学投屏用自然语言操作自己的文献库

**第2节 · 综述写作规范与最小 RAG 实操（45 分钟，跟做 `RAG-lab.md` 跑出 `rag_demo.py`）**

5. 综述写作规范（10 分钟）：逻辑框架（主题/方法论/时间线），而非逐篇罗列；好综述与差综述对比；以 Webster & Watson（2002）"概念中心矩阵"说明如何按概念而非按篇组织文献——W2 检索阶段的方法论阅读（Levy & Ellis 2006、Webster & Watson 2002）在综述写作阶段的延续；这也是第3节大纲闸口的方法论前置
6. RAG 原理：从"读完记住"到"检索即用"（8 分钟）：概念讲解（检索→增强→生成）；**衔接第1节末尾体验的 `zotero_semantic_search`——它就是 RAG 检索环节的自动化封装**；以 LiRA（AAAI 2026）为实例：其 Subsection Writer 用 FAISS 按小节检索文献、引用完整标题作"语义锚点"抑制幻觉
7. 最小 RAG 跟做（22 分钟）：按 `RAG-lab.md` 第3–5步全员跑通 `rag_demo.py`——嵌入存库 → 交互式语义检索（默认问题"数字技术加剧了什么样的不平等？"，体会字面无关仍可命中）→ 换自己的问题连续检索 → 字面包含对比（同场看清语义检索与关键词检索的差异）。跑通即完成"检索是核心瓶颈"的体感；课前已装依赖、模型已分发，课上零下载。💡 课堂互动：抽 1-2 位投屏演示自己的检索问题与结果对比
8. 三条检索路线对比与小结（5 分钟）：谁在检索、检索什么——第1节的 MCP 直连（实时检索原库）、刚跑的手写 RAG（预先嵌入再语义检索）、LiRA 的 FAISS 小节级检索（生成时按节检索）；从理解 RAG 到 AI 辅助综述生成的过渡

**第3节 · 设计人在环路的 AI 辅助综述生成 skill（45 分钟）**

9. 综述生成工作流设计（10 分钟）：承接第2节的综述写作规范（概念中心组织）与三条检索路线——分析综述写作流程，拆解为子任务（检索→筛选→提取→组织→生成→审查），明确每步的输入输出与知识库配置——先让学生自己拆，并为每个子任务标注"AI 可自动 / 必须人裁决"，再对照 LiRA 的四智能体参考架构（Outline Drafter 大纲起草 → Subsection Writer 分节写作 → Editor 润色 → Reviewer 质量审查）：LiRA 把哪些判断自动化了（Reviewer 循环自动跑、引用自动核查）？课堂版为什么要把这些裁决权拿回来？
10. 个人文献智能检索实操（15 分钟）：每人用自己的文献库，用 Zotero MCP 直连检索（第1节已配好；关键词检索零门槛，装了 `[semantic]` 的可用语义检索），提取综述所需的关键信息（完整步骤见 `review-gen-lab.md` 设计参考）。💡 课堂互动：抽取 2-3 位同学投屏演示智能检索过程，全班交流检索策略与效果（3-4 分钟）
11. skill 骨架设计与搭建（15 分钟）：把第9步的工作流设计落成 `.opencode/skills/` 下的 skill 骨架——阶段划分、人工闸口位置、各阶段规程要点与输出文件约定；检索层约定调用 Zotero MCP（关键词 + 语义检索）。**课堂只做设计 + 骨架，实现全部课后完成**：设计要点清单见 `review-gen-lab.md`（闸口设计、防幻觉引用核查、留痕），课堂不给参考实现。课堂内对自己的设计做一次单节试跑演练（用一段 Prompt 走通"检索→提取→写一节"）。💡 课堂互动：抽 2-3 位投屏讲自己的闸口设计——闸口放在哪、为什么
12. 阶段收束（5 分钟）：Phase 1 完成，回顾文献智能模块；布置课后练习（完成 skill 实现并用它生成个人选题方向文献综述——skill 是结课作业的组成部分，此处点到为止——不提交，W4 课堂随机抽取 2-3 位同学分享自设计的 skill 与综述成果）；教师参考实现将在 W4 学生分享后发布供对照；预告 Phase 2 选题

## 交付物

- **文献综述**（评分锚点，对应 milestones §1.3，计入阶段成果一（研究选题与研究设计）25%）
  - Zotero 文献库（去重、标签、规范化，至少 20 篇）
  - **个人 review-gen skill**（`.opencode/skills/` 下的设计 + 实现：SKILL.md、阶段规程、人工闸口；**结课作业组成部分**，W4 课堂分享，教师参考实现于学生分享后发布）
  - 用自设计 skill 生成的综述文稿（含逻辑框架与闸口判断记录）
  - 个人文献库 MCP 接入与检索配置（`opencode.jsonc` 配置 + 自然语言检索交互截图，见 `zotero-lab.md` 实验交付）
  - **`rag_demo.py` 与运行截图**（课堂跟做产出，按 `weeks/week03/RAG-lab.md`：能跑通语义检索 + 字面包含对比）
  - 进阶（加分）：RAG 扩展（按 `weeks/week03/RAG-lab.md` 进阶路线任选：Streamlit 检索测试台 `rag-app/`、换中文嵌入模型 `BAAI/bge-small-zh-v1.5` 对比、接入 LLM 完成"G"环节）
  - 阶段反思日志（阶段成果一组成部分，模板见 `templates/reflection-log.md`；素材建议：闸口设计取舍——闸口放在哪、为什么、对照 LiRA 的偏离度；可结合 LiRA（AAAI 2026）"用多智能体模拟人类综述写作过程"的思路，讨论把综述写作流程拆解为可执行工作流的方法论意涵）
- 方法论锚定：体现文献库整理逻辑、RAG 概念理解、智能检索工具使用、综述写作规范、人在环路的 skill 设计（闸口取舍）、MCP 直连检索与工作流配置思路

## 课后自学

- RAG 原理与实践：`knowledge/m1-5-rag.md`
- 手写最小 RAG（课堂已跑通主线）：`weeks/week03/RAG-lab.md`——课后从"进阶"与"观察与记录"（第6步三问）继续；配套 `zotero-lab.md` 走 Zotero MCP 直连路线（第1节课堂已覆盖第4–6步核心，课后从第5步的深入阅读与第8步语义检索继续，两条路线对比见 zotero-lab.md 第8.8节；LiRA 的 FAISS 小节级检索可作为第三条对照路线，见 `knowledge/m1-7-ai-literature-review.md`）
- AI 辅助文献综述：`knowledge/m1-7-ai-literature-review.md`（含 LiRA 多智能体综述生成案例）
- **论文精读（推荐）**：Go et al. (2026). "LiRA: A Multi-Agent Framework for Reliable and Readable Literature Review Generation." *AAAI 2026*——课程 Zotero 库有全文；扩展版 [arXiv:2510.05138](https://arxiv.org/abs/2510.05138)
- 综述写作规范：`knowledge/m1-8-review-writing.md`
- 知识图谱与文献计量（推荐）：`knowledge/m1-9-knowledge-graph-bibliometrics.md`

## 课后作业

整理 Zotero 文献库（去重、标签、规范化，至少 20 篇）；完成个人 review-gen skill 的实现（课堂骨架 → 完整规程与闸口），并用它进行智能检索、生成 AI 辅助综述初稿；提交 Zotero 文献库导出文件、skill 目录（`.opencode/skills/`）、综述文稿、`rag_demo.py` 与运行截图、MCP 配置与检索交互截图和阶段反思日志。（推荐）课后完成 `zotero-lab.md` 第5步起的深入实验与 `RAG-lab.md` 第6步观察记录。（加分）按 `RAG-lab.md` 进阶路线任选一项扩展（Streamlit 检索测试台 / 换中文嵌入模型 / 接入 LLM）并提交。

## 课后练习（不需要提交）

**个人选题方向的文献综述**：围绕自己的研究方向，用自己实现的 review-gen skill 跑通完整工作流（检索→筛选→提取→组织→生成→审查），生成并迭代打磨一篇综述初稿——第二轮起对 OpenCode 说"修改综述：<反馈>"，让 skill 在既有产出基础上重评重入（具体行为由你的 skill 设计决定，留痕见 `review-gen-lab.md` 设计参考）。

- 不需要提交，但 **W4 课堂将随机抽取 2-3 位同学分享自设计的 skill 与综述成果**——这份综述同时是 W4 gap 分析的语料输入，认真做直接服务于下周的选题；教师参考实现将在 W4 学生分享后发布，供对照自己的设计
- 鼓励将打磨后的优秀综述投稿：期刊/会议的综述栏目，或预印本平台（arXiv、SSRN 等）——AI 辅助不改变发表规范，署名与责任仍在作者

## 参考资料

- 理念：Lewis, P. et al. (2020). "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks." *NeurIPS 2020*
- 综述写作方法论（W2 贯通至 W3）：Webster, J., & Watson, R. T. (2002). "Analyzing the Past to Prepare for the Future: Writing a Literature Review." *MIS Quarterly*, 26(2), xiii–xxiii；Levy, Y., & Ellis, T. J. (2006). "A Systems Approach to Conduct an Effective Literature Review in Support of Information Systems Research." *Informing Science*, 9, 181–212（均在课程 Zotero 库）
- 多智能体综述生成：Go, G. H. T., Ly, K., Søgaard, A., & Tabatabaei, S. A. (2026). "LiRA: A Multi-Agent Framework for Reliable and Readable Literature Review Generation." *AAAI 2026*（课程 Zotero 库；扩展版 [arXiv:2510.05138](https://arxiv.org/abs/2510.05138)；代码 [github.com/lira-workflow/auto-review-writing](https://www.github.com/lira-workflow/auto-review-writing)）
- RAG 科研智能体：Lála, J., O'Donoghue, O., Shtedritski, A., Cox, S., Rodriques, S. G., & White, A. D. (2023). "PaperQA: Retrieval-Augmented Generative Agent for Scientific Research." [arXiv:2312.07559](https://arxiv.org/abs/2312.07559)（课程 Zotero 库）——RAG 智能体做文献问答的早期代表作，与 LiRA 对照：PaperQA 侧重单篇/问答式检索增强，LiRA 侧重整篇综述的生成工作流
- 工具：[Zotero](https://www.zotero.org/)、[AMiner](https://www.aminer.cn/)
- `weeks/week03/lecture.md`、`weeks/week03/lab.md`
- 实验材料：`weeks/week03/review-gen-lab.md`（第3节实验文档：skill 设计指引与设计要点清单）
- 教师参考实现：学生 W4 课堂分享后发布，供对照自己的 skill 设计
