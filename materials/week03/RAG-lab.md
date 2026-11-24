# Week 03 课堂实验：手写最小 RAG

> 本实验配合 `plan.md` 第2节"综述写作规范与最小 RAG 实操"。`zotero-lab.md` 走 MCP 直连路线（AI 帮你检索 Zotero），本实验走 **手写 RAG** 路线——自己搭一遍切分→嵌入→存库→检索，理解 RAG 的每一步。**W3 第2节课堂全员跟做第3–5步跑通 `rag_demo.py`**（课前已完成第1–2步：装依赖、建脚本、模型文件夹分发到位，课上零下载）；两条路线对比见 `zotero-lab.md` 第8.8节。课后从第6步观察记录与"进阶"继续。

## 实验目标

用几十行代码搭一个能用的**交互式**语义检索 demo——运行后输入问题即可连续检索（回车用默认问题、再回车退出），理解 RAG 的"检索"部分是怎么工作的。

## 背景概念

RAG = Retrieval-Augmented Generation，三步：

```
你的问题 → [检索] 从文献库找相关片段 → [增强] 把片段塞进 Prompt → [生成] LLM 基于片段回答
```

本实验只做**检索**这一步（切分→嵌入→存库→检索），生成部分留给 LLM。因为检索是 RAG 的核心瓶颈——检索不到，再强的 LLM 也编不出靠谱答案。

## 关于 sentence-transformers 模型共享

> **不同应用下载的 sentence-transformers 模型能通用吗？** 能。所有用 `sentence-transformers` 或 `transformers` 的程序共享同一个缓存目录：
>
> ```
> C:\Users\<用户名>\.cache\huggingface\hub\
> ```
>
> 本实验的脚本直接复用，零额外下载。反过来也一样——你先跑本实验下载了模型，zotero-mcp 也能用。首次下载慢，之后所有应用都快。

模型的最小用法就三行——加载模型 + 编码一句话：

```python
from sentence_transformers import SentenceTransformer

# 首次运行时，库会自动从 Hugging Face 下载模型并缓存到本地
model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

# 之后就可以直接使用模型了
vectors = model.encode("这是一个测试句子")
```

首次运行会从 Hugging Face 下载模型并缓存；之后所有程序（包括本实验第 3 步、zotero-mcp 语义检索）都从同一缓存读，零额外下载。本实验脚本里用的短名 `"all-MiniLM-L6-v2"` 和这里的全名 `"sentence-transformers/all-MiniLM-L6-v2"` 指向同一个模型、同一份缓存。

> **课前分发（教师发布模型文件）**：教师会把 `models--sentence-transformers--all-MiniLM-L6-v2` 整个文件夹发给大家（压缩包）。**解压到上面的缓存目录里**，注意两点：
>
> 1. 路径精确到 `C:\Users\<你的用户名>\.cache\huggingface\hub\models--sentence-transformers--all-MiniLM-L6-v2\`（`.cache\huggingface\hub` 不存在就手动逐级创建）
> 2. 文件夹名**原样保留，不要改名**——程序按"组织名/模型名"（`sentence-transformers/all-MiniLM-L6-v2`）查缓存，`/` 自动转成 `--`，名字对不上就找不到
>
> 放好后本实验和 `zotero-lab.md` 第8步**全程零下载**。验证方法：跑本实验第3步，看到 `Loading weights` 秒级完成、没有任何下载进度条，就说明缓存命中了。

## 实验步骤

> 以下命令都在 `ai4s/weeks/week03/` 目录下执行（`cd ai4s\weeks\week03` 后再激活 `.venv`）。`rag_demo.py`、`./rag_chroma_db/`、`rag-app/` 都建在这个目录里。

### 第1步：安装依赖

在项目根目录激活 `.venv`：

```bash
.\.venv\Scripts\activate
python -m pip install chromadb sentence-transformers
```

就这两个包。`chromadb` 是向量数据库，`sentence-transformers` 是嵌入模型库。不需要 LangChain、不需要 FAISS、不需要 API Key。

> 如果你已经装了 `zotero-mcp-server[semantic]`（见 `zotero-lab.md` 第8步），这两个包已经包含了，不用重复装。

### 第2步：准备示例文本

不需要真实 PDF，先用几段示例文本跑通流程。新建 `rag_demo.py`：

```python
# rag_demo.py — 最小 RAG 检索 demo（交互式）

import sys

# Windows 终端默认 GBK，强制 UTF-8 输出中文
if hasattr(sys.stdout, "reconfigure"):
    sys.stdout.reconfigure(encoding="utf-8")

