# 把检索脚本做成 skill（课堂实验文档）

> 第3节实验配套文档。目标：把第2节写的检索脚本升级成一个**可复用的 skill**——
> 学会"为什么需要 skill、skill 怎么构成、怎么从头构建、怎么质检改进"。
> 脚本开发见 `opencode-search-lab.md`；课堂大纲见 `lab.md`。

---

## 0. 为什么需要 skill：一次性脚本 → 参数化脚本 → skill

| 形态 | 跑完之后 | 下次要用时 |
|---|---|---|
| 一次性脚本 | 沉底在目录里，慢慢被忘掉 | 让 AI 重写一个（重新交代所有流程） |
| 参数化脚本 | 能复用，但**只有你知道**怎么用 | 还是你手动跑，换台电脑/换个同学就断了 |
| **skill** | `SKILL.md` 写清"何时用、怎么用、有什么约定" | 对 opencode 说一句"帮我检索文献"，**自动触发**，跨项目复用 |

skill 的价值不止省事：它把你第1节学到的方法论（检索式怎么设计、留痕留什么、
边界在哪里）**固化成资产**——你自己、你的同学、未来的 AI 会话都按同一套约定干活。

---

## 1. 解剖一个 skill（活教材：课程 skill）

先看一个真实样本。在 opencode 里输入：

```text
读 .opencode/skills/search-paper-ai4s-class/ 下的 SKILL.md，
向我解释它的结构：① frontmatter 里哪些内容负责"让 AI 什么时候想起用它"；
② 正文怎么约定工作流程和默认命令；③ 边界（做什么/不做什么）写在哪里。
```

看完你应该能指出它的三块结构（对照自己的 skill 也要有）：

```
.opencode/skills/search-paper-ai4s-class/
├── SKILL.md     ← 唯一必填：frontmatter（元信息）+ 正文（使用说明）
└── scripts/     ← 可执行脚本（统一入口 search.py + 三个源客户端）
```

- **frontmatter 的 `description` 是触发的唯一杠杆**：要写清使用场景
  （"多源检索、三源对比、原始文献库"）、触发关键词（"用 arxiv/scholar/wos
  搜论文"）、明确不做什么（"不做知网/CNKI，不导入 Zotero"）
- **正文写约定**：工作流程（先确认检索式 → 构造 → 运行 → 对比）、
  默认命令、参数表、结果怎么呈现
- **边界写进 skill 本身**："人定纳入排除标准，本 skill 只检索与去重"——
  这是方法论落地，不是可有可无的客套话

### 验收检查点

- [ ] 你能说出：为什么 `description` 决定触发，而正文不影响触发？
- [ ] 你能指出课程 skill 的哪句话体现了"AI 执行、人定标准"？

---

## 2. 实验一 · 用 skill-creator 从头构建个人检索 skill

**原料**：第2节你写的三个脚本（`my_arxiv_search.py` / `my_scholar_search.py`
/ `my_wos_search.py`）。没写全的同学直接以课程 skill 为案例拆解重建，别硬凑。

构建走四步：**① 定边界 → ② 起草 SKILL.md（人改触发词）→ ③ 收编脚本 → ④ 实测**。
让 skill-creator 带你走（在 opencode 里输入，按你的实际改）：

```text
用 skill-creator 帮我把我的检索脚本做成一个 skill，要求：

1. 原料：当前目录下的 my_arxiv_search.py / my_scholar_search.py /
   my_wos_search.py（分别调用 arXiv / Google Scholar / Web of Science，
   输出带 query/source/retrieved_at 留痕列的 CSV）
2. skill 名：my-paper-search，放在 .opencode/skills/my-paper-search/
3. 动手前先和我确认边界：
   做——三源检索、留痕 CSV、多源覆盖对比
   不做——知网/CNKI、导入 Zotero、替我决定纳入排除标准（人定标准）
4. SKILL.md 的 description 必须写清使用场景和触发关键词（如
   "检索文献、查论文、多源检索、my-paper-search"），
   草稿先给我看，我改完再继续
5. 把三个脚本收编进 scripts/，做一个统一入口 search_all.py，
   用 --sources arxiv,scholar,wos 选择数据源
```

**这一步人是主角**，机器起草你把关：

- description 里的触发词——想想你平时会怎么开口（"帮我检索文献""查一下
  XX 主题的论文"），够不够触发？太窄漏触发、太宽误触发都不行
