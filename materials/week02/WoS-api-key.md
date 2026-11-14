# WoS-paper-search：从 Web of Science 获取文献的简易指南

> 一个独立的入门教程，不依赖项目里的其他实验。
> 跑通之后，你手里会有一份 JSON 格式的英文文献元数据
> （标题、作者、年份、期刊、DOI、被引……），可以直接进 Excel 或 pandas。

---

## 一、原理：一句话说清楚

Web of Science 官方提供了一个 **Starter API**（REST 接口）：
用 Python 的 `requests` 库发一个 HTTP GET 请求，把检索式交给它，
它返回一段 JSON，里面就是命中的文献条目。

```
你的代码 ──GET 请求(检索式 + API Key)──→ WoS 服务器
你的代码 ←──────JSON 响应(每页最多 50 条)──── WoS 服务器
```

免费、不需要装任何额外软件，只需要一个 Key。

---

## 二、准备工作（只做一次）

### 1. 申请免费的 API Key

API 入口页面：<https://developer.clarivate.com/apis/wos-starter>
（这是 **Web of Science Starter API** 的官方页面，介绍、订阅、文档都在这里；
Clarivate 平台上还有几十个其他 API，做文献检索只认准 Starter 这一个。）

1. 打开 <https://developer.clarivate.com>，注册账号（用学校邮箱即可）；
2. 登录后：**Create App** → 随便起个名字 → 订阅 **Starter** 计划（免费）；
3. 复制生成的 Key。

> 免费额度：**1 次/秒、50 次/天**。每次请求最多拿 50 条，
> 做课程实验足够了，不用担心超额。

> **为什么是 Starter？** Clarivate 平台上与 Web of Science 相关的 API 有好几个：
> - **Starter API** —— 唯一免费的个人可用检索接口 ✅
> - API Expanded —— 功能最全但收费，需机构采购
> - API Lite —— 旧接口，已被 Starter 取代，不要订阅
> - Journals / Researcher / SUSHI 等 —— 查期刊指标、作者档案、使用统计，都不能用来搜论文
>
> 所以个人做文献检索，订阅 Starter 即可，其他都不用碰。

### 2. 保管好 Key

Key 不要写进代码、不要截图发群里。推荐放在 `.env` 文件里（已被 .gitignore 忽略）：

```
WOS_API_KEY=你复制的Key
```

注意：等号前后不要有空格，Key 不要带引号。

---

## 三、写检索式（本教程的核心）

### 1. 按语法拼起来

本实验用的检索式（就这么简单）：

```python
QUERY = 'TS=("prediction market" OR "information market")'
```

概念更多时用 `AND` 连接，例如：

```python
# QUERY = 'TS=("prediction market") AND TS=("blockchain") AND PY=2015-2026'
```

语法规则就五条：

| 规则 | 说明 |
|---|---|
| `TS=(...)` | 在**标题/摘要/作者关键词**里搜（最常用的字段标签） |
| 同义词之间用 `OR` | 扩大召回：`"prediction market" OR "information market"` |
| 概念之间用 `AND` | 收窄范围：两个概念都要命中，如 `TS=("prediction market") AND PY=2015-2026` |
| 多词短语加英文双引号 | 不加引号会被拆成单词分别 AND，命中率暴跌 |
| `PY=2019-2026` | 限定出版年，按需修改 |

其他常见字段标签：`TI=`（仅标题）、`AU=`（作者）、`SO=`（期刊名）、
`DO=`（DOI）、`DT=`（文献类型）。
> 注意：Starter API 只支持一部分字段标签（完整清单见官方客户端仓库的 README：
> <https://github.com/clarivate/wosstarter_python_client>），
> 带基金、通讯作者等高级字段的检索只有收费的 API Expanded 才支持。

### 2. 自检两个常见错误

- 括号不配对、引号用了中文引号 `“”` → 会被 API 以 400 拒绝；
- 短语超过 5 个词 → 几乎不可能连续命中，改成更短的词。

---

## 四、发请求：逐行读懂 `wos_get()`

### 1. 它是什么、为什么要有它

`wos_get()` 是我们**自己写的一个 Python 函数**（WoS 官方不提供 Python 库，
任何语言都是发 HTTP 请求）。它只干一件事：