documents = [
    {
        "text": "布迪厄将文化资本分为三种形态：身体化的文化资本（如教养、品味）、客体化的文化资本（如艺术品、书籍）、制度化的文化资本（如学历证书）。文化资本通过家庭教育传递，是阶层再生产的重要机制。",
        "metadata": {"author": "布迪厄", "title": "文化资本与社会再生产", "year": "1986"},
    },
    {
        "text": "数字鸿沟不仅指接入差距，更包括使用技能、信息素养和内容创造能力的差异。老年人、低收入群体和农村居民在数字技能方面显著落后，这加剧了既有的社会不平等。",
        "metadata": {"author": "van Dijk", "title": "数字鸿沟的深层结构", "year": "2020"},
    },
    {
        "text": "社会资本理论认为，个人的社会网络中蕴含的资源可以转化为经济收益和职业机会。弱关系（ acquaintances）比强关系（亲密朋友）更能提供非重复性信息，因此在求职中更有价值。",
        "metadata": {"author": "Granovetter", "title": "弱关系的力量", "year": "1973"},
    },
    {
        "text": "平台经济通过算法管理重新定义了劳动关系。外卖骑手虽然名义上是独立承包人，但平台通过派单算法、评分系统和时间监控实施着比传统雇主更精细的控制。",
        "metadata": {"author": "陈龙", "title": "算法控制与劳动异化", "year": "2023"},
    },
    {
        "text": "数字素养已成为新型文化资本。掌握数字工具的能力不仅影响就业前景，还塑造了个人在公共领域的表达能力和参与能力，形成新的社会区隔。",
        "metadata": {"author": "Ignatow", "title": "数字时代的文化资本", "year": "2017"},
    },
]
```

这是 5 段社科文献的摘要片段，覆盖了文化资本、数字鸿沟、社会资本、平台经济等主题。

### 第3步：嵌入并存入 ChromaDB

接着在 `rag_demo.py` 里加（注意：这段和第4、5步最终会被包进 `main()` 函数里，先把逻辑看懂，文末"完整代码"一节有整体结构）：

```python
import chromadb
from sentence_transformers import SentenceTransformer

def build_collection(client, embedder):
    """把 5 篇文档嵌入并存入向量库"""
    print("嵌入并存储文档...")
    collection = client.get_or_create_collection(name="literature")
    for i, doc in enumerate(documents):
        collection.add(
            ids=[f"doc_{i}"],
            documents=[doc["text"]],
            metadatas=[doc["metadata"]],
            embeddings=[embedder.encode(doc["text"]).tolist()],
        )
    print(f"已存入 {collection.count()} 篇文档到 ChromaDB")
    return collection
```

主流程：加载模型 → 连接数据库 → **库空才建库，已存在就问你要不要重建**：

```python
def main():
    import chromadb
    from sentence_transformers import SentenceTransformer

    # 1. 加载嵌入模型（首次运行自动下载，约 90MB，缓存到 ~/.cache/huggingface/）
    print("加载嵌入模型...")
    embedder = SentenceTransformer("all-MiniLM-L6-v2")

    # 2. 连接 ChromaDB（本地文件存储，存在 ./rag_chroma_db/ 目录）
    client = chromadb.PersistentClient(path="./rag_chroma_db")
    collection = client.get_or_create_collection(name="literature")

    # 3. 建库：库空才写入；已存在则问用户是否重建
    if collection.count() == 0:
        collection = build_collection(client, embedder)
    else:
        ans = input(f"向量库已存在（{collection.count()} 篇）。回车直接检索，输入 r 删除重建：").strip().lower()
        if ans == "r":
            client.delete_collection("literature")
            collection = build_collection(client, embedder)