- 边界——和你的研究习惯一致吗？中文文献怎么处理要写明白
- 默认参数——`--limit`、年份下限，写你最常用的默认值

### 验收检查点

- [ ] `.opencode/skills/my-paper-search/` 下有 `SKILL.md` 和 `scripts/`
- [ ] description 里能找到至少 3 个你会真实说出口的触发词
- [ ] 统一入口能跑通：`python .opencode/skills/my-paper-search/scripts/search_all.py --query "你的检索词" --sources arxiv`

---

## 3. 实测触发 + 多源实操

**新开一个 opencode 会话**（重要：旧会话里有上下文，测不准触发），输入：

```text
帮我检索文献：检索词用 "你的研究主题关键词"，三个源都查，每个源 20 条
```

- [ ] opencode **自动**用了你的 `my-paper-search`（而不是现写一个新脚本）——触发成功
- [ ] 触发失败？改 `description`（加场景描述和关键词）再测——description 是
      唯一杠杆，这就是第 4 步质检的重点对象

触发成功后做**多源实操**，填出三源对比表（第2节对比表的进阶版，加你自己的数据）：

| 对比项 | arXiv | Google Scholar | Web of Science |
|---|---|---|---|
| 命中数（同一检索式） |  |  |  |
| 有被引数的条数 |  |  |  |
| 有摘要的条数 |  |  |  |
| 独有文献（他源没有）举例 |  |  |  |

并标记**去重字段优先级**：DOI → arXiv ID → 规范化题名（为什么要这个顺序？
同一篇论文三个源给的元数据完整度不同，想一想）。

---

## 4. 实验二 · 质检与改进：AI 产品的"同行评审"

skill 写完不等于写好——像论文要送审一样，给 skill 做一轮体检：

```text
用 skill-quality-reviewer 给 .opencode/skills/my-paper-search 做质量体检，
把报告给我讲清楚：哪些项有问题、为什么算问题、怎么改。
```

看完报告，把改进落地：

```text
用 skill-improver 按上面的体检报告改进 .opencode/skills/my-paper-search，
改完列出每处改动对应报告里的哪条建议。
```

然后**同座位互换**，互相测对方 skill 的触发准确度：每人准备 3 组说法
（1 组标准说法、2 组口语变体，如"帮我找找 XX 方向最近的文章"），
在对方机器上新开会话测——触发了几次？

> 💡 这一轮的理念和论文发表一样：**写完 → 送审 → 修回**。
> skill-quality-reviewer 是审稿人，skill-improver 是修回执行者，
> 同座互测是预答辩。落选的表述、冗余的说明就在这一轮删掉。

### 验收检查点

- [ ] 有体检报告，且每条改进建议都有对应的落实（或说明不改的理由）
- [ ] 同座互测 3 组说法至少触发 2 次，触发失败的场景你已回改 description

---

## 5. 用自己的 skill 干活：AI 辅助初筛

skill 产出的留痕 CSV 正好用来做初筛。**先人定标准，再让 AI 执行**：

```text
读我 my-paper-search 产出的 CSV（三源合并前的原始版，不去重）。
我的纳入排除标准：
- 纳入：2020 年及以后；与"你的研究主题"直接相关；同行评审期刊或
  知名预印本
- 排除：纯新闻/博客；与本主题仅沾边的泛 AI 综述
按标准对每条标注：拟纳入 / 拟排除 / 待定，各附一句理由。
不要替我决定，我抽查后定稿。
```

AI 标完后：**抽查 10 条**，记下你不同意的条数和原因——不同意率高说明
标准写得含糊，回去把标准改具体再让 AI 重标，而不是手改结果。

- [ ] 你抽查了 10 条，同意率心中有数
- [ ] 你能回答：为什么标准要"事先"定，而不是看着结果定？（呼应
      Levy & Ellis "Testing for applicability"）

---

## 6. 实验交付（对接 W2 交付物与课后作业）

1. **个人检索 skill**：`.opencode/skills/my-paper-search/`（SKILL.md + scripts/，含一轮质检改进记录）
2. **三源对比表**（第 3 节实测数据）
3. **标注了拟纳入/拟排除/待定的文献库 CSV**；不足 ≥20 篇的课后用自己的
   skill 迭代检索式补全（这正好验证你的 skill 真能复用）

> 课后作业的选做加分项（skill 构建 + 质检改进）在课堂上已经完成大半，
> 课后把补全文献、再跑一轮 reviewer 的记录补齐即可。
