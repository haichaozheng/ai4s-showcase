# Week 03 课堂实验：Zotero MCP

> **课堂进度**：W3 第1节已覆盖本实验核心——**第6步**（AI 查重 + 批量打标签，第1节步骤 1）、**第4–5步**（验证连接与找→读→标注流水线，第1节步骤 3）、第8步的语义检索初尝（第1节末 2 分钟埋点，第2节 RAG 原理时回收）。安装配置（第1–3步）与写入授权（6.1 节）课前完成。本文档供课后深入：从**第5步**（文献检索与阅读的深读技巧）继续，第6步的完整整理流程、第8步语义检索配置与构建（第2、3节课前已建索引）。

## 实验目标

把 Zotero 文献库接入 OpenCode，让 AI 直接检索、阅读、标注你的文献，体会 MCP 如何把本地工具变成 AI 的"手和眼"。

> **本课约定：所有命令都在 OpenCode 里执行**——对 AI 说"跑 XXX"，AI 通过 shell 执行。这样的好处：环境变量由 `opencode.jsonc` 统一注入，不存在"终端里读不到配置"的分叉（原因见 8.3 的环境隔离说明）。

## 背景概念

- **MCP（Model Context Protocol）**：AI 助手与外部工具通信的标准协议。AI 通过 MCP 调用工具（检索、读取、写入），无需你手动复制粘贴。
- **Zotero MCP**（[github.com/54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp)）：把 Zotero 文献库暴露为 MCP 工具，AI 可检索文献、读 PDF 全文、提取标注、按 DOI 添加文献。
- **本地模式**（`ZOTERO_LOCAL=true`）：直接读本地 `zotero.sqlite`，离线、快、无需 API Key。

## 实验步骤

### 第1步：Zotero 准备

1. 确认 Zotero 已安装并在运行（W1 装好的）
2. 打开 **Settings → Advanced**，勾选 *Allow other applications on this computer to communicate with Zotero*
3. 确认 W2 的文献已导入（≥20 篇），W3 课堂上去重、打标签

### 第2步：安装 zotero-mcp

在项目根目录激活 `.venv`（见 `weeks/week02/venv-setup.md`），然后安装：

```bash
.\.venv\Scripts\activate
python -m pip install zotero-mcp-server
```

验收：

```bash
zotero-mcp version
```

应输出 `Zotero MCP v0.13.x` 或更高。

### 第3步：配置 OpenCode 连接