```

跑一下：

```powershell
python rag_demo.py
```

首次运行会下载 `all-MiniLM-L6-v2` 模型（约 90MB），之后从缓存读。你应该看到 `已存入 5 篇文档到 ChromaDB`。

> 已按"课前分发"放好教师分发的模型文件夹的同学，这里**不会出现下载进度条**，`Loading weights` 秒级完成——那就是缓存命中。

> **建议先设离线模式再跑**（模型已在缓存时）：启动前先执行一次
>
> ```powershell
> $env:HF_HUB_OFFLINE = "1"
> ```
>
> 不设的话，程序启动会先去 huggingface.co 联网校验模型版本——国内网络不通时白等约 10 秒超时（输出 `[WinError 10060]`/`Could not reach the Hub`），然后才回退本地缓存。**这条提示无害**（模型在用缓存，不是报错），但设了离线模式能直接跳过等待。注意：只对当前终端窗口有效，新开窗口要重新设；而且设了它之后**首次下载**也会被禁止——两台机器/新环境需要下载模型时，先别设。

**发生了什么？**

```
文档文本 ──embedder.encode()──→ 384维向量 ──collection.add()──→ ChromaDB（本地磁盘）
```

ChromaDB 把向量、原文、元数据一起存在 `./rag_chroma_db/` 目录里，重启程序不丢失。所以第二次运行**不需要重新嵌入**——库已存在时按提示回车直接检索即可；脚本也不会再报 `already exists`（库空才写入）。

### 第4步：交互式语义检索

接着加检索代码——核心是一个**问问题的循环**：第一次回车 = 默认问题，之后回车 = 退出：

```python
    # 4. 交互式检索：可以连续问多个问题
    print("\n输入自然语言问题做语义检索。第一次回车 = 默认问题，之后回车 = 退出。")
    first = True
    while True:
        hint = "回车=默认问题" if first else "回车=退出"
        query = input(f"检索问题（{hint}）：").strip()
        if not query:
            if first:
                query = "数字技术加剧了什么样的不平等？"
            else:
                break
        first = False

        # 把问题也嵌入成向量，和文档向量算相似度
        query_embedding = embedder.encode(query).tolist()
        results = collection.query(
            query_embeddings=[query_embedding],
            n_results=3,  # 返回最相似的 3 段
        )

        print("\n检索结果（按相似度排序）：")
        for i, (doc, meta, dist) in enumerate(zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0],
        )):
            print(f"\n[{i+1}] 相似度距离: {dist:.4f}")
            print(f"    来源：{meta['author']}({meta['year']}), 《{meta['title']}》")
            print(f"    内容：{doc[:80]}...")
```

跑一下，先回车用默认问题，你应该看到排名靠前的是 van Dijk 的"数字鸿沟"和 Ignatow 的"数字时代的文化资本"——**即使你的问题里没有"鸿沟""素养"这些词**，模型理解了"数字技术加剧不平等"和"数字鸿沟""数字素养"是语义相近的。

接着**再输入一个问题**（比如"找工作时认识的人比亲密朋友更有用？"）看结果怎么变——不用重新启动脚本、不用重建库，这就是向量库"一次建、反复查"的用法。

这就是语义检索和关键词检索的核心区别。

### 第5步：对比关键词检索

最后在**循环内、打印完语义检索结果之后**加一段字面包含对比（缩进和第4步对齐，还在 `while` 循环里）。

> ⚠️ **别直接写 `query_texts=["文化资本"]`**——不传向量时 ChromaDB 会调它**内置的默认嵌入函数**（ONNX MiniLM），首次运行会从 Chroma S3 下载 ~80MB 模型，国内网络实测只有十几 kB/s，大概率卡死超时，而且那个模型和本实验用的 sentence-transformers 模型**不是同一份文件**（见下文对比表）。正确写法是自己嵌入查询词，字面匹配用 `where_document` 过滤：

```python
        # 5. 字面包含检索（where_document $contains），和语义检索同场对比
        literal_results = collection.query(
            query_embeddings=[query_embedding],
            where_document={"$contains": query},
            n_results=3,
        )
        n_literal = len(literal_results["documents"][0])
        print(f"\n--- 字面包含对比 ---")
        print(f"字面包含 '{query}' 的结果（{n_literal} 条）：")
        for doc, meta in zip(literal_results["documents"][0], literal_results["metadatas"][0]):
            print(f"  - {meta['author']}({meta['year']}): {doc[:60]}...")
        if n_literal == 0:
            print("  （无——整句问题不会逐字出现在文本里；但语义检索照样按概念相近找到，这就是两者的核心区别）")
        print()
