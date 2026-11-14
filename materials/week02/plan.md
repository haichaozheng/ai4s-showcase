# Week 02 教学计划

## 本周主题

文献检索 — 检索工具与策略

> Phase 1: Foundation 第2周。文献检索是方法论（检索式、纳入排除标准、留痕可复现），不是"搜一下"；AI 辅助筛选但人来定标准。本周是 W1 AI 辅助编程的第一次真实科研应用——用 opencode 开发结构化检索脚本。

## 学习目标

- 能说明系统性文献检索的策略：检索词设计、数据库选择、纳入排除标准
- 能说明 Levy & Ellis（2006）§2 "Literature Review: Inputs" 一节的核心要求（检索式构建、数据库选择、纳入排除标准、留痕可复现）
- 能用课程 skill `search-paper-ai4s-class` 调用 arXiv / Google Scholar（SerpAPI）/ Web of Science（机构 Key）批量获取文献元数据，并说明三者的工具边界（开放 API / 无官方 API / 机构订阅）
- 能用 opencode 开发结构化检索脚本（检索式参数化、API 批量调用、CSV 存储），并与 AI 结对调试迭代
- 能用 AI 辅助初筛文献，并说明 AI 辅助筛选的边界（人来定标准）
- 能说明检索可复现意识：检索式、数据库、日期、命中数留痕，每次检索可复现

## 教学步骤（3节课）

**第1节 · 文献检索方法论（45 分钟）**

1. 💡 **必读分享：Can Generative AI improve social science?（8 分钟）**：W1 课后必读（Bail, C.A. 2024, *PNAS*，见 `week01/plan.md`"课后自学"）——抽取 2-3 位同学分享读后感想：生成式 AI 能改进社科研究的哪些环节（调研、实验、自动化内容分析、ABM），局限在哪（训练数据偏见、伦理、可复现性、"低质研究"泛滥），与"人判断、AI 执行"如何呼应；教师收拢到本周主线：文献检索正是"AI 执行、人定标准"的第一个真实科研环节
2. 文献检索方法论讲解（20 分钟，教师讲解为主，含现场演示）：以 Levy & Ellis（2006）"输入—加工—输出"系统框架中的 **§2 Literature Review: Inputs（文献获取与筛选）** 一节为纲，讲三组要点——
   - **检索式构建**（原文 "Keywords search"）：检索词作用于标题/摘要/关键词字段；新手"冷启动"难题——检索词从哪来？最佳来源是文献本身，ACM 分类体系、MIS Quarterly Roadmap 可作起点；关键词有生命周期（MRP→ERP）、技术热词（如 phishing）易失效，应上升到稳定的理论构念；关键词检索只是起点，需配合回溯检索（backward，查参考文献）与前向检索（forward，查施引文献）——此法源自 Webster & Watson（2002）
   - **数据库选择**（原文 "Where to look for quality IS literature"）：同行评审权威期刊应为主体，非评审/从业者来源仅限事实性信息；用期刊排名列表（ISWorld）反推数据库覆盖（原文 Figure 2 将排名期刊映射到 ProQuest/ScienceDirect/IEEE Xplore/ACM DL 等库）；勿把检索范围锁死在单一数据库
   - **纳入排除标准**（原文 "Testing for applicability"）：适用性检验含纳入/排除与文献伦理使用两个面向；检索命中过多时（原文以 TAM 海量文献为例）依据什么纳入、排除须事先定标准；守住质量底线（同行评审优先）
   
   以 Webster & Watson（2002）"分析过去、准备未来"点明检索的价值：检索质量决定综述上限。多出的时间做一次现场演示：从学生提的一个真实关键词出发，示范"关键词检索 → 回溯（查参考文献）→ 前向（查施引文献）"的完整链条