在项目根目录的 `opencode.jsonc` 中添加 zotero MCP 服务器：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "zotero": {
        "type": "local",
        "command": ["zotero-mcp"],
        "environment": {
          "ZOTERO_LOCAL": "true",
          "PYTHONUTF8": "1"
        }
      }
    }
  }
}
```

> **Windows 必须加 `PYTHONUTF8: "1"`**。Windows 默认 stdout 编码是 GBK，zotero-mcp 通过 stdout 传输含中文/emoji 的文献数据时会触发 `UnicodeEncodeError` 崩溃。设 `PYTHONUTF8=1` 强制 Python 使用 UTF-8。

字段说明（参考 [OpenCode MCP 文档](https://opencode.ai/v2/docs/mcp-servers/)）：

| 字段 | 作用 |
|---|---|
| `type: "local"` | 本地 stdio 进程，OpenCode 自动启停 |
| `command` | 可执行文件 + 参数 |
| `environment` | 注入子进程的环境变量，覆盖继承值 |

### 第4步：验证连接

1. 重启 OpenCode（`opencode` 命令重开，或在 TUI 里 `/mcps`）
2. 在 OpenCode 里输入 `/mcps`，应看到 `zotero` 状态为 connected
3. 试一条简单指令：

```text
列出我 Zotero 文献库里的所有分类（collections）
```

AI 应调用 `zotero_list_libraries` 或 `zotero_get_collections`，返回你的分类列表。若报错见文末"卡住了"。

### 第5步：文献检索与阅读

用自然语言驱动 AI 检索和阅读文献，体会"检索即用"。一次完整的课堂操作分四个动作：**找 → 读元数据 → 读全文 → 读标注**，按 5.1–5.4 顺序逐个做。

#### 5.1 找：关键词检索

```text
在我的 Zotero 库里搜索关键词 "silicon samples"，列出相关文献标题和作者
```

AI 调用 `zotero_search_items`。返回结果里每条都带 **Item Key**（8 位字符，如 `5RXFETC2`）——后续所有读取操作都靠它定位文献，先把 Key 记下来。

**验收**：列表里每篇有标题、作者、年份和 Item Key。

三条检索规则（工具描述里写明，课堂上易踩坑）：

1. **子串匹配，不是语义匹配**。`silicon samples` 是在标题/作者/年份里找字面连续匹配；每加一个词只会**收窄**结果（AND 关系），不会放宽。按"概念相近"找文献是第 8 步语义检索的事。
2. **默认只搜元数据**（标题/作者/年份）。想连摘要一起搜，说"**连摘要一起搜**"——AI 会切到 `qmode: everything`。
3. **查询没命中时工具会自救**：自动降级为简化查询，再降级到语义搜索。所以"没结果"可能是降级后仍未命中，换更短的词（只留一个实词）再试。

#### 5.2 读：先元数据与摘要，再全文

元数据 + 摘要很便宜（几百 token），先确认是你要的那篇：

```text
读取《Using large language models to generate silicon samples...》的元数据和摘要
```

AI 调用 `zotero_get_item_metadata`。

确认要精读再上全文——**一次调用返回 10K+ token**：

```text
读取《Using large language models to generate silicon samples...》全文
```

AI 调用 `zotero_get_item_fulltext`。

⚠️ **全文默认只返回前 10 页**。17 页的论文读到一半会被截断，输出末尾出现 `TRUNCATED`、`pages 1-10 of 17` 和续读提示——这不是故障，是防爆 token 的设计。接着说：

```text
还剩 7 页，继续读完
```

AI 调用 `zotero_read_pdf_pages(start_page=11, end_page=17)` 补齐（页码为 PDF 物理页，1 起始）。

**省 token 的两个习惯**：

- 只想找全文里某段内容 → 别整篇读，说"在《某篇》里找讲 XX 的段落"，让 AI 用语义检索定位（配合第 8 步）
- 想先看结构再决定读哪 → 说"列一下这篇 PDF 的目录"，AI 调用 `zotero_get_pdf_outline`（几百 token，比全文便宜两个数量级）

#### 5.3 读：提取高亮标注

在刚读过的那篇上继续问（AI 会沿用对话里的 Item Key）：

```text
这篇我标了哪些高亮？
```

AI 调用 `zotero_get_annotations`，返回每条高亮的**页码、颜色、高亮文本**，以及挂在标注上的评论。

**课堂验证（体验直连原库的即时性）**：

1. 第一次提问，记下结果（比如 1 条高亮）
2. 去 Zotero 阅读器里给这篇 PDF 再高亮一句话
3. 回到 OpenCode 原样再问一次 → 新高亮应出现（比如变成 2 条）

同一问题隔一会儿再问，结果可能不同——**这就是 MCP 直连原库和 RAG 预建快照的区别**，第 7 步反思第 2 问要用。

一次汇总多篇的标注：

```text
把我在 [某分类] 里的所有高亮和笔记汇总成一份 digest
```

AI 调用 `zotero_synthesize_annotations`——它只收集、按论文分组素材，**不调用 LLM**；综述式的综合由 AI 在拿到素材后自己完成。

#### 5.4 完整流水线（课堂跟做）

```text
1. 在我的 Zotero 库里搜索关键词 "silicon samples"，列出相关文献标题和作者
2. 读取其中 [某篇文献] 的摘要和全文，总结它的核心论点（全文被截断就续读完）
3. 提取这篇的高亮标注，结合标注说明我最关心这篇的什么
```

**验收**：三轮依次触发 `zotero_search_items` → `zotero_get_item_metadata` / `zotero_get_item_fulltext`（+ `zotero_read_pdf_pages` 续读）→ `zotero_get_annotations`。至少 3 条交互截图计入实验交付。

#### 5.5 工具速查

| 你说的话 | AI 调用的工具 |
|---|---|
| "搜索 XXX" | `zotero_search_items` |
| "连摘要一起搜 XXX" | `zotero_search_items`（qmode=everything） |
| "找 XXX 作者 / 某年的文献" | `zotero_search_items`、`zotero_advanced_search` |
| "读这篇的摘要" | `zotero_get_item_metadata` |
| "读这篇的全文" | `zotero_get_item_fulltext`（仅前 10 页） |
| "继续读 / 读第 3-5 页" | `zotero_read_pdf_pages` |
| "列这篇 PDF 的目录" | `zotero_get_pdf_outline` |
| "提取高亮标注" | `zotero_get_annotations` |
| "汇总标注" | `zotero_synthesize_annotations` |

### 第6步：用 AI 辅助文献库整理

这是 W3 plan.md 第1节"文献库整理与去重"的 AI 增强版：

```text
1. 检查我的 Zotero 库里有没有重复文献，列出疑似重复项
2. 把标签 "todo" 的文献按方法论类型分类，输出一张表
3. 把 W2 导入的文献里没打标签的全部标上 "unsorted"
```

对应工具（已对照 v0.13 的 41 个工具实测——**没有 `zotero_find_duplicates`，也没有合并重复的工具**，第 1 条要靠 AI 组合实现）：

| 你说的话 | AI 调用的工具 | 说明 |
|---|---|---|
| "查重复" | `zotero_advanced_search`（或 `zotero_export_bibliography` 导 BibTeX）+ AI 脚本比对 | 无专用查重工具；AI 拉全库元数据，按**标题归一化 / DOI** 分组，输出疑似重复组 |
| "标签为 XXX 的文献" | `zotero_search_by_tag` | 库里必须先有该标签，否则返回 `No items found` |
| "给一批文献打标签" | `zotero_batch_update`（`add_tags`） | 一次调用打一批，比逐条调 `zotero_update_item` 快得多 |
| "给这一篇打标签" | `zotero_update_item`（`add_tags`） | 单条修改。⚠️ `tags` 参数是**整体替换**，追加要用 `add_tags` |
| "合并重复项" | 无 MCP 工具 → 回 Zotero 界面做 | 左侧栏 **Duplicate Items** 视图，勾选后一键合并 |

**教师库实测记录（2026-11-24）**，供你对照预期结果：

- **第 1 条查重**：全库 500+ 条比出 **29 组疑似重复**（例：4 份 *Mathematical discoveries from program search with large language models*、4 份 *NBER working paper series*、3 份 *A Turing test of whether AI chatbots are behaviorally similar to humans*）。多为 W2 重复导入。**查出来后去 Zotero 界面合并，别让 AI 直接删**——删/合并是破坏性操作，先人工核对哪份带 PDF 和标注。
- **第 2 条分类**：W2 如果没建 "todo" 标签，这条返回 `No items found`。先在 Zotero 里打好标签再跑；或让 AI 换成库里已有的标签演示。
- **第 3 条打标**：W2 窗口（11/17–11/23）新增 58 篇，其中 **45 篇无标签**。正确姿势是**先 1 篇试水**（`zotero_update_item` + `add_tags`），去 Zotero 里确认标签出现后，再让 AI 用 `zotero_batch_update` 批量打剩余的。

> **"W2 导入的文献"怎么定位？** Zotero 不记"哪周导入的"，但条目有 `dateAdded` 字段。直接告诉 AI 具体日期，例如"W2 = 2026-11-17 到 2026-11-23 导入的文献"，AI 会用 `zotero_advanced_search` 的 `dateAdded isAfter / isBefore` 圈出这批条目。

#### 6.1 写入授权（做写入操作前必须先做一次）

上面的第 3 条（批量打标签）是**写入操作**——会修改你的 Zotero 库。Zotero 的本地 API 默认只读，写入需要单独授权一次。（第 1 条查重是只读的；第 2 条只读，除非让 AI 替你建标签；合并重复在 Zotero 界面做，不经 MCP。）

动手前可以先让 AI 报一次权限状态：

```text
我的 Zotero 有写入权限吗？
```

AI 调用 `zotero_write_capabilities`，正常应看到 `Mode: local`、`Running Zotero supports local writes: yes`。

**为什么需要授权？** Zotero 本地 API 从设计上就是只读的，防止恶意程序乱改你的文献库。Zotero 10+ 加了本地写入端点，但要用户**显式授权**某个应用才能写——就像手机 App 请求权限一样，你点了"允许"它才能动手。

**前置条件**：Zotero 10 或更新版本。在 Zotero 里点 **Help → About Zotero** 确认版本号 ≥ 10。旧版本没有本地写入端点，只能走 Web API（需要云端 API Key，本实验不涉及）。

**操作步骤**：

1. 确保 Zotero 在运行
2. 在终端（`.venv` 激活状态）运行：

```bash
zotero-mcp authorize-local
```

3. Zotero 会弹出一个对话框，有三个按钮：

| 按钮 | 含义 | 推荐度 |
|---|---|---|
| **Always Allow** | 授权一次，永久有效，密钥存到 `~/.config/zotero-mcp/config.json` | ✅ **选这个** |
| **Allow** | 只授权这一次，关了就失效，不存盘 | 不推荐，每次都要点 |
| **Deny** | 拒绝授权 | — |

4. 点 **Always Allow**
5. 验证授权成功：

```bash
zotero-mcp authorize-local --status
```

应输出类似 `write path: local` 或 `can write: true`。

**授权信息存在哪里？** 密钥存在本地文件：

```
C:\Users\<用户名>\.config\zotero-mcp\config.json
```

这个文件只有你能读（owner-only 权限），不上云，和 Zotero 网站的 API Key 无关。

**如果授权失效了**（换了 Zotero 数据库、重装了 Zotero 等），重新跑一次 `zotero-mcp authorize-local` 即可。彻底撤销已存的密钥：

```bash
zotero-mcp authorize-local --revoke
```

或在 Zotero 里 **Settings → Advanced → Clear Write Authorizations**。

**不授权也能用**：第1步（查找重复）和第2步（按标签检索）都是只读操作，不需要授权。只有打标签、添加文献等**经 MCP 的写入**才需要；合并重复在 Zotero 界面里做，与授权无关。如果你只做检索和阅读，跳过这一节。

> 也可以让 AI 在对话里直接调用 `zotero_authorize_local_writes` 工具触发同样的弹窗——效果和命令行一样。详见 [配置文档](https://github.com/54yyyu/zotero-mcp/blob/main/docs/configuration.md#local-write-support)。

### 第7步：对比与反思

完成上述操作后，回答以下问题（写在实验记录里）：

1. **MCP vs 手动**：让 AI 直接读 Zotero，和"手动导出 PDF → 上传到在线 AI 问答工具"相比，省了哪些步骤？哪些场景手动反而更好？
2. **MCP vs RAG**（与 `RAG-lab.md` 对比）：MCP 直连是实时检索原库，RAG 是预先嵌入再语义检索。两者在"找一篇我知道存在的文献"和"找概念相似但我不知道的文献"上各有什么优劣？
3. **写综述时用哪条路线**：你的研究方向下，哪条路线更适合综述写作？为什么？

### 第8步：向量检索（语义搜索）

**先理解：什么是向量化（embedding，也叫嵌入）？** 把一段文字交给**嵌入模型**，输出一串固定长度的浮点数（向量）——可以理解为文本在高维语义空间中的"坐标"。意思相近的文本坐标相近，意思无关的坐标相距很远："数字鸿沟"与"数字不平等"字面不同，向量距离却很小。于是"找相关文献"变成一个几何问题：把你的问题也向量化，到向量库里找**最近邻**即可，不再要求关键词字面匹配。

前面第5步的关键词检索是"字面匹配"——换个说法就找不到。zotero-mcp 还支持**语义检索**：把文献全文向量化存入向量库，按概念相似度排序，能找到"用词不同但概念相近"的文献。

#### 8.1 安装语义搜索组件

`zotero-mcp-server` 基础包不含语义检索依赖，需要额外安装 `[semantic]` extras：

```bash
.\.venv\Scripts\activate
python -m pip install "zotero-mcp-server[semantic]"
```

> `[semantic]` 是 pip 的 **extras 语法**：`pyproject.toml` 里声明了这组可选依赖包括 `chromadb`（向量数据库）、`sentence-transformers`（本地嵌入模型）等，pip 会把它们一起装进 `.venv`。

#### 8.2 选择嵌入模型

通过环境变量 `ZOTERO_EMBEDDING_MODEL` 配置，支持五种嵌入源：

| 值 | 模型 | 需要什么 | 适合场景 |
|---|---|---|---|
| `default`（不设） | ChromaDB 内置 ONNX `all-MiniLM-L6-v2` | 无需 API Key，首次自动下载（~80MB，Chroma CDN，国内实测可能极慢） | 备选：网络通畅时的零配置路线 |
| `openai` | `text-embedding-3-small` | `OPENAI_API_KEY` | 效果好，花钱，需联网 |
| `gemini` | `gemini-embedding-001` | `GEMINI_API_KEY` | 效果好，Google 免费额度 |
| `ollama` | `qwen3-embedding` 等 | 本地跑 Ollama 服务 | 完全离线，需 GPU |
| 任意 HF 模型名（如 `sentence-transformers/all-MiniLM-L6-v2`） | `sentence-transformers` 本地跑 | 无需 API Key，首次从 HF Hub 下载（可走镜像） | **本课推荐**：中文优化、复用 RAG-lab.md 已下载的缓存、换模型对比实验 |

> **本课统一走 HF 路径**（上表最后一行）——与 RAG-lab.md 共享同一份模型缓存，且国内网络下可走 hf-mirror 镜像（`default` 的 Chroma S3 国内实测可能只有十几 kB/s）。配置示例见 8.4。

在 `opencode.jsonc` 的 zotero 环境变量里加一行：

```jsonc
"environment": {
  "ZOTERO_LOCAL": "true",
  "PYTHONUTF8": "1",
  "ZOTERO_EMBEDDING_MODEL": "sentence-transformers/all-MiniLM-L6-v2"
}
```

> **`environment` 字段是什么意思？** OpenCode 启动 zotero-mcp 子进程时注入的环境变量，相当于启动前在终端敲 `set ZOTERO_EMBEDDING_MODEL=...`，只不过 OpenCode 自动做。三个变量的含义：
>
> | 变量 | 作用 |
> |---|---|
> | `ZOTERO_LOCAL=true` | 走本地模式，读本机 `zotero.sqlite` 和 Zotero 本地 API，不连云端 |
> | `PYTHONUTF8=1` | 强制 Python 用 UTF-8 编码（Windows 默认 GBK 会崩，见第3步说明） |
> | `ZOTERO_EMBEDDING_MODEL=...` | 语义检索用哪个嵌入模型：HF 模型名走 sentence-transformers；不设则落到 Chroma 内置 ONNX 模型 |

#### 8.3 模型权重是谁下载的？存在哪？

先区分两个层面：

- **Python 包**（`chromadb`、`sentence-transformers` 代码）→ **pip** 从 PyPI 装，就是上面那行 `[semantic]` 命令
- **模型权重**（几十到几百 MB 的神经网络参数）→ **首次运行时由库自动下载**，不是 pip 装的

权重从哪下、存到哪，**取决于你选的嵌入源——两条路径完全不同**：

| 嵌入源 | 谁下载 | 下载源 | 缓存位置 | 权重文件名 |
|---|---|---|---|---|
| `default` | ChromaDB | Chroma 的 S3 CDN | `C:\Users\<用户名>\.cache\chroma\onnx_models\all-MiniLM-L6-v2\` | `model.onnx`（~80MB） |
| HF 模型名 | `sentence-transformers` | Hugging Face Hub | `C:\Users\<用户名>\.cache\huggingface\hub\models--<org>--<name>\snapshots\<hash>\` | `model.safetensors` |

流程都是：你跑 `zotero-mcp update-db` → 对应库发现本地没权重 → 自动下载 → 缓存到上表位置 → 之后不再重复下载。

> 两个缓存目录互相独立，**文件不通用**：`default` 的 ONNX 权重和 HF 的 safetensors 是两份不同的文件，换嵌入源会重新下载。RAG-lab.md 里 `SentenceTransformer("all-MiniLM-L6-v2")` 走 HF 路径，和 zotero-mcp `default` 的 ONNX 路径虽是同名模型，但格式与存放位置都不同。

> **HF 路径的两个实操要点**：
>
> 1. **模型名怎么写**：直接写 repo ID（`sentence-transformers/all-MiniLM-L6-v2`），不用绝对路径。`sentence-transformers` 按此名先查本地缓存（`models--sentence-transformers--all-MiniLM-L6-v2`，repo ID 里的 `/` 自动转成 `--`），命中就零下载加载；没命中才去 HF Hub 下载。
> 2. **缓存就位后加 `"HF_HUB_OFFLINE": "1"`**：不联网校验版本、直接用缓存，省去每次启动约 100 秒的 huggingface.co 超时重试（国内直连不通时必加）。注意：**首次下载前不要设**——它会让"缓存没有 → 去下载"直接报错。正确顺序：先配 `HF_ENDPOINT` 镜像下载一次 → 再加 `HF_HUB_OFFLINE`。
>
> ⚠️ **CLI 与 OpenCode 环境隔离**：`opencode.jsonc` 的 `environment` 只对 OpenCode 拉起的 MCP 服务生效，你在终端手动跑 `update-db` 时**读不到**。**本课统一在 OpenCode 里让 AI 执行命令**（见 8.5），天然规避此问题；如果确实要自己开终端跑，两个解法任选：
>
> - 设用户级环境变量（推荐，CLI 和 OpenCode 都认）：`[Environment]::SetEnvironmentVariable("ZOTERO_EMBEDDING_MODEL","sentence-transformers/all-MiniLM-L6-v2","User")`，设完**重开终端**才生效
> - 或跑 CLI 前手动补：`$env:ZOTERO_EMBEDDING_MODEL="sentence-transformers/all-MiniLM-L6-v2"`

#### 8.4 国内网络：镜像与下载慢的对策

**`HF_ENDPOINT` 只对 HF 路径有用**——它是 Hugging Face 官方支持的环境变量，指向国内镜像 `hf-mirror.com`。`default` 走 Chroma 的 S3 CDN，**设了也不加速**。

| 你用的嵌入源 | 下载慢 / 不通时怎么办 |
|---|---|
| `default`（Chroma S3） | 重试；或改用 HF 模型名 + 镜像（见下） |
| HF 模型名 / RAG-lab.md | 在 `opencode.jsonc` 加 `"HF_ENDPOINT": "https://hf-mirror.com"`（首次下载用） |

想走 HF 路径（本课主线），把 `ZOTERO_EMBEDDING_MODEL` 设为具体模型名。**首次下载**用镜像配置：

```jsonc
"environment": {
  "ZOTERO_LOCAL": "true",
  "PYTHONUTF8": "1",
  "ZOTERO_EMBEDDING_MODEL": "sentence-transformers/all-MiniLM-L6-v2",
  "HF_ENDPOINT": "https://hf-mirror.com"
}
```

> **首次下载完成后**，把 `HF_ENDPOINT` 换成 `HF_HUB_OFFLINE`（模型已在缓存，无需再联网校验，启动快很多）：

```jsonc
"environment": {
  "ZOTERO_LOCAL": "true",
  "PYTHONUTF8": "1",
  "ZOTERO_EMBEDDING_MODEL": "sentence-transformers/all-MiniLM-L6-v2",
  "HF_HUB_OFFLINE": "1"
}
```

> 没有网络顾虑、或想用中文优化模型，把模型名换成 `BAAI/bge-small-zh-v1.5` 即可（首次需下载）。

> ⚠️ 换嵌入模型 = 换向量空间，已建好的索引不能复用。zotero-mcp 会检测到模型变化并自动重置 collection、要求重建（`update-db --force-rebuild`）。**第一次就定好模型**，别建完索引再换。

#### 8.5 构建向量索引

> **本课统一在 OpenCode 里执行命令**：直接对 AI 说"跑 `zotero-mcp update-db --fulltext` 建语义索引"，AI 通过 shell 执行，环境变量由 `opencode.jsonc` 注入、天然一致，不存在 CLI 与 OpenCode 环境分叉的问题（见 8.3 的隔离说明——那只影响你自己开终端手动跑的场景）。

安装好后，构建一次索引（把文献全文切块、嵌入、存入 ChromaDB）：

```text
跑 zotero-mcp update-db --fulltext 建语义索引
```

| 参数 | 作用 |
|---|---|
| `--fulltext` | 提取 PDF 全文进行嵌入（不加则只索引元数据，语义检索效果差） |
| `--force-rebuild` | 强制完整重建（用于索引出问题时） |

> ⚠️ **跑之前先让 AI 执行 `zotero-mcp db-status`，确认 `Embedding model` 与你在 `opencode.jsonc` 里配的一致**——模型不一致时会静静用错模型建索引，之后查询检测到模型变化、索引被自动重置，几百篇嵌入白跑。显示 `Embedding model: default` 但你配的是 HF 模型名 → 命令跑在了读不到配置的终端里，回到 OpenCode 里让 AI 执行。

- 教师库实测（2026-11-24）：690 条候选、去重后 687 篇待索引，首次构建约几分钟到十几分钟（模型已缓存、本地 CPU 嵌入）；19 篇附件提取不到文本 + 49 篇无文本附件，以 metadata-only 计入——属正常
- 之后每次 Zotero 有新文献，再跑一次 `zotero-mcp update-db` 增量更新即可
- 查看索引状态：让 AI 执行 `zotero-mcp db-status`，完成后 `Document count` 应接近你的库藏量

#### 8.5.1 索引存在哪？（三个存储位置各管一摊）

跑完 update-db，你机器上其实有**三处**东西在配合工作，位置和角色都不同：

| 东西 | 位置 | 角色 | 教师库实测大小 |
|---|---|---|---|
| 原文献库 | `C:\Users\<用户名>\Zotero\`（zotero.sqlite + storage） | Zotero 自己的数据，MCP 直连读它 | — |
| 模型权重 | `C:\Users\<用户名>\.cache\huggingface\hub\`（见 8.3 表） | 算向量用的"公式" | 86.7 MB |
| **向量索引** | `C:\Users\<用户名>\.config\zotero-mcp\chroma_db\` | **算出来的结果**（向量 + 被索引的文本/元数据） | **11.4 MB** |

索引目录长这样（`db-status` 输出的 `Database path` 就是指它）：

```
C:\Users\<用户名>\.config\zotero-mcp\chroma_db\
├── chroma.sqlite3                       11.25 MB  主库：687 条的向量 + 元数据 + 文档文本
└── 184a5ba4-...\                        HNSW 向量索引结构（近似最近邻检索用的数据结构）
    ├── data_level0.bin
    ├── header.bin
    ├── length.bin
    └── link_lists.bin