```

在循环里**输入短词"文化资本"**（直接敲这四个字，回车），对照两栏结果：

- **语义检索**"文化资本"：布迪厄（距离 0.43，文本里字面就有）排第一，但**"数字素养"（Ignatow）、"平台控制"（陈龙）也被捞出来**——概念相近就算命中
- **字面包含**"文化资本"：只返回文本里真含这四个字的布迪厄和 Ignatow 两篇

同一份向量、两种问法，结果集的差异就是语义检索和关键词检索的核心区别——语义检索能找到"概念相近但用词不同"的文献，字面匹配换个说法就找不到。输入整句问题时字面包含几乎必然 0 条（没人会把你的问题原句写进论文），语义检索照样工作——这个对比本身就是答案。

> **为什么 ChromaDB 的默认嵌入函数是另一个模型？** `query_texts` 不带 `query_embeddings` 时，ChromaDB 用自己内置的 ONNX 版 all-MiniLM-L6-v2（从 Chroma S3 下载，存 `~\.cache\chroma\onnx_models\`）；本实验显式传入 `embeddings=`，全程用的是 sentence-transformers 版（HF 缓存，`model.safetensors`）。**同名模型、两份文件、两个缓存目录、互不通用**——这正是"谁在嵌入、用什么嵌入"要时刻清楚的原因（zotero-lab.md 8.3 有同样的对照表）。

> **input() 的中文乱码（Windows）**：VS Code / PowerShell 里中文显示正常但输入变问号，或想彻底防乱码，在文件开头（`sys.stdout.reconfigure` 旁边）加一行 `sys.stdin.reconfigure(encoding="utf-8")`；仍乱码就先执行 `chcp 65001` 再跑。

### 第6步：观察与记录

跑完 demo 后回答以下问题（写在实验记录里）：

1. **语义检索 vs 关键词检索**：用你自己的话说明，为什么"数字技术加剧不平等"这个问题，语义检索能找到"数字鸿沟"的文献，而字面匹配找不到？
2. **向量是什么**：`embedder.encode("一段文本")` 返回的是一个 384 维的浮点数列表。你觉得这 384 个数字代表了什么？
3. **换模型**：把 `all-MiniLM-L6-v2` 换成 `BAAI/bge-small-zh-v1.5`（中文优化模型），检索结果有变化吗？为什么？（提示：MiniLM 是多语言通用模型，bge-small-zh 专注中文；记得在启动时输入 `r` 删库重建，两套向量不通用）

## 完整代码

把上面的片段拼起来就是完整的 `rag_demo.py`（函数 + 交互循环的结构，第3–5步的代码都在 `main()` 里按缩进对齐）。也可以让 OpenCode 帮你生成：

```text
在当前目录写一个 rag_demo.py，用 chromadb 和 sentence-transformers 做最小 RAG 检索 demo。
要求：
1. 内置 5 段社科文献摘要文本（含 author/title/year 元数据）
2. 用 all-MiniLM-L6-v2 嵌入，存入本地 ChromaDB（PersistentClient，路径 ./rag_chroma_db）
3. 交互式检索：运行后 input() 问用户问题（第一次回车 = 默认问题，之后回车 = 退出，
   可连续问多个问题）→ 嵌入 → 查询向量库 → 返回 top3 结果（含相似度距离）
4. 每个问题附字面包含对比：查询词自己 embedder.encode 后传 query_embeddings，
   字面匹配用 where_document={"$contains": 查询词} 过滤。
   不要用 query_texts 不带 query_embeddings 的写法——那会触发 ChromaDB 下载内置
   ONNX 模型（Chroma S3 国内极慢），且与本实验的 sentence-transformers 模型不是同一份
5. 向量库已存在时不重复建库（库空才 add），并提供输入 r 删除重建的选项
6. Windows 终端能跑：python rag_demo.py（stdout 用 sys.stdout.reconfigure(encoding="utf-8") 防 GBK 乱码）
```

## 实验交付

- `rag_demo.py`（代码 + 运行截图）
- `./rag_chroma_db/` 目录（向量库，本地生成）
- 第6步观察记录（3 个问题各 2-3 句）
- 进阶做了 rag-app 的同学：`rag-app/` 运行截图（含至少一段自己拷贝入库的文本 + 对它的检索结果）

## 卡住了

- **模型下载慢 / 超时**
  - huggingface.co 国内不通 → 设环境变量：`set HF_ENDPOINT=https://hf-mirror.com` 后再跑
  - 或在代码里加：`export HF_ENDPOINT="https://hf-mirror.com"`（Linux/Mac）
  - 已拿到教师分发的模型文件夹 → 按上文"课前分发"放对位置（文件夹名不能改），再跑就是零下载
  - 出现 `Could not reach the Hub / Using cached commit hash` / `[WinError 10060]` → 模型在用缓存，只是联网校验超时（会白等约 10 秒再回退，不是报错）；想跳过等待，跑之前在终端执行 `$env:HF_HUB_OFFLINE = "1"`（PowerShell，第3步有说明；新开窗口要重设）
- **卡在下载 `onnx.tar.gz`（~80MB，速度只有十几 kB/s）**
  - 你用了 `query_texts` 不带 `query_embeddings`，触发了 ChromaDB 内置 ONNX 模型的下载（Chroma S3 国内极慢）→ Ctrl+C 停掉，改用第5步的写法：自己 `embedder.encode()` 后传 `query_embeddings`
- `chromadb` **装不上**
  - 确认在 `.venv` 激活状态
  - 网络问题加镜像：`pip install chromadb -i https://pypi.tuna.tsinghua.edu.cn/simple`
