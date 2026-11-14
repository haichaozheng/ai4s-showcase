# 用 opencode 开发文献检索脚本（课堂实验文档）

> 第2节实验配套文档。目标：**学会"用 opencode + 官方 API 文档"写出一个能跑的检索脚本**，
> 而不是背代码。脚本只是载体，方法是可以迁移到任何 API 的。
> 环境没配好先看 `venv-setup.md`；课堂大纲见 `lab.md`。

---

## 0. 为什么这样学：AI 执行、人定标准

写检索脚本时，分工是这样的：

| 谁 | 做什么 |
|---|---|
| **你（人）** | 提需求、找文档、判断结果对不对、决定检索式和筛选标准 |
| **opencode（AI）** | 读文档后写代码、解释报错、改脚本 |

**AI 不会替你读文档**——你把文档要点喂给它，它才能写出对的代码。
所以本实验反复练同一个循环：

```
① 查官方文档（endpoint、参数、认证、限流）
② 把需求 + 文档要点写成 prompt 交给 opencode
③ 运行脚本，看结果是否符合预期
④ 不对 → 把报错原样贴回给 opencode → 改 → 再跑
```

> 这个循环和课程 skill `search-paper-ai4s-class` 的实现过程一模一样——
> 那套脚本也是这样写的。写完自己的脚本后，可以对照 `.opencode/skills/search-paper-ai4s-class/scripts/` 里的参考实现。

> 💡 **把官方文档直接写进提示词**：opencode 能自己抓取网页——在 prompt 里给出官方文档 URL、指明重点章节，让它**读了文档再写代码**。本实验的所有提示词模板都这样做，养成习惯：**凡是调 API，先把官方文档链接（和要点摘录）喂给 AI**。模型记忆可能过时或有错（参数名记错、限流规则过时），官方文档才是权威依据——这也是"人提供可靠信源、AI 执行"在写代码环节的体现。

---

## 1. 准备（只做一次）

1. 项目根目录打开终端，确认 Python 可用：`py -3.13 --version`（环境步骤见 `venv-setup.md`）
2. 项目根目录运行 `opencode` 进入对话界面
3. 凭据（没有也能做实验 1，见第 4 节的"降级"说明）：

   仓库根目录 `.env` 里配好 `Serp_API_KEY`（Google Scholar）和 `WOS_API_KEY`（Web of Science），申请入口见第 4 节表格

---

## 2. 三源总览：先看清楚要打交道的三个 API

| | arXiv | Google Scholar（经 SerpAPI） | Web of Science（Starter API） |
|---|---|---|---|
| 官方文档 | https://info.arxiv.org/help/api/user-manual.html | https://serpapi.com/google-scholar-api | https://developer.clarivate.com/apis/wos-starter |
| 认证 | 无需 | `api_key` 参数 | `X-ApiKey` 请求头 |
| 每页上限 | `max_results` 建议 ≤100 | `num` ≤20 | `limit` ≤50 |
| 限流（免费） | 1 次 / 3 秒 | 约 100 次/月 | 1 次/秒、50 次/天 |
| 返回格式 | Atom XML | JSON | JSON |
| 覆盖特点 | 预印本（CS/物理/量化） | 综合最广、有被引数、含灰色文献 | SSCI/SCI 正式刊、有被引数 |
| 拿到的字段 | 标题/作者/摘要/DOI | 标题/被引/出版信息（**无摘要**） | 标题/作者/年份/期刊/DOI/被引 |

> 三个 API 三种认证方式、两种返回格式——这正是真实世界的样子。
> **读文档时永远先找四样东西**：接口地址（endpoint）、必填参数、认证方式、限流规则。

---

## 3. 微实验 1 · arXiv 最小调用（先跑通一个闭环）

**任务**：用 opencode 写一个最小脚本，检索 arXiv 上含关键词的论文，打印前 5 条标题。

### 第 1 步：先自己看一眼文档

打开 https://info.arxiv.org/help/api/user-manual.html （arXiv API User's Manual），扫过这些要点（不用全懂）：