3. 检索工具盘点（9 分钟）：检索数据库（Google Scholar / Web of Science / Semantic Scholar / Crossref / arXiv）、第三方文献平台（AMiner，补足中文文献覆盖）、个人文献管理数据库（Zotero）；说明：课堂实操只覆盖 arXiv / Google Scholar / Web of Science 三源（整合为 skill `search-paper-ai4s-class`），不做知网/CNKI，中文文献课后自查
4. 个人研究领域检索式初稿与经验分享（8 分钟）：每位博士生先用 3 分钟草拟自己领域的检索式。💡 课堂互动（5 分钟）：抽取 2-3 位同学分享自己平时检索文献的真实经验——常用关键词及同义词怎么积累、惯用哪些数据库/平台、踩过什么坑（如只用技术热词、只查一个库），并投屏简单演示；教师把分享内容对号入座到 Inputs 三要点，顺势过渡到下一节"把检索式写成脚本"

**第2节 · AI 辅助编程深化：开发结构化检索脚本（45 分钟）**

5. opencode 开发检索脚本（20 分钟，两个递进微实验，每个实验都是"提任务 → 对话式生成 → 点运行 → 看结果"，教师边做学生边跟；学生操作文档：`opencode-search-lab.md`，含提示词模板与排错速查）：
   - **微实验 1 · 最小调用（8 分钟）**：开场点明检索脚本需要的结构——检索式参数化、API 调用、分页与限流、异常处理、CSV 存储（对比 W1 的 `mini_emotion.py` 单文件小脚本），随后让 opencode 写最小脚本：调用 arXiv 开放 API（免 key、返回 XML）检索一个关键词，解析并打印前 5 条论文标题——体验"API 调用 + 异常处理"最小闭环
   - **微实验 2 · 换库与边界（12 分钟）**：同款脚本改 API 端点与参数，切换到 Google Scholar（SerpAPI Key）与 Web of Science（机构 Key），结果写入带留痕列（query / source / retrieved_at）的 CSV，同一检索式对比三源覆盖差异（arXiv 只有预印本、GS 有被引数与灰色文献、WoS 是 SSCI/SCI 正式库）；AGENTS.md 同步更新（目录约定、原始文献库数据说明）。收尾讲清工具边界：Google Scholar **无官方 API**（爬取违反 ToS 且不稳定，讨论"能不能爬"的伦理与合规）；Web of Science 需机构订阅 Clarivate API key（引导有条件的同学课后向图书馆申请；订阅未覆盖时 403，正是"工具可得性"的活案例）——呼应 Inputs"勿锁死单一数据库"
6. 学生跟做与结对调试（15 分钟）：每人用课程 skill `search-paper-ai4s-class` 把检索式换成自己领域，用三源中可用的源跑通拿到第一批元数据（arXiv 适合 CS/物理/量化方向，社科方向多用 scholar/wos；缺 Key 时先跑 arxiv，课后申请 Key 补全）；报错按"问 AI（附报错）→ 同学互助 → 举手教师"顺序排障
7. 检索留痕意识（10 分钟）：检索式、数据库、日期、命中数——每次检索可复现，为 W3 清洗与综述写作留证据；留痕字段就写进 CSV 的列名里（query、source、retrieved_at），让"可复现"从口号变成脚本的一部分

**第3节 · Skill 构建：把检索方法沉淀成可复用工具（45 分钟）**

8. Skill 基础：为什么需要 skill、如何构建（12 分钟，教师讲解为主）：以课程 skill `search-paper-ai4s-class` 为活教材——
   - **升级链条**：一次性脚本 → 参数化脚本 → skill。脚本跑完就沉底，每次让 AI 重写都得重新交代流程；skill 把"什么时候用、怎么用、有哪些约定"写进 `SKILL.md`，opencode 在对话中自动触发、跨项目复用——把第1节 Inputs 里的个人检索经验固化成可复用资产
   - **skill 的解剖**：frontmatter 的 `name` / `description` 决定"何时被触发"（description 要写清使用场景与触发关键词）；正文写工作流程、默认命令、参数表；`scripts/` 放脚本、`references/` 放 API 文档摘录；**边界写进 skill 本身**（如"人定纳入排除标准，本 skill 只检索与去重"）——工具设计就是方法论落地，"AI 执行、人定标准"从口号变成规则