> 把「检索式 + 第几页 + 每页几条」打包成一个 HTTP GET 请求发给 WoS，
> 把返回的 JSON 变成 Python 字典交回来。

写成函数而不写成一行代码，是因为这个请求**会失败**——网络抖动、请求太快、
Key 写错……失败的处理逻辑写一遍就够了，翻页时反复调用它。

### 2. 先看懂一次请求长什么样

一次 API 请求由三部分组成，对应下面注释里的三块：

```python
url = "https://api.clarivate.com/apis/wos-starter/v1/documents"
#      └────────── 固定的接口地址，永远不变 ──────────┘

params = {"q": QUERY, "db": "WOS", "limit": 25, "page": 1, "sortField": "RS+D"}
#         └─检索式─┘  └─哪个库─┘  └─每页几条─┘  └─第几页┘  └─排序方式─┘

headers = {"X-ApiKey": "你的Key", "Accept": "application/json"}
#           └──身份凭证：证明你有资格调用──┘  └─我要JSON格式的回复─┘
```

`params` 里每个参数的含义（`requests` 会自动把它们拼到 URL 后面变成
`...?q=...&db=WOS&limit=25&page=1`，不用手动拼）：

| 参数 | 必填？ | 含义 | 注意 |
|---|---|---|---|
| `q` | 是 | 检索式（第三节写好的那串） | 括号、引号要配对，否则 400 |
| `db` | 是 | 搜哪个库，固定写 `WOS` | Starter 只提供这一个 |
| `limit` | 否 | 每页返回几条，**最大 50** | 默认 10；想快就用 50 |
| `page` | 否 | 第几页，从 1 开始 | 翻页就是改这个数字 |
| `sortField` | 否 | 排序 | `RS+D` = 按相关度降序（默认），`PY+D` = 按年份降序 |
| `detail` | 否 | 返回数据详略 | `full`（默认）返回全部字段；`short` 只返回精简版 |

`headers` 里 `X-ApiKey` 是 WoS 规定的身份头（注意不是常见的
`Authorization: Bearer ...`，每家 API 规矩不同）。

### 3. 完整代码（先整体看一遍）

`wos_get()` 的完整代码如下，一共 30 行左右，先整体扫一眼，
然后看 3.1 的逐行拆解。**这段可以直接复制运行**（需要先定义 `API_KEY`、
`WOS_BASE` 两个常量，见第五节的完整脚本）：

```python
def wos_get(q: str, page: int = 1, limit: int = 25, timeout: int = 40) -> dict:
    """发一次 WoS /documents 检索请求，带退避重试，返回解析好的字典。

    参数
    ----
    q       : 检索式（第三节写好的那串），必填
    page    : 第几页，从 1 开始，默认 1
    limit   : 每页几条，最大 50，默认 25
    timeout : 单次等待回复的秒数，默认 40

    返回
    ----
    成功时返回响应字典（用 raw["metadata"]["total"] 和 raw["hits"] 取数据）；
    失败时抛 RuntimeError，异常消息里写了怎么修。
    """
    # ① 组装请求：地址 + 查询参数 + 身份头（三块的含义见本节第 2 小节）
    url = f"{WOS_BASE}/documents"
    params = {"q": q, "db": "WOS", "limit": limit, "page": page,
              "sortField": "RS+D"}
    headers = {"X-ApiKey": API_KEY, "Accept": "application/json"}

    # ② 重试循环：最多试 4 次
    last = ""
    for attempt in range(4):
        try:
            r = requests.get(url, params=params, headers=headers, timeout=timeout)
        except requests.RequestException as exc:
            # 断网/超时抛的是异常不是错误码，必须接住
            last = f"{exc.__class__.__name__}: {exc}"
            time.sleep(2 * (attempt + 1))       # 退避：等 2, 4, 6, 8 秒
            continue

        if r.status_code == 200:
            return r.json()                     # 成功，解析成字典返回

        if r.status_code == 401:                # 身份问题，重试没用
            raise RuntimeError(
                "401：Key 无效或未填。检查 .env 里的 WOS_API_KEY，重启后再试。"
                f"服务端原文：{r.text[:200]}"
            )
        if r.status_code == 400:                # 检索式语法问题，重试也没用
            raise RuntimeError(
                f"400：检索式有问题（常见：括号不配对、短语没加引号、"
                f"用了中文引号）：{r.text[:300]}"
            )
        if r.status_code in (429, 500, 502, 503, 504):
            # 请求太快或服务器抖动，退避后重试
            last = f"HTTP {r.status_code}"
            time.sleep(3 * (attempt + 1))
            continue

        raise RuntimeError(f"未预期的响应（HTTP {r.status_code}）：{r.text[:300]}")

    # ③ 4 次都没成功：宁可报错，也不默默返回空数据
    raise RuntimeError(f"重试 4 次仍失败：{last}")
```