- 接口地址：`http://export.arxiv.org/api/query`
- 核心参数：`search_query`（检索式）、`start`（起始条数）、`max_results`（条数）
- 检索式语法：字段前缀 `ti:`（标题）`au:`（作者）`abs:`（摘要）`all:`（全部），
可用 `AND` / `OR` / `ANDNOT` 连接
- 返回是 **Atom XML**，每篇论文在 `<entry>` 节点里

### 第 2 步：给 opencode 下任务

在 opencode 对话框里输入（照抄即可，换掉检索词）：

```text
帮我写一个 Python 脚本 my_arxiv_search.py，要求：

1. 先读取官方文档 https://info.arxiv.org/help/api/user-manual.html
   （重点：第 3.1 节 Query Interface 的参数表、附录 5.1 Query
   Construction 的字段前缀与布尔算子），读完再写代码
2. 调用 arXiv API（http://export.arxiv.org/api/query）检索论文，
   文档要点：参数 search_query 是检索式（字段前缀 ti:/au:/abs:/all:，
   可用 AND/OR/ANDNOT），start 是起始条数，max_results 是返回条数；
   返回 Atom XML，每篇论文在 <entry> 节点里，命名空间是
   http://www.w3.org/2005/Atom
3. 检索式从命令行参数 --query 传入，字段为 all:
4. 只取前 5 条，打印每条的标题、年份、arXiv ID
5. 请求超时 30 秒，网络错误要重试 3 次、带退避等待
6. 每一行代码加中文注释，写完向我解释这个脚本的结构

写完后帮我运行：python my_arxiv_search.py --query "large language model"
```

> 注意上面第 1 条：**文档 URL 写进了提示词**。这样 opencode 会先抓取官方文档、
> 按最新参数表写代码，而不是凭训练记忆。如果它抓取失败，你再把要点复制给它——
> 但要先问一句"你刚才读到的限流规则是什么"，检验它真的读了。

### 第 3 步：验收检查点

- [ ] 脚本能跑通，打印出 5 条真实论文标题（不是编造的）
- [ ] **你能回答**：`search_query` 里 `all:` 换成 `ti:` 结果有什么不同？
- [ ] 故意把 `max_results` 改成 2，结果变 2 条——参数是你控制而不是 AI 控制

> ⚡ **加练（完成快的同学，其余课后补）**：一句话让 opencode 把脚本升级——
> "把 my_arxiv_search.py 升级：新增 `--max-results`（默认 20，arXiv 单次请求
> 最多 100 条，一次请求就够，不用翻页），结果写入带 query / source /
> retrieved_at 留痕列的 `my_arxiv.csv`"。这份 CSV 是后面多源对比和
> skill 构建的原料。

> 💡 如果报错：**把完整报错原样复制给 opencode**，附一句"这是运行结果，帮我修"。
> 不要自己转述"大概是什么错"——AI 看原文比看转述诊断准得多。

---

## 4. 微实验 2 · 换源：Google Scholar 与 Web of Science

**任务**：同一检索式再查 GS 和 WoS，结果写入带留痕列的 CSV，对比三源覆盖差异。
这一步练的重点是：**换一个 API，你的工作流程不变——找文档 → 找四样东西 → 改 prompt**。

### 第 1 步：读两份新文档，各自找出"四样东西"

打开第 2 节总览表里的两份官方文档，每份花 5 分钟填出：

| 要找的 | SerpAPI (Google Scholar) | WoS Starter |
|---|---|---|
| 接口地址 | ？ | ？ |
| 必填参数 | ？ | ？ |
| 认证放哪 | ？ | ？ |
| 限流规则 | ？ | ？ |

> 填不出来就问 opencode："帮我把这份文档里的接口地址、必填参数、认证方式、
> 限流规则整理成表格"，但**先自己读一遍再问**——这是练的目标。

### 第 2 步：给 opencode 的 prompt

