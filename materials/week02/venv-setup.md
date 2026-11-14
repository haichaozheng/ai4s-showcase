# W2 个人项目虚拟环境

> 给谁：AI4S 课程第 2 周，社科类博士生。W1 已在系统 Python 里跑通了课上脚本（`mini_emotion.py`），现在把环境收拢到项目内的 `.venv`，后续 9 周的科研脚本都从这里跑。  
> 前提：已按 `weeks/week01/setup.md` 装好 **Python 3.13** + Cursor + opencode；opencode 能在**项目根目录**打开。  
> 本课约定：用 `python -m venv .venv`，不要用 Anaconda / conda 当项目环境。

W1 装在系统里的 Python 只保证「电脑上有解释器」。从 W2 起你开始构建个人 AI4S 工具栈——文献检索脚本、RAG 知识库、实证代码都跑在项目里。这些包要装进仓库里的 `.venv`，以后换机器、同学复现、期末提交工具栈仓库都靠 `requirements.txt`，不要再往系统 Python 里堆包。

**必须在项目根目录打开 opencode**（有 `AGENTS.md` 的那一层）。不要在别的文件夹建 `.venv`。

---

## 1. 先问：虚拟环境是什么、有哪些做法

把下面整段交给 opencode（小白就按小白问）：

> 我是一个 python 项目小白，告诉我虚拟环境有什么用，有哪些方法可以为我的项目创建虚拟环境？

听完即可。它可能会提到 conda、poetry、uv、`virtualenv` 等。**本课只用一种**：Python 自带的 `venv`，目录名统一叫 `.venv`。不要用 Anaconda；不要把环境命名成 `.env`（那是放 API key 的，W2 起你会用到）。

本课为什么叫 `.venv`：Cursor 会去找这个名字；常见 `.gitignore` 也写了 `.venv/`。课堂和作业不要改成别的名字。

---



## 2. 让 opencode 按本课约定建环境

接着问（可以同一轮对话）：

> 请用 Python 自带的 venv，在当前项目根目录创建名为 `.venv` 的虚拟环境。我用的是 Python 3.13。不要用 conda。建好后告诉我 Windows 和 Mac 分别怎么激活，以及如何把 `.venv/` 写进 `.gitignore`。

你也可以自己在根目录终端执行：

```text
python -m venv .venv
```

含义：`python` 是解释器；`-m` 表示运行自带工具（不是当前文件夹里的文件）；`venv` 负责建环境；`.venv` 是新建的目录。建好后，包只装进这个项目，不会弄脏系统或 Anaconda。

若 `python --version` 不是 3.13（常见于装过 Anaconda），改用：

```text
py -3.13 -m venv .venv
```

Mac 一般用 `python3 -m venv .venv`。

---



## 3. 激活（终端里装包时用；不是每次打开项目都要做）

Windows（PowerShell）：

```text
.\.venv\Scripts\activate
```

含义：`.` 是当前文件夹 → `.venv` 是刚建的环境目录 → `Scripts` 里放 Windows 版的启动脚本 → `activate` 用来启用这个环境。启用后，本窗口里的 `python` / `pip` 指向 `.venv`。提示符前出现 `(.venv)` 即成功。

若提示无法运行脚本：先执行 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`，再重试激活。

Mac / Linux：

```text
source .venv/bin/activate
```

激活后再跑 `python --version`，应仍是 3.13。

---



## 4. 装本课起步包

在**已经出现** `(.venv)` **的终端**里装。W1 的文本脚本要 `pandas` + `jieba`；W2 的文献检索要 `requests`（调用 Semantic Scholar / Crossref API）；W3 的 RAG 知识库要 `chromadb` + `sentence-transformers`（嵌入模型，首次运行会自动下载约 90MB 模型文件）。也可以问 opencode：

> 环境已经激活。请帮我安装 pandas、jieba、requests、chromadb、sentence-transformers，并生成 `requirements.txt`。不要提交 `.venv` 文件夹。

已有 `requirements.txt` 时：

```text
python -m pip install -r requirements.txt
```

还没有该文件时：

```text
python -m pip install pandas requests chromadb sentence-transformers
python -m pip freeze > requirements.txt
```

`pip freeze` 会把当前环境里已装的包及版本写进文件。PowerShell 与 CMD 都可用 `>`。

W4 起按研究需要追加包：调 LLM API 装 `openai`；计量实证装 `statsmodels` / `linearmodels`；文本分析装 `scikit-learn` / `jieba`。每次追加后重新执行一次 `python -m pip freeze > requirements.txt`。

---



## 5. 在 Cursor 里选解释器（日常用法，选一次会记住）

创建 `.venv` 之后，**不要默认它会自动激活。** 以后做项目，在 Cursor 里用：

`Ctrl+Shift+P` → **Python: Select Interpreter** → 选项目里的 `.venv`（路径中应有 `.venv`）

这主要管**点「运行」用哪个 Python**，选一次会记住。不要再选 Anaconda 的 `base`，也不要选系统里的 `Python313` 当本项目的日常解释器。

终端里若没有 `(.venv)` 是正常的：点运行仍然走你选中的解释器。只有要在终端里自己敲 `python` / `pip` 时，才再执行第 3 步。

---



## 卡住了

- 终端里没有 `(.venv)`：正常。日常点运行靠第 5 步选过的解释器；只有在终端里敲 `python` / `pip` 时才激活。
- `python -m venv` 报错或建成的不是 3.13：确认 `py -3.13 --version`，再用 `py -3.13 -m venv .venv`。
- 激活后仍是 Anaconda / 3.9：执行 `conda config --set auto_activate_base false`，关掉全部终端再开；Cursor 里重新选 `.venv` 解释器。
- opencode 建议你用 conda / poetry：谢过它，仍按本页用 `python -m venv .venv`。
- `sentence-transformers` 模型下载慢 / 超时：设环境变量 `set HF_ENDPOINT=https://hf-mirror.com` 后重试（详见 W3 `RAG-lab.md`）。
- 装包很慢或失败：换校园网 / 手机热点；或用清华镜像 `pip install 包名 -i https://pypi.tuna.tsinghua.edu.cn/simple`。
- 同学克隆你的仓库后跑不起来：检查是否提交了 `requirements.txt`、是否误提交了 `.venv/`。对方应自己在本地执行本页第 2–5 步。

