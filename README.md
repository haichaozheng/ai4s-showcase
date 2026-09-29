# 人工智能大模型应用 — AI4S 课程展示

2026 秋季课程展示网站（社科博士生 AI4S 工具栈课程的公开展示入口）。

线上地址：<https://haichaozheng.github.io/ai4s-showcase>

## 技术栈

纯静态 HTML + CSS，无构建步骤。直接推送到 GitHub Pages 即可上线。

## 页面清单

| 页面 | 说明 |
|---|---|
| index.html | 首页：hero + 六大入口卡片 |
| intro.html | 课程简介：目标、PBL、五模块工具栈、共同 AI 基础工具栈、跨学科适配、评分结构 |
| timeline.html | 项目之旅：9 周 4 阶段里程碑（含交付物徽章） |
| projects.html | 精彩项目：开源 AI4S 项目精选（按课程五模块归类，非学生项目） |
| resources.html | 课程资源：视频资料、电子书与社区资源 |
| papers.html | 学术前沿：AI × 社会科学重要论文速递与解读 |
| news.html | 产业新闻：AI4S 领域动态 |

注：本站不设"课程公约"页面。

## 维护约定

### 导航与页脚

导航栏与页脚在所有页面中**重复**。修改导航或页脚时，需同步更新全部 HTML 文件的 `<nav>...</nav>` 与 `<footer>...</footer>` 区块。

### 新增开源项目卡片（projects.html）

复制下方模板，填入内容后放在 projects.html 对应模块 section 的项目区：

```html
<div class="project-card">
    <div class="project-card-head">
        <div class="project-card-title">项目名称</div>
        <span class="badge badge-live">开源</span>
    </div>
    <div class="project-card-author">机构 · 团队</div>
    <div class="project-card-desc">一两句话说明项目解决什么问题、用什么技术、对本课程哪个模块有参照价值。</div>
    <div class="project-tags">
        <span class="tag">标签</span>
        <span class="tag tag-ai">AI 能力</span>
    </div>
    <div class="project-links">
        <a href="#">GitHub</a>
        <a href="#">论文</a>
    </div>
</div>
```

#### 状态徽章

- `badge-live` 开源（绿）
- `badge-demo` 内测/演示（靛蓝）
- `badge-progress` 快速迭代中（黄）

### 新增新闻条目（news.html）

news.html 内已内置注释模板，复制后填入内容、置于 `.news-list` 顶部即可。

## 本地预览

```bash
python -m http.server 8000
```

浏览器访问 <http://localhost:8000/index.html>