#### 3.1 逐行拆解

**函数签名**

```python
def wos_get(q: str, page: int = 1, limit: int = 25, timeout: int = 40) -> dict:
```
四个参数：检索式必给；`page`/`limit` 有默认值（不传就第 1 页、25 条）；
`timeout=40` 表示 40 秒还没等到回复就放弃本次尝试——**没有 timeout 的网络
请求可能永远挂住**，所以永远要设。`-> dict` 是类型注解，告诉读代码的人
这个函数成功时返回字典。

**① 组装请求**

```python
    url = f"{WOS_BASE}/documents"
    params = {"q": q, "db": "WOS", "limit": limit, "page": page,
              "sortField": "RS+D"}
    headers = {"X-ApiKey": API_KEY, "Accept": "application/json"}
```
对应第 2 小节讲的三块：接口地址、查询参数、身份头。

**② 重试循环**

```python
    for attempt in range(4):
```
最多试 4 次。为什么？服务器偶尔抽风（5xx）、请求偶尔太快（429），
这些都不是你的错，等一两秒再试通常就好了。注意 401/400 不在重试之列——
Key 错了、检索式写错了，试多少次都是错，要人去修。

```python
        try:
            r = requests.get(url, params=params, headers=headers, timeout=timeout)
        except requests.RequestException as exc:
            last = f"{exc.__class__.__name__}: {exc}"
            time.sleep(2 * (attempt + 1))
            continue
```
`requests.get()` 发出请求。**try/except 包住它**是因为断网、超时会直接抛异常
（不是返回错误码），不接住程序就崩了。接住后 `sleep(2, 4, 6, 8 秒)` ——
等待时间随尝试次数**递增**，这叫"指数退避"，是礼貌行为：
对方正在拒绝你，别立刻又戳一下。`last` 把失败原因记下来，
万一 4 次全失败，报错信息里能看得见。

```python
        if r.status_code == 200:
            return r.json()
```
HTTP 状态码 `200` = 成功。`r.json()` 把 JSON 文本解析成 Python 字典，
这就是函数的最终产出。

```python
        if r.status_code == 401:
            raise RuntimeError(...)
        if r.status_code == 400:
            raise RuntimeError(...)
```
两类**不该重试**的错误，直接抛异常让人来修：
- `401`：身份问题（Key 错、没填、没订阅 Starter）；
- `400`：检索式语法问题（括号不配对、中文引号、字段标签拼错）。

```python
        if r.status_code in (429, 500, 502, 503, 504):
            time.sleep(3 * (attempt + 1))
            continue
```
两类**值得重试**的错误：
- `429`：你请求太快或超了每日额度，等一下再试；
- `5xx`：WoS 服务器自己的问题，等一下也许就好了。

**③ 兜底**

```python
    raise RuntimeError(f"重试 4 次仍失败：{last}")
```
4 次都没成功，抛出异常终止——**宁可报错也不要默默返回空数据**，
否则后面的代码会把"没抓到"当成"真的没有结果"。

### 4. 返回的 JSON 里有什么

成功的响应是一个字典，两个顶层键最常用：

```python
raw = wos_get(QUERY, page=1, limit=2)
raw["metadata"]["total"]   # 整个库一共命中多少条（决定要不要翻页）
raw["hits"]                # 本页的文献列表，每个元素是一篇文献的字典
```

一篇文献（`hits` 里的一个元素）长这样（节选）：