```

三个认识要点：

1. **为什么这么小**：687 篇 × 384 维 × 4 字节（float32）≈ 1 MB 裸向量，其余是被索引的文本和元数据。**模型（86.7MB）不在索引里**——它是"公式"，索引是"算出来的答案"，两者分开存放（对照 8.3 的表）。
2. **不在项目里、不在 `.venv` 里**：索引存在用户目录的 zotero-mcp 配置目录下——换电脑、重装系统时索引不会跟着项目走，需要重跑 `update-db` 重建（好在几分钟的事，且向量是文献库的派生物，重建无损失）。
3. **删掉即清零**：想彻底重建索引，删掉整个 `chroma_db\` 文件夹再跑 `update-db --fulltext` 即可，无任何风险。

#### 8.6 用自然语言做语义检索

索引建好后，直接用自然语言问：

```text
我的文献库里哪些论文讨论了"集体决策中的信息聚合"？
```

AI 会调用 `zotero_semantic_search` 工具，按语义相似度返回文献——不需要标题或关键词精确匹配，只要概念相近就能找到。

#### 8.7 关键词检索 vs 语义检索

| | 关键词检索（`zotero_search_items`） | 语义检索（`zotero_semantic_search`） |
|---|---|---|
| 原理 | 标题/标签/全文里找字面匹配 | 文献全文嵌入成向量，按语义相似度排序 |
| 优势 | 精确，知道关键词时最快 | 能找到"概念相近但用词不同"的文献 |
| 劣势 | 换个说法就找不到 | 依赖嵌入模型质量，需先建索引 |
| 典型场景 | "找 Attention Is All You Need 这篇" | "找讨论协商民主与数字媒体交叉的文献" |

#### 8.8 与 RAG-lab.md 本地 RAG 的关系

这其实就是 `RAG-lab.md` RAG 路线的**自动化版本**——zotero-mcp 把"切分→嵌入→存向量库→检索"整套 pipeline 封装好了，不用自己写嵌入 + 存库代码：

| | RAG-lab.md（手写 RAG） | zotero-mcp 语义检索 |
|---|---|---|
| 嵌入 | 本地 `sentence-transformers`（HF 路径，同款模型） | 本地 `sentence-transformers`（HF 路径） |
| 向量库 | ChromaDB | ChromaDB |
| 文献来源 | 脚本内置示例文本 | 直连 Zotero 库 |
| 维护 | 自己写代码 | `zotero-mcp update-db` 一条命令 |
| 可控性 | 高（可调模型、chunk 策略） | 低（封装好） |

## 实验交付

- `opencode.jsonc` 配置（含 zotero MCP 段）
- 3 条以上自然语言 → AI 调用 zotero 工具的交互截图
- 第7步反思记录（3 个问题各 2-3 句）
- **选做（第8步）**：语义检索索引构建截图 + 一次语义检索的交互截图

> **下一步**：用这条 MCP 管道生成综述——见 `review-gen-lab.md`（自己设计人在环路的 review-gen skill：检索→筛选→提取→组织→生成→审查，闸口由你设计并裁决）。

## 卡住了

- **`/mcps` 显示 zotero 未连接 / fail**
  - 确认 Zotero 在运行：任务管理器看 `zotero.exe` 进程
  - 确认本地 API 开了：浏览器访问 `http://localhost:23119/api/users/0/items?limit=1`，应返回 JSON
  - 确认 `zotero-mcp` 装在 `.venv` 里：`zotero-mcp version` 能跑
  - **Windows 崩溃 / 无输出**：检查 `opencode.jsonc` 里有没有 `"PYTHONUTF8": "1"`（不加必崩）