- `SentenceTransformer` **报错** `OSError: Model not found`
  - 首次下载需要网络，确认能访问 huggingface.co 或已设镜像
  - 教师分发的文件夹没放对位置或改了名 → 对照"课前分发"检查路径
- **重复运行报 `doc_0 already exists`**
  - 旧版脚本的问题；现在的版本库空才写入，不会再报。如果你的脚本仍报这个错 → 删掉 `./rag_chroma_db/` 重跑，或把 `collection.add` 换成 `collection.upsert`
- **检索结果不对 / 全是距离很大的**
  - 检查 `embedder.encode()` 是否对 query 和 document 用了同一个模型
  - `all-MiniLM-L6-v2` 对中文支持一般，换 `BAAI/bge-small-zh-v1.5` 试试（注意：换了模型要删 `./rag_chroma_db/` 重建——启动时输入 `r`，两套向量不通用）
- `streamlit` **跑不起来 / 报 `streamlit: command not found`**
  - 确认在 `.venv` 激活状态且在 `rag-app/` 目录下；或用 `python -m streamlit run app.py`
- 卡住 → 问 AI（附报错）→ 同学互助解答 → 举手教师

## 进阶（可选）

跑通 demo 后，可以往真实方向扩展：

0. **升级成 Streamlit 应用（推荐先做这个）**：`rag-app/` 文件夹里已备好 `app.py`——把 RAG demo 变成网页版"检索测试台"，不断接收你拷贝的文本做嵌入，随时做检索测试（附字面包含对比、库内容管理）。启动方式见下文"rag-app：Streamlit 检索测试台"。
1. **换成真实文献**：用 PyMuPDF 从 Zotero 导出的 PDF 提取文本，替换示例文本
2. **加入切分**：一篇论文太长，用 `RecursiveCharacterTextSplitter` 或自己按段落切分后再嵌入
3. **接入 LLM 生成**：把检索到的片段拼进 Prompt，调 LLM API 生成回答——这就完成了 RAG 的"G"部分
4. **对比 zotero-mcp 语义检索**：把同样的 5 段文本导入 Zotero，用 `zotero_semantic_search` 检索，对比结果

进阶路线详见 `knowledge/m1-5-rag.md`（完整 LangChain + Chroma Pipeline）。

## rag-app：Streamlit 检索测试台

命令行 demo 跑通后，`rag-app/` 把同样的逻辑（同一个模型、同样的"自己嵌入 + `query_embeddings`"写法）包装成网页应用：

```powershell
cd ai4s\weeks\week03\rag-app   # 进入 rag-app 目录（app.py 在这里）
$env:HF_HUB_OFFLINE = "1"      # 模型已缓存时跳过启动时的联网校验（同第3步说明）
streamlit run app.py           # 必须在 rag-app 目录里运行，否则找不到 app.py
```

浏览器自动打开（默认 `http://localhost:8501`）。界面三个标签页：

| 标签页 | 干什么 |
|---|---|
| **📝 添加文本** | 粘贴任意文本（可附作者/标题/年份）→ 点"嵌入并入库"。从论文摘要、笔记、网页拷贝什么都行，**这就是"不断接收用户拷贝的文本做嵌入"** |
| **🔍 检索测试** | 输入自然语言问题 → 语义检索 top-k（滑条可调），附字面包含对比开关 |
| **📚 库内容** | 查看库内全部文档、逐条删除 |

侧栏有：库状态（篇数）、**一键导入示例文本**（和 `rag_demo.py` 相同的 5 段）、**清空向量库**。

几个认识要点：

- **向量库在 `rag-app/chroma_db/`**（app.py 用 `Path(__file__).parent` 定位，和 `rag_demo.py` 的 `./rag_chroma_db/` 是两个独立的库，互不影响）
- 依赖只多了 `streamlit`（`.venv` 里已装）；`@st.cache_resource` 让模型和库只加载一次，页面反复交互不重复初始化
- 想做的实验：粘贴 3-5 段你**自己研究方向**的文本进库，再用完全不同的说法提问——体会检索质量取决于"入库粒度"和"问法"的关系

## 参考资料

- ChromaDB 文档：[trychroma.com](https://www.trychroma.com/)
- sentence-transformers 文档：[sbert.net](https://www.sbert.net/)
- Hugging Face 镜像：[hf-mirror.com](https://hf-mirror.com/)
- RAG 原理详解：`knowledge/m1-5-rag.md`
- MCP 直连路线：`weeks/week03/zotero-lab.md`
- Python 环境约定：`AGENTS.md` → "本地 Python 环境"一节