```json
{
  "uid": "WS:000123456700010",
  "title": "Social network-embedded prediction markets: The effects of...",
  "source":     {"publishYear": "2013", "sourceTitle": "Decision Support Systems"},
  "names":      {"authors": [{"displayName": "Qiu, L."}, ...]},
  "identifiers": {"doi": "10.1016/j.dss.2013.01.007"},
  "citations":  [{"db": "WOS", "count": 25}, {"db": "MEDLINE", "count": 8}],
  "keywords":   {"authorKeywords": ["prediction market", "forecasting"]},
  "links":      {"record": "https://www.webofscience.com/..."}
}
```

注意三点：
- 字段名是 WoS 自己的一套（`sourceTitle` 而不是 `journal`），**嵌套好几层**，
  取值要一层层 `.get()`；
- `citations` 是数组，被引数**按库分开计**，一般取 WOS 那条的 count；
- `source.publishYear` 是**字符串** `"2024"`，进数据库前记得转 int。

### 5. 翻页：就是把 page 加一

```python
MAX_RECORDS = 25
all_hits, page = [], 1
while len(all_hits) < MAX_RECORDS:
    data = wos_get(QUERY, page=page, limit=min(50, MAX_RECORDS - len(all_hits)))
    hits = data.get("hits") or []
    if not hits:
        break                                   # 没有更多了
    all_hits.extend(hits)
    if len(all_hits) >= data["metadata"]["total"]:
        break                                   # 抓完库内全部命中
    page += 1
    time.sleep(1.05)                            # 尊重 1 次/秒限速，别删
```

三个退出条件一个都不能少：抓够 `MAX_RECORDS`、库内没更多、`hits` 为空。
每次翻页之间 `sleep(1.05)` 秒——超过对方 1 次/秒的限制一点点，避免 429。

---

## 五、完整可运行代码（复制就能跑）

```python
import json
import time
from pathlib import Path

import requests

API_KEY = "你的Key"          # 建议改成从 .env 读取
WOS_BASE = "https://api.clarivate.com/apis/wos-starter/v1"

QUERY = 'TS=("prediction market" OR "information market")'


def wos_get(q: str, page: int = 1, limit: int = 25, timeout: int = 40) -> dict:
    """发一次 /documents 请求，带退避重试。逻辑见第四节逐行讲解。"""
    url = f"{WOS_BASE}/documents"
    params = {"q": q, "db": "WOS", "limit": limit, "page": page,
              "sortField": "RS+D"}
    headers = {"X-ApiKey": API_KEY, "Accept": "application/json"}

    for attempt in range(4):
        try:
            r = requests.get(url, params=params, headers=headers, timeout=timeout)
        except requests.RequestException as exc:
            print(f"网络错误，重试中：{exc.__class__.__name__}")
            time.sleep(2 * (attempt + 1))
            continue
        if r.status_code == 200:
            return r.json()
        if r.status_code == 401:
            raise RuntimeError("401：Key 无效或未填，检查后重试")
        if r.status_code == 400:
            raise RuntimeError(f"400：检索式有问题：{r.text[:300]}")
        if r.status_code in (429, 500, 502, 503, 504):
            time.sleep(3 * (attempt + 1))
            continue
        raise RuntimeError(f"HTTP {r.status_code}：{r.text[:300]}")
    raise RuntimeError("重试 4 次仍失败")


# ---- 第 1 步：试水，只拿 2 条 ----
raw = wos_get(QUERY, page=1, limit=2)
print("库内总命中数：", (raw.get("metadata") or {}).get("total"))
print("本次返回条数：", len(raw.get("hits") or []))

# ---- 第 2 步：翻页抓全 ----
MAX_RECORDS = 25          # 想多抓就改这里（25 条 = 1 次请求，50 条 = 2 次）
all_hits, page = [], 1
while len(all_hits) < MAX_RECORDS:
    data = wos_get(QUERY, page=page, limit=min(50, MAX_RECORDS - len(all_hits)))
    hits = data.get("hits") or []
    if not hits:
        break
    all_hits.extend(hits)
    total = (data.get("metadata") or {}).get("total")
    print(f"第 {page} 页：+{len(hits)} 条（累计 {len(all_hits)}，库内共 {total}）")
    if total is not None and len(all_hits) >= total:
        break
    page += 1
    time.sleep(1.05)

# ---- 第 3 步：抽出关心的字段，存成 JSON ----
records = [
    {
        "title":  h.get("title"),
        "year":   int((h.get("source") or {}).get("publishYear") or 0) or None,
        "venue":  (h.get("source") or {}).get("sourceTitle"),
        "doi":    ((h.get("identifiers") or {}).get("doi") or "").lower(),
        "authors": [a.get("displayName") for a in (h.get("names") or {}).get("authors") or []],
        "cited_by": next((int(c.get("count", 0)) for c in h.get("citations") or []
                          if str(c.get("db", "")).upper() == "WOS"), 0),
        "url":    (h.get("links") or {}).get("record"),
        "ut":     h.get("uid"),
    }
    for h in all_hits
]
out = Path("wos_records.json")
out.write_text(json.dumps(records, ensure_ascii=False, indent=2), encoding="utf-8")
print(f"已保存 {len(records)} 条 → {out.resolve()}")
```