- **能连接但工具调用报错**
  - 看报错信息里的 `UnicodeEncodeError` → 确认 `PYTHONUTF8` 环境变量
  - 写入被拒（`write capabilities: none`）→ 运行 `zotero-mcp authorize-local` 授权，弹窗选 **Always Allow**（详见第6.1节）
  - 授权了但仍写入失败 → 确认 Zotero 版本 ≥ 10（旧版本不支持本地写入）；或重跑 `zotero-mcp authorize-local --revoke` 后重新授权
  - 找不到文献 → 确认 Zotero 里确实有文献，且不是空库
  - AI 说"没有 `zotero_find_duplicates` 工具" → 正常，v0.13 就没有；让它用 `zotero_advanced_search` 拉全库后比标题/DOI（见第6步工具表）
- **装不上 zotero-mcp-server**
  - 确认在 `.venv` 激活状态：终端前缀有 `(.venv)`
  - 网络问题换热点；pip 太慢加 `-i https://pypi.tuna.tsinghua.edu.cn/simple`
- **语义检索不可用 / `Semantic search is not available`**
  - 没装 `[semantic]` 组件：`python -m pip install "zotero-mcp-server[semantic]"`
  - `update-db` 卡住或超时：`default` 走 Chroma S3 CDN（国内实测可能极慢）→ 改用 HF 模型名 + `"HF_ENDPOINT": "https://hf-mirror.com"`（见 8.4）；只有 HF 路径才吃镜像
  - 磁盘空间不够：模型 ~90MB + 索引（每百篇约 2MB，见 8.5.1）——空间需求远比想象小，确认空间充足即可
  - 首次很慢正常：要下载模型 + 跑全文嵌入，后续增量更新很快
