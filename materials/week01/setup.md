# W1 环境安装：Python + Cursor + opencode

> 给谁：AI4S 课程第1周，社科类博士生。  
> 目标：电脑上有一个能跑科研脚本的 Python，Cursor 能找到它，opencode 能在项目里接管工作流。  
> 先修假设：你有一定 Python 基础和大模型使用经验。本文只装环境，不讲语法。

本课程统一口径：**官网 Python 3.13 + 项目内 `.venv` + Cursor（编辑器）+ opencode（命令行 AI 助手）**。不引入 Anaconda 作为默认路径，不用 Trae 备选。

---

## 第一部分：安装 Python 3.13

全班统一 **3.13.x（该系列最新补丁）**。官网首页大按钮现在是 3.14，**不要点它**——后面部分金融/社科数据包、云端镜像仍以 3.13 最省事。不要装低于 3.11 的版本。

### 1.1 下载与安装

1. 打开 [Python 3.13 发布页](https://www.python.org/downloads/)，在页面里找 **3.13** 的 Windows installer（64-bit）。不要选 embeddable、不要选 free-threaded（名字里带 `t` 的）。
2. 运行安装程序，**必须勾选** `Add python.exe to PATH`，再点 Install Now。
3. Mac 同样装 3.13，用 `python3 --version` 验收。

### 1.2 验收

关掉安装窗口后，打开 PowerShell（Windows）或终端（Mac），分别输入：

```text
python --version
pip --version
```

应看到 `Python 3.13.x` 和一行 pip 版本。

**Windows 常见坑**：输入 `python` 却弹出微软应用商店。到「设置 → 应用 → 高级应用设置 → 应用执行别名」里，关掉 `python.exe` / `python3.exe` 两个开关，再重开终端试一次。

**已装过 Anaconda 的**：不要卸载。本课在 IDE 里选官网 3.13，不要用 `base`。若终端里 `python` 总是落到 Anaconda 3.9，执行：

```text
conda config --set auto_activate_base false
```

关掉全部终端再开一个新终端。验收：`python --version` 为 3.13，或用 `py -3.13 --version` 确认 3.13 仍在。

**验收一句**：`python --version` 能打印 3.13，本步才算过。把这行截图留给环境记录。

---

## 第二部分：安装 Cursor 并选中 Python

[Cursor](https://cursor.com/cn) 是基于 VS Code 的 AI-first 编辑器，课堂主演示用它。它**不自带 Python**——只装 Cursor、不装 Python，AI 能写代码但一点"运行"就报找不到解释器。

### 2.1 安装与登录

1. 打开 [https://cursor.com/cn](https://cursor.com/cn)，下载并安装。
2. 首次打开时登录（可用 Google / GitHub 账号）。

### 2.2 选中 Python 解释器

这是本周最容易出错的一步，务必当堂完成。

1. 打开一个空文件夹（或教师发的示例项目文件夹）。
2. 按 `Ctrl+Shift+P`（Mac：`Cmd+Shift+P`），输入并选择 **Python: Select Interpreter**。
3. 选中第一步装的 3.13，路径里应有 `Python313`（例如 `C:\Users\你的用户名\AppData\Local\Programs\Python\Python313\python.exe`）。
   - **不要**选 `anaconda3`
   - **不要**选 WindowsApps 里的商店别名
4. 关掉旧终端，新开一个，再跑 `python --version`，应看到 `3.13.x`。

> 只走命令面板（`Ctrl+Shift+P`）这一条路。不要去改系统 PATH 或 settings.json，容易选错。

### 2.3 先装本周要用的包

本周还没有个人项目仓库，先在当前 Python 里装本周案例要用的包：

```text
python -m pip install pandas jieba
```

- **pandas**：数据分析基础库，案例数据读进来就是 pandas DataFrame
- **jieba**：中文分词库，case-nlp 股吧文本分析案例的依赖

装完让 Cursor 生成脚本验证：读 `weeks/week01/case-nlp/cleaned_300251_comments.csv`，统计包含"韭菜"的评论有多少条，打印结果，**必须点运行且终端无报错**。

个人项目的 `.venv` 虚拟环境留到下面第三部分建仓库时再配。

---

## 第三部分：安装 opencode

[opencode](https://opencode.ai/) 是命令行里的 AI 编程助手。本课程从 W1 起用它接管个人项目（读取 `AGENTS.md`、对话生成代码、直接改文件并运行）。它基于 Node.js，要先有 Node，再全局安装 `opencode-ai`（名字必须带 `-ai`，不要装成 `opencode`）。

### 3.1 安装 Node.js

1. 打开 [https://nodejs.org](https://nodejs.org)
2. 下载 **LTS**（现在是 **24.x**，例如 24.21.0）。**不要点 Current**（现在是 26.x）。
3. 装完后，**关掉旧终端，新开一个** PowerShell（或 Mac 终端），分别输入：

```text
node -v
npm -v
```

应看到 `v24.x.x` 和一行 npm 版本。`npm` 是 Node 的包管理器。

**Windows PowerShell 常见坑**：若出现"无法加载 `npm.ps1`，因为在此系统上禁止运行脚本"——**Node 已经装好了**（`npm.cmd -v` 能打出版本即可）。任选其一修复：

```text
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

输入 `Y`，关掉终端再开，再试 `npm -v`。不必开管理员。或者不改策略，后面一律用 `npm.cmd`（例如 `npm.cmd install -g opencode-ai`）。

已有 Node 18 / 20 且 `npm -v` 能跑的：先试 3.2 步；装 `opencode-ai` 失败再改装 LTS 24。

### 3.2 安装 opencode

Node 验收通过后，在同一终端执行：

```text
npm install -g opencode-ai
```

若 PowerShell 仍禁止运行脚本，改用：

```text
npm.cmd install -g opencode-ai
```

验证：

```text
opencode --version
opencode --help
```

能打印版本号，且 help 里能看到用法，本步才算过。把 `--version` 那行截图留给环境记录。

### 3.3 第一次打开 opencode

**必须在项目根目录打开**（有 `AGENTS.md` 的那一层），不要在别的文件夹。本周先建一个练习用的项目文件夹：

```text
mkdir ai4s-my-project
cd ai4s-my-project
```

在这个文件夹里输入：

```text
opencode
```

会打开终端界面（TUI）。第一次打开需要选模型/登录：

- **默认免费模型可以跟课**
- 有付费模型或 token plan 的，可自行接入

### 3.4 装好后练几条命令

界面打开后，输入框里先打 `/`，再试下面几条：

| 命令 | 做什么 |
|---|---|
| `/connect` | 接入模型服务商（登录或填 API key / token plan）。默认免费模型可以先不连 |
| `/model` | 查看并切换模型（默认免费模型可以跟课） |
| `/session` | 查看、切换对话；一次课里换话题可以用它，不必退出 |
| `/compact` | 把当前对话压短，腾出上下文。聊久了变慢或提示太长时用 |

### 3.5 opencode 的工作模式：对话→改文件→运行

opencode 不只是聊天，它能**直接改文件、运行脚本**。这是科研编程的关键工作模式，与网页版 Chat 的核心区别。下面的两次练习对应课堂分组实验（同题不同问 / 让 AI 改脚本），课上跟着做，课后照着复盘。

**第1练 · 同题不同问（对应课堂实验一）**：同一个任务，两种问法，看输出差多少。

> 先准备数据：把 `cleaned_300251_comments.csv`（约 46MB）复制到你的 `ai4s-my-project` 文件夹——从课程仓库 `weeks/week01/case-nlp/` 复制，或从课程群下载。下面提示词里的路径就按这个写。

先试模糊问法——在 opencode 输入框里打：

```text
帮我分析一下这些股吧评论
```

看它给出什么（多半是一段泛泛的文字建议，或反问你数据在哪）。

再用明确问法——把数据说清楚、把要求列全：

```text
我是社科博士生，刚装好 Python 和 pandas。当前目录的 cleaned_300251_comments.csv
是光线传媒（300251）股吧约 12.4 万条评论，含 date 和 comment 两列。

要求：
1. 在当前目录写一个脚本 mini_emotion.py，统计包含"韭菜""利好""崩盘""涨停"的评论各有多少条
2. 每个关键词再统计评论最集中的年份（按 date 的年份分组）
3. 打印结果
4. 用 pandas 读取，不要自己调 API
5. Windows 终端要能跑：python mini_emotion.py
```

对比两种问法的输出：**说清楚要什么、给足上下文、给足约束**，问得好，AI 才做得准。

opencode 会生成文件、可以让你确认后直接写入磁盘。**点运行**，终端无报错，本步才算过。

**第2练 · 让 AI 改脚本（对应课堂实验二）**：接着让 opencode 把写好的脚本改成用词典打分、输出每日情绪分数。

> 数据也要备齐：把 `zhang_unformal_pos.txt`（正面词表）和 `zhang_unformal_neg.txt`（负面词表）也复制到你的 `ai4s-my-project` 文件夹（与 CSV 一起，都在课程仓库 `weeks/week01/case-nlp/`）。这是网络股吧情感词典，每行一个词，与课堂 app.py Part 3 词典法、课后 notebook 用的是同一份。

```text
请修改 mini_emotion.py：在关键词统计之外，增加词典法每日情绪分数——
当前目录有 zhang_unformal_pos.txt（正面词表）和 zhang_unformal_neg.txt（负面词表），每行一个词。

要求：
1. 读入两个词典，去掉空行和首尾空格
2. 用 jieba 对每条评论分词，词命中正面词表计 +1、命中负面词表计 −1（同一词重复出现重复计）
3. 按日期汇总每条评论的分数，得到每日情绪分数
4. 打印分数最高和最低的各 5 天，并把每日分数存成 daily_sentiment.csv
5. 保留原有的关键词统计功能，不要删
6. Windows 终端要能跑：python mini_emotion.py
```

分词打分 12.4 万条评论约需 1-2 分钟（学生机器可能更久），属正常等待，正好体会"文本数据是要算的"。

改完**再点运行**，看终端输出——这就是"对话→改文件→运行"的完整闭环：AI 直接改磁盘上的文件、改完就能跑。这次改出来的每日情绪分数，与第1节 app.py Part 5 的"每日情绪曲线"用的是同一套词典逻辑——词典法是什么，你亲手跑了一遍。跑通后还可以继续改：往词典里加你自己的词，看情绪分数怎么变。参考/兜底脚本：`case-nlp/daily_sentiment.py`。

### 3.6 AGENTS.md：让 opencode 懂你的项目

在项目根目录建一个 `AGENTS.md`，写清楚：

- 项目目标（一句话）
- 技术栈（Python 3.13、pandas、jieba 等）
- 目录结构约定
- 代码风格偏好

opencode 打开时会自动读取这个文件，作为后续所有对话的上下文。这是 opencode 区别于普通 Chat 的关键——它"记得"你的项目。详细规则见 [opencode 文档：AGENTS.md 与 `/init`](https://opencode.ai/docs/zh-cn/rules/#%E5%88%9D%E5%A7%8B%E5%8C%96)。

本周先写一个最简版 `AGENTS.md`（3-5 行即可），后续随项目演进逐步完善。

---

## 安装顺序速查

| 步骤 | 验收命令 | 截图留档 |
|---|---|---|
| 1. Python 3.13 | `python --version` → 3.13.x | ✅ |
| 2. Cursor + 选解释器 | `Ctrl+Shift+P → Python: Select Interpreter` 选 Python313 | ✅ |
| 3. 装包 pandas + jieba | `python -m pip install pandas jieba` 无报错 | — |
| 4. Node.js LTS | `node -v` → v24.x | — |
| 5. opencode | `opencode --version` | ✅ |
| 6. 第一个脚本跑通 | `python mini_emotion.py` 终端无报错 | ✅ |

---

## 卡住了

- **安装包下不动**：换校园网 / 手机热点；或用课程群里教师提供的离线安装包。
- **`python` 不是 3.13**：把安装目录加到 PATH，或重装并勾选 Add to PATH；已装 Anaconda 的执行 `conda config --set auto_activate_base false` 后重开终端。
- **Cursor 运行按钮灰 / 提示 No interpreter**：`Ctrl+Shift+P → Python: Select Interpreter` 选 `Python313`。
- **PowerShell 禁止运行 npm**：用 `npm.cmd` 代替 `npm`，或执行 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` 后重开终端。
- **opencode 装不上**：先确认 `node -v` 能跑；仍失败带报错截图进课程群，周末补齐。
- **模型连不上**：先用默认免费模型；有 API key 的按 `/connect` 指引接入。
- **电脑装不上环境**：借同学机器完成课堂演示，报错截图进课程群；本周交付不因安装失败清零，但须本周内补齐。

---

## 参考资源

- [opencode 官网](https://opencode.ai/) / [opencode 文档](https://opencode.ai/docs/) / [AGENTS.md 规则](https://opencode.ai/docs/zh-cn/rules/)
- [Cursor 官网](https://cursor.com/cn) / [Cursor 文档](https://docs.cursor.com/)
- [Python 官网](https://www.python.org/downloads/)
- [Node.js 官网](https://nodejs.org/)
- 本科生课程同名材料（含更多排障细节，可参考）：`data-analysis/weeks/week01/python-setup.md`、`data-analysis/weeks/week02/opencode-setup.md`