9. 从头构建多源检索 skill（18 分钟，师生共建微实验；学生操作文档：`skill-lab.md`，含提示词模板与验收检查点）：以第2节自己写的三个脚本为原料，用 skill-creator 走一遍构建流程——①定边界（做什么 / 不做什么）→ ②让 opencode 起草 `SKILL.md`，人修改 description 的触发词与场景 → ③收编脚本进 `scripts/` → ④实测：对 opencode 说"帮我检索文献"看是否自动触发，用自己构建的 skill 对同一检索式分别查 arXiv / Google Scholar / Web of Science，比较覆盖与被引数差异、标记去重字段（DOI → arXiv ID → 规范化题名）。第2节脚本没写全的同学，直接以课程 skill 为案例拆解重建
10. skill 质检与改进（7 分钟）：AI 产品也有"同行评审"——用 **skill-quality-reviewer** 给自己的 skill 体检（description 质量、结构完整性、写作风格），用 **skill-improver** 把改进建议落地；同座位互换，互相测对方 skill 的触发准确度。落选的表述、冗余的说明就在这一轮里删掉
11. AI 辅助初筛 + 下周铺垫（8 分钟）：人定纳入排除标准 → AI 按标准对元数据（标题/摘要）做初步筛选分类 → 抽查校准；边界：AI 执行筛选，人来定标准。文献库不足 ≥20 篇的课后用自己构建的 skill 补全。W3 把原始文献库导入 Zotero 清洗、用两篇方法论文献指导综述写作；预告进阶轨（手写最小 RAG，加分）

## 交付物

- **文献检索工具列表 + 原始文献元数据**（评分锚点，对应 milestones §1.2，计入阶段成果一）
  - 工具列表：所用的数据库、API、工具、检索式、纳入排除标准
  - 检索脚本：可运行的检索脚本（自写，或基于课程 skill `search-paper-ai4s-class`；检索式参数化，可复现）
  - 原始文献元数据：按检索策略获取的文献集合（CSV 格式，尚未清洗去重，含检索留痕：检索式、数据库、日期、命中数）
- 方法论锚定：体现系统性检索策略、检索方法论框架意识（Levy & Ellis）、AI 辅助筛选的边界、检索可复现意识

## 课后自学

- 文献检索策略：`knowledge/m1-6-literature-search.md`
- AI 辅助编程：`knowledge/m1-2-ai-coding.md`

## 课后作业

构建个人研究领域的检索式，用 API 批量获取至少 20 篇文献元数据（CSV）。提交检索式、纳入排除标准、文献元数据 CSV 与检索脚本代码。选做（加分）：用 skill-creator 把自己的检索流程构建成个人 skill，并用 skill-quality-reviewer / skill-improver 完成一轮质检与改进。

## 参考资料

- 理念：Levy, Y., & Ellis, T. J. (2006). "A Systems Approach to Conduct an Effective Literature Review in Support of Information Systems Research." *Informing Science*, 9, 181–212；Webster, J., & Watson, R. T. (2002). "Analyzing the Past to Prepare for the Future: Writing a Literature Review." *MIS Quarterly*, 26(2), xiii–xxiii（两篇均在课程 Zotero 库，W3 综述写作阶段继续使用）
- 工具：课程 skill `search-paper-ai4s-class`（`​.opencode/skills/search-paper-ai4s-class/`，三源检索 + 留痕 CSV）、skill-creator / skill-quality-reviewer / skill-improver（第3节 skill 构建与质检）、[arXiv API](https://info.arxiv.org/help/api/index.html)、[SerpAPI](https://serpapi.com/manage-api-key)（Google Scholar）、[Clarivate WoS API](https://developer.clarivate.com/apis/wos-starter)、[opencode 文档](https://opencode.ai/docs/)
- 文献平台与管理：[AMiner](https://www.aminer.cn/)、[Zotero](https://www.zotero.org/)
- `weeks/week02/lecture.md`、`weeks/week02/lab.md`