- **db-status / update-db 卡在 `Could not reach the Hub` / `Retrying`**
  - 正在联网校验模型版本，国内直连 huggingface.co 超时 → 缓存已就位时加 `"HF_HUB_OFFLINE": "1"`（见 8.3）
  - 出现 `offline mode is enabled` 提示 → **不是报错**，是离线模式用缓存加载成功的标志
  - 首次下载报 offline 错误 → 先去掉 `HF_HUB_OFFLINE`，配 `"HF_ENDPOINT": "https://hf-mirror.com"` 下载一次（见 8.4）
  - `db-status` 显示 `Embedding model: default` 但你配的是 HF 模型名 → 命令跑在了读不到 `opencode.jsonc` 环境变量的终端里；在 OpenCode 里让 AI 执行（本课统一方式），或按 8.3 设用户级环境变量
- 卡住 → 问 AI（附报错）→ 同学互助解答 → 举手教师

## 参考资料

- Zotero MCP 项目：[github.com/54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp)
- Zotero MCP 工具列表：[docs/tools.md](https://github.com/54yyyu/zotero-mcp/blob/main/docs/tools.md)
- Zotero MCP 配置：[docs/configuration.md](https://github.com/54yyyu/zotero-mcp/blob/main/docs/configuration.md)
- OpenCode MCP 配置：[opencode.ai/v2/docs/mcp-servers](https://opencode.ai/v2/docs/mcp-servers/)
- 本项目 `opencode.jsonc`：仓库根目录
- Python 环境约定：`AGENTS.md` → "本地 Python 环境"一节