```text
再写两个脚本，检索词同样从 --query 传入，结果都写入带
query/source/retrieved_at 留痕列的 CSV。每个脚本动手前都先读
官方文档（URL 见各条），按文档写、不要凭记忆：

脚本 A：my_scholar_search.py
- 先读官方文档 https://serpapi.com/google-scholar-api
  （重点：engine=google_scholar 的参数表、organic_results
  返回字段结构）
- 调用 SerpAPI（https://serpapi.com/search.json，engine=google_scholar），
  API key 从环境变量 Serp_API_KEY 读取
- 参数 q=检索词，num=20，start 翻页；结果在 organic_results 里，
  抽取 title、publication_info.summary（从中用正则解析年份）、
  cited_by.pid 有值时记被引数
- 没有配置 key 时打印一句提示并正常退出（不要报堆栈错误）

脚本 B：my_wos_search.py
- 先读官方文档 https://developer.clarivate.com/apis/wos-starter
  （重点：/documents 端点的请求参数、hits 返回结构；该页需要
  登录，读不到就直接用我下面给的字段说明）
- 调用 Web of Science Starter API
  （https://api.clarivate.com/apis/wos-starter/v1/documents，
  注意是 apis 不是 api）
- 请求头 X-ApiKey 从环境变量 WOS_API_KEY 读取
- 参数：q=检索式、db=WOS、limit=25、page 翻页、sortField=RS+D
- 检索式写法：裸词变成 TS=("词") 并用 AND 连接，
  例如 large language model 应生成
  TS=("large") AND TS=("language") AND TS=("model")；
  支持 --from-year 参数，生成 PY=2023-2026 形式的年份过滤
- 结果在 hits 数组里，抽取 title、source.publishYear、
  source.sourceTitle、identifiers.doi（转小写）、
  citations 里 db=WOS 那条的 count 作为被引数
- 免费额度 1 次/秒，翻页间隔 sleep 1.1 秒

分别运行，检索词用我 arXiv 用过的同一个，CSV 分别存 my_scholar.csv / my_wos.csv
```

### 第 3 步：验收检查点

- [ ] 三份结果用同一检索词——打开对比：谁覆盖广？谁有被引数？谁没给摘要？
  ```
  （arXiv 一份来自微实验 1 的加练 CSV，或用课程 skill
  `search-paper-ai4s-class` 跑 `--sources arxiv` 补上）
  ```
- [ ] 没配 key 的同学：脚本 A/B 打印提示后**正常退出**而不是崩溃——
  ```
  这叫**降级**，课堂上课只跑 arXiv，课后申请 key 再补全（申请入口：
  SerpAPI https://serpapi.com/manage-api-key 免费约 100 次/月；
  WoS https://developer.clarivate.com 注册学校邮箱 → Create App →
  订阅 Starter 计划，免费 1 次/秒、50 次/天）
  ```
- [ ] **你能回答**：WoS 检索式里 `TS=` 和 `PY=` 分别限定什么？
  ```
  为什么 Google Scholar 无官方 API、我们经 SerpAPI 而不是自己爬？
  ```

> 💡 WoS 报 401/400 时对照：401 = key 无效或没配；400 = 检索式语法错
> （常见：TS 子句之间漏 AND、括号不配平、用了中文引号）。把服务端返回的
> JSON 报错原文贴给 opencode，它会指出检索式哪里错。

---

## 5. 常见报错速查（先自己查，再问 AI）

| 现象 | 多半是 | 一句话给 opencode |
|---|---|---|
| arXiv 没报错但结果为 0 条 | 检索式没加字段前缀或引号没配对 | "检索式语法帮我检查" |
| CSV 乱码 | 编码 | "改成 utf-8-sig" |
| 请求挂住很久 | 没设 timeout | "所有请求加 timeout 30" |
| SerpAPI 401 | key 没读到 | "打印一下环境变量读到了吗" |
| WoS 400 | 检索式语法 | 把返回 JSON 原文贴给它 |
| WoS 429 | 超 1 次/秒 | "翻页间隔加到 1.1 秒" |
| 终端中文乱码（Windows） | 控制台 GBK | 结果写入 .md 文件再打开 |

**贴报错的三要素**：完整报错原文 + 你刚才运行的完整命令 + 一句"帮我修"。

---

## 6. 实验交付（对接 W2 交付物）

本实验结束时你手里应有：

1. **三个可运行脚本**（`my_arxiv_search.py` / `my_scholar_search.py` / `my_wos_search.py`）
2. **三份带留痕列的 CSV**（query / source / retrieved_at，合计 ≥20 篇不重复文献）
3. **一张三源对比表**（覆盖、字段、限流差异——第3节多源实操直接用）