跑完你会得到一个 `wos_records.json`，每条文献一个字典，
或 `pd.DataFrame(records)` 进 pandas。

> 另一种方式：Clarivate 官方还发布了一个现成的 Python 客户端
>（<https://github.com/clarivate/wosstarter_python_client>），
> 封装了同样的请求——**本实验不使用**，感兴趣可自行查看参考资料。

---

## 六、三个真实世界的坑

1. **Starter API 不返回摘要**——拿到的记录只有标题、关键词等字段。
   需要摘要时可以拿着 DOI 去开放库（如 OpenAlex）按条补，或人工下载题录。
2. **被引次数按库分开计**——响应里 `citations` 是数组（WOS/MEDLINE 等各一个），
   上面代码优先取 WOS 口径。
3. **额度只有 50 次/天**——翻页别贪多：25 条 = 1 次请求，50 条 = 2 次请求。
   超额了当天就跑不了，第二天再继续。

---

## 七、常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| `401` | Key 错 / 没订阅 Starter / 前后有空格 | 重新复制 Key，确认已订阅 Starter 计划 |
| `400` | 括号不配对、短语没加引号、用了中文引号 | 逐字符检查检索式 |
| `429` | 超过 1 次/秒或 50 次/天 | 代码已自动退避重试；额度用完明天再跑 |
| 结果太少 | 检索式太窄 | 减少一个 `AND` 概念，或给概念加同义词 |
| 结果太多不相关 | 检索式太宽 | 增加一个 `AND` 概念，或把 `TS=` 改成 `TI=` |
| 命中的论文都太旧 | 没限年份 | 加 `AND PY=起始年-结束年` |

> 调检索式的小技巧：同样的检索式可以先在 WoS **网页版**的高级检索里试，
> 网页版会显示命中数和前几条标题，比 API 调试直观；调好后再把式子抄进代码。

---

## 八、参考资料

| 资料 | 链接 | 什么时候看 |
|---|---|---|
| Starter API 官方页面（订阅、拿 Key） | <https://developer.clarivate.com/apis/wos-starter> | 申请 Key 时 |
| 官方 Python 客户端（含支持的字段标签完整清单） | <https://github.com/clarivate/wosstarter_python_client> | 想用现成封装、查字段标签时 |
| 接口更新日志（2023-07-20 起新增 `detail` 参数等） | <https://clarivate.com/academia-government/release-notes/wos-apis/release-notes-july-20-2023/> | 接口行为和文档对不上时，先来这查是不是更新了 |
| WoS 高级检索语法（网页版帮助） | <https://webofscience.help.clarivate.com/en-us/Content/advanced-search.html> | 写检索式卡住时 |

> 注：`detail` 参数是 2023 年 7 月的更新加入的——`detail=full` 返回全部字段，
> `detail=short` 返回精简版。不给这个参数就默认返回全量，本教程的代码不用改。

---

## 九、三条纪律

1. **只抓元数据**（标题、作者、期刊、年份），不抓全文、不绕付费墙；
2. **守住限速**：1 次/秒是对方明说的规矩，代码里的 `time.sleep(1.05)` 不要删；
3. **原始响应原样落盘**，所有清洗都在副本上做——这是"可复现"的最低要求。
