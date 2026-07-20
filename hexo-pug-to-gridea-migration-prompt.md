# Hexo 主题 → Gridea Pro Pongo2 (Jinja2) 迁移全流程 Prompt

> 本 Prompt 基于 gridea-theme-builder Skill（`theme-builder-skill` 仓库）的完整能力设计，覆盖从逆向分析、变量映射、模板重写、CSS 移植、自动化验证、真机走查到映射积累的全流程。**支持 Pug / Swig / Nunjucks / EJS 四种 Hexo 源模板引擎**，目标统一为 Gridea Pro Pongo2 (Jinja2)。直接将本文档交给 AI 助手即可执行。

> **命名约定：** 仓库目录名为 `theme-builder-skill/`，Skill 注册名为 `gridea-theme-builder`（见 `SKILL.md` 前置元数据）。本文档中所有路径均相对于仓库根目录 `theme-builder-skill/`。

---

## 使用方式

**第一步**：将本文件放到仓库根目录 `theme-builder-skill/` 下（即与 `SKILL.md`、`scripts/`、`references/` 同级）。

**第二步**：根据需要选择以下两种模式之一：

**完整迁移模式**（从零开始迁移一个 Hexo 主题，AI 会自动检测源模板引擎）：

```
加载 gridea-theme-builder skill。

请严格按照 theme-builder-skill/hexo-pug-to-gridea-migration-prompt.md 中的流程，
将 Hexo 主题 {HEXO_THEME_PATH} 迁移为 Gridea Pro Pongo2 主题，目标主题名 {THEME_NAME}。
```

**仅积累映射模式**（对已有迁移主题进行事后交叉比对，不执行迁移）：

```
加载 gridea-theme-builder skill。

请严格按照 theme-builder-skill/hexo-pug-to-gridea-migration-prompt.md 中阶段七的流程，
对以下源主题和迁移后的主题执行交叉比对，将映射结果追加到 references/hexo-to-gridea-mappings.md。

源 Hexo 主题：{HEXO_THEME_PATH}
迁移后的 Gridea Pongo2 主题：{GRIDEA_THEME_PATH}
来源名称：{theme-name}
```

**第三步**：AI 将按对应流程执行，每阶段结束后汇报进度并等待你确认。

---

## 前置知识加载

在执行任何迁移操作前，必须完成以下知识加载。所有文件路径均相对于 `theme-builder-skill/` 根目录：

1. 阅读 `SKILL.md`（Skill 总入口，6 步工作流 + 20 条关键规则）
2. 阅读 `references/template-variables.md`（Gridea 所有模板变量，**最重要**）
3. 阅读 `references/jinja2-guide.md`（14 个 Pongo2 致命差异 + 常用模式代码）
4. 阅读 `references/theme-architecture.md`（目录结构、渲染生命周期、静态资源路径规则）
5. 阅读 `references/theme-config-schema.md`（config.json 规范、5 种 GUI 控件类型）
6. 阅读 `references/css-patterns.md`（CSS 变量体系、暗色模式、Markdown 样式）
7. 阅读 `references/quality-checklist.md`（P0/P1/P2 检查清单）
8. **如果存在** `references/hexo-to-gridea-mappings.md`，阅读该文件作为**先验映射知识**（历史迁移中积累的变量对应关系，来源引擎不限）

---

## 阶段一：逆向分析源主题（理解而非翻译）

### 1.0 源引擎自动检测（优先于 1.1 执行）

在扫描目录结构之前，先通过文件扩展名自动识别源主题的模板引擎：

| 文件扩展名 | 源引擎 | 典型目录 | 本项目缩写 |
|-----------|--------|---------|-----------|
| `.pug` | Pug | `layout/` | `pug` |
| `.swig` | Swig | `layout/` | `swig` |
| `.njk` | Nunjucks | `layout/` | `njk` |
| `.ejs` | EJS | `layout/` | `ejs` |

检测后输出：`检测到源引擎：{引擎名}（扩展名：{ext}）`。后续所有阶段一和阶段三的源语法引用均使用此检测结果。

### 1.1 目录结构扫描

遍历源 Hexo 主题的完整目录结构，输出一份**组件清单**，按以下分类（以 Pug 为例，其他引擎替换扩展名即可）：

| 分类 | 源文件（示例：Pug） | 功能描述 | 对应 Gridea 目标 |
|------|-------------------|----------|-----------------|
| 布局 | `layout/layout.pug` | 全局 HTML 骨架 | `templates/base.html` |
| 布局 | `layout/post.pug` | 文章页布局 | 并入 `templates/post.html` |
| 页面 | `index.pug` | 首页 | `templates/index.html` |
| 页面 | `post.pug` | 文章详情 | `templates/post.html` |
| 页面 | `archive.pug` | 归档 | `templates/archives.html` |
| 页面 | `tag.pug` | 标签 | `templates/tag.html` + `templates/tags.html` |
| 页面 | `category.pug` | 分类 | `templates/tag.html`（Gridea 无独立分类页，需自行聚合） |
| 局部 | `_partial/head.pug` | `<head>` 区域 | `templates/partials/head.html` |
| 局部 | `_partial/header.pug` | 导航栏 | `templates/partials/header.html` |
| 局部 | `_partial/footer.pug` | 页脚 | `templates/partials/footer.html` |
| 局部 | `_partial/post-card.pug` | 文章卡片 | `templates/partials/post-card.html` |
| 局部 | `_partial/sidebar.pug` | 侧边栏 | `templates/partials/sidebar.html` |
| 局部 | `_partial/pagination.pug` | 分页 | `templates/partials/pagination.html` |
| 局部 | `_partial/comments.pug` | 评论 | `templates/partials/comments.html` |
| 局部 | `_partial/scripts.pug` | JS 脚本 | 并入 `templates/base.html` |
| 组件 | `_mixins/xxx.pug`（Pug）/辅助函数（EJS）/macro（Swig/Nunjucks） | 可复用组件 | 转换为 `include` 组件或 inline 逻辑 |
| 脚本 | `source/js/` 或 `scripts/` | 主题 JS | 放入 `assets/scripts/` |
| 样式 | `source/css/` | 主题 CSS/SCSS | 放入 `assets/styles/` |
| 图片 | `source/images/` | 静态图片 | 放入 `assets/media/images/` |
| 配置 | `_config.yml` | Hexo 主题配置 | 映射为 `config.json` 的 `customConfig` |

> **注意**：源引擎不同会导致组件复用模式的叫法不同（Pug 叫 mixin、EJS 叫 function/include、Swig/Nunjucks 叫 macro）。阶段一分析时保留原始叫法，阶段三重写时统一转换为 Pongo2 的 `include` 模式。

### 1.2 页面-组件依赖图

对每个页面模板，画出其依赖的组件树（以下以 Pug 为例，不同引擎的 extends/include 语法见阶段三转换表）：

```
Pug 示例：
index.pug
  extends layout.pug
    include _partial/head.pug
    include _partial/header.pug
    block content
      include _mixins/post-card.pug
      include _partial/pagination.pug
    include _partial/footer.pug
    include _partial/scripts.pug
```

### 1.3 关键逻辑提取

对每个源模板文件，提取以下信息并记录：

- **条件分支**：哪些 UI 块有 `if`/`else` 控制？（如侧边栏开关、暗色模式、评论开关、封面图有无）
- **循环逻辑**：哪些元素通过循环生成？（如文章列表、标签云、导航菜单、归档列表）。不同引擎语法不同：Pug 用 `each`，Swig/Nunjucks 用 `{% for %}`，EJS 用 `forEach`/`for`
- **变量使用清单**：列出每个模板中出现的所有 Hexo 变量（`page.xxx`、`config.xxx`、`theme.xxx`、`site.xxx`）和 Helper 函数调用（`url_for()`、`date_xml()`、`truncate()`、`__()` 等），标注出现位置和语义。**这是阶段二推导映射表的输入。**
- **组件复用模式的调用**：按源引擎类型记录：
  - **Pug**：每个 `mixin` 的输入参数和输出 HTML 结构
  - **Swig/Nunjucks**：每个 `macro` 的参数签名和输出，或 `include` 的相对路径
  - **EJS**：`include()` 调用或辅助函数的签名和输出

### 1.4 设计语言提取

从源主题的 CSS/SCSS 中提取以下设计参数（不依赖 AI 猜测，直接阅读源码）：

- **色板**：从 `:root` 变量或 SCSS 变量中提取主色、背景色、文字色、边框色、暗色变量
- **字体栈**：`font-family` 声明，区分 heading 字体和 body 字体
- **间距系统**：根字体大小、行高、段落间距、卡片间距
- **布局参数**：内容区最大宽度、侧边栏宽度、header 高度
- **断点**：`@media` 查询中使用的断点值
- **圆角**：按钮、卡片、图片的 `border-radius`
- **阴影**：卡片的 `box-shadow`
- **过渡**：hover 动画的 `transition` 时长和缓动

---

## 阶段二：推导变量映射表（交叉比对，不硬编码）

### 2.1 核心原则

**不在 Prompt 中硬编码任何 Hexo→Gridea 变量映射。** 所有映射关系由 AI 通过以下步骤动态推导：

1. 从阶段一 1.3 中获取**源主题的所有变量引用清单**（Hexo 侧）
2. 从 `references/template-variables.md` 中获取**Gridea 侧的所有可用变量及字段名**（已在阶段一前置知识中阅读）
3. **如果存在** `references/hexo-to-gridea-mappings.md`，将其作为先验知识——其中已积累的映射关系可以直接复用，但需与当前源主题的实际变量使用情况交叉验证
4. 逐项匹配：对每个 Hexo 变量，在 Gridea 变量目录中寻找语义等价的对应物

### 2.2 推导输出格式

完成推导后，输出一张**本次迁移的映射表**，包含以下维度：

| Hexo 变量 | Gridea 变量 | 匹配依据 | 是否需特殊处理 |
|-----------|------------|---------|--------------|
| `config.title` | `config.siteName` | 语义等价（站点名称） | 字段名不同 |
| `page.date` | `post.dateFormat` | 语义等价（日期展示） | **禁止 `|date` filter**（`post.date` 是 RFC3339 字符串） |
| `page.cover` | `post.feature` | 语义等价（封面图） | 字段名不同，可能为空 |
| `page.prev` | `post.prevPost` | 语义等价（邻接文章） | **方向相反！** 见下方陷阱说明 |
| `site.tags` | `tags` | 语义等价（标签列表） | 子字段名可能不同，需逐一对照 |
| `url_for(path)` | 直接写相对路径 | 功能等价 | 无等价 helper，需手动替换 |

### 2.3 映射推导中的硬性规则（仅限 `template-variables.md` 未覆盖的跨系统陷阱）

以下规则来自实际迁移中踩过的坑，`template-variables.md` 中不会提及（因为那是 Gridea 文档，不涉及 Hexo 对比）：

1. **`post.date` 在 Pongo2 中是 RFC3339 字符串，不是 `time.Time`：** 展示用 `post.dateFormat`，相对时间用 `post.date|relative`，`datetime` 属性直接输出 `post.date`。**禁止 `post.date|date:"..."` filter，会直接报错整页降级。**
2. **`post.prevPost` / `post.nextPost` 的方向语义与 Hexo 相反：** Hexo 的 `page.prev` = 更早的文章；Gridea 的 `post.prevPost` = 更新的文章。移植时要么交换 prev/next 的标签文案，要么交换两个变量在模板中的位置。
3. **`post.content` 是 HTML，必须 `|safe` 输出**，否则标签被转义为纯文本。
4. **archives 的分组键是大写 `group.Year` / `group.Posts`**，小写 `group.year` 静默取空（不报错，循环体不输出）。
5. **友链字段名被重命名：** `link.siteName`（不是 `link.name`）、`link.siteLink`（不是 `link.url`）。
6. **`theme_config` 中的数字值比较前需 `|to_int`：** `{% if loop.index <= theme_config.count|default:8|to_int %}`。
7. **Hexo 的 `config.subtitle`、`config.author` 等不存在于 Gridea `config` 中**，需通过 `customConfig` 声明，模板中通过 `theme_config.xxx` 访问。
8. **Hexo 的 `site.categories` 和 `list_categories()` 在 Gridea 无对应**：没有全局分类列表，需从 `posts` 手动聚合 `post.categories`。
9. **Hexo 的 `__('key')` 多语言机制在 Gridea 不存在**：需硬编码中文文案。
10. **Hexo 的 `partial('path', {data})` 传参机制在 Pongo2 中不可用**：Pongo2 的 `{% include %}` 不传参，数据通过 `{% set %}` 在上文设置，或被 include 的模板直接访问外层变量。

### 2.4 映射表使用方式

AI 完成推导后，**将映射表输出到对话中供用户确认**（不写文件，那是阶段七的事）。用户确认后，AI 在后续所有模板重写中严格参照此映射表进行变量替换。

---

## 阶段三：脚手架生成 + 模板重写

### 3.1 生成脚手架

```bash
python scripts/scaffold_theme.py {THEME_NAME} --engine jinja2 --output-dir ./themes
```

这将生成完整的可运行主题骨架。**不要修改目录结构**，直接在骨架文件上替换内容。

### 3.2 模板重写原则

**核心原则：不是翻译源模板语法，而是理解源模板渲染出的 HTML 结构，用 Pongo2 重新生成同样的 HTML。**

#### 3.2.1 源语法 → Pongo2 转换对照

根据阶段一 1.0 检测到的源引擎，使用对应的转换表。**目标引擎均为 Pongo2 (Jinja2)。**

##### 表 A：Pug → Pongo2

| Pug 语法 | 等价 Pongo2 写法 |
|----------|-----------------|
| `extends layout.pug` | `{% extends "base.html" %}` |
| `block content` | `{% block content %}` |
| `include partials/head.pug` | `{% include "partials/head.html" %}` |
| `if condition` (缩进) | `{% if condition %}...{% endif %}` |
| `else if condition` | `{% elif condition %}` |
| `each item in items` | `{% for item in items %}...{% endfor %}` |
| `+mixinName(arg1, arg2)` | 用 `{% include "partials/xxx.html" %}` 替代（数据通过上下文传，见附录 A） |
| `= variable`（输出） | `{{ variable }}` 或 `{{ variable|safe }}` |
| `!= variable`（不转义） | `{{ variable|safe }}` |
| `// 注释` | `{# 注释 #}` |
| `case page.type` | `{% if %}{% elif %}` 链 |
| `a(href=url) Text` | `<a href="{{ url }}">Text</a>` |
| `div.class#id` | `<div class="class" id="id">` |

##### 表 B：EJS → Pongo2

| EJS 语法 | 等价 Pongo2 写法 |
|----------|-----------------|
| `<% code %>` | `{% code %}` |
| `<%= value %>` | `{{ value }}` |
| `<%- value %>` | `{{ value|safe }}` |
| `<% include('partials/x') %>` | `{% include "partials/x.html" %}` |
| `if (condition) { }` | `{% if condition %}...{% endif %}` |
| `} else if (condition) {` | `{% elif condition %}` |
| `arr.forEach(function(item){ ... })` | `{% for item in arr %}...{% endfor %}` |
| `a && b` / `a \|\| b` | `a and b` / `a or b` |
| `!condition` | `not condition` |
| `arr.length` | `arr|length` |
| `var x = value` | `{% set x = value %}` |
| `a ? b : c` | `{% if a %}{{ b }}{% else %}{{ c }}{% endif %}` |
| `typeof x !== 'undefined'` | `{% if x %}` |
| `<%# 注释 %>` | `{# 注释 #}` |

##### 表 C：Swig → Pongo2

| Swig 语法 | 等价 Pongo2 写法 | 差异说明 |
|----------|-----------------|---------|
| `{% extends "layout" %}` | `{% extends "base.html" %}` | 添加 `.html` 后缀 |
| `{% block content %}` | `{% block content %}` | 完全相同 |
| `{% include "partial" %}` | `{% include "partials/xxx.html" %}` | 路径添加后缀，相对于 `templates/` 根 |
| `{% for item in items %}` | `{% for item in items %}` | 完全相同 |
| `{% if condition %}` | `{% if condition %}` | 完全相同 |
| `{% elseif condition %}` | `{% elif condition %}` | Swig 用 `elseif`，Pongo2 用 `elif` |
| `{{ value }}` | `{{ value }}` | 完全相同 |
| `{{ value|safe }}` | `{{ value|safe }}` | 完全相同 |
| `{% macro name(args) %}` | `{% include "partials/xxx.html" %}` | Pongo2 不支持 macro，改用 include（见附录 A） |
| `{% set x = value %}` | `{% set x = value %}` | 完全相同 |
| `{% filter name %}...{% endfilter %}` | 使用 Pongo2 filter 管道：`{{ value|filtername }}` | filter 语法不同 |
| `{# 注释 #}` | `{# 注释 #}` | 完全相同 |

##### 表 D：Nunjucks → Pongo2

| Nunjucks 语法 | 等价 Pongo2 写法 | 差异说明 |
|--------------|-----------------|---------|
| `{% extends "layout" %}` | `{% extends "base.html" %}` | 添加 `.html` 后缀 |
| `{% block content %}` | `{% block content %}` | 完全相同 |
| `{% include "partial" %}` | `{% include "partials/xxx.html" %}` | 路径添加后缀，相对于 `templates/` 根 |
| `{% for item in items %}` | `{% for item in items %}` | 完全相同 |
| `{% if condition %}` | `{% if condition %}` | 完全相同 |
| `{% elif condition %}` | `{% elif condition %}` | 完全相同 |
| `{{ value }}` | `{{ value }}` | 完全相同 |
| `{{ value\|safe }}` | `{{ value\|safe }}` | 完全相同 |
| `{% macro name(args) %}` | `{% include "partials/xxx.html" %}` | Pongo2 不支持 macro，改用 include（见附录 A） |
| `{% set x = value %}` | `{% set x = value %}` | 完全相同 |
| `a and b` / `a or b` / `not a` | `a and b` / `a or b` / `not a` | 完全相同 |
| `{# 注释 #}` | `{# 注释 #}` | 完全相同 |

> **Swig/Nunjucks 主题的迁移成本远低于 Pug 和 EJS。** 这两种引擎与 Pongo2 共享 90% 的语法，主要差异仅在于文件后缀、路径约定和 macro 转 include。

#### 3.2.2 Pongo2 致命规则（每次写模板前回顾）

编写任何 Pongo2 模板时，必须遵守以下规则，**每修改完一个模板就自查一遍**：

1. **Filter 参数用冒号不用括号**：`{{ value|default:"x" }}`（正确）/ `{{ value|default("x") }}`（错误）
2. **`post.date` 是字符串，禁用 `|date` filter**：用 `post.dateFormat` 展示，`post.date` 用于 `datetime` 属性
3. **`post.content` 必须 `|safe`**：`{{ post.content|safe }}`
4. **逻辑运算符用英文单词**：`and` / `or` / `not`，不能用 `&&` / `||` / `!`
5. **不等判断用 `!=`**：`{% if x != y %}`（正确）/ `{% if not x == y %}`（错误，静默失效！）
6. **长度用 `|length`**：`{% if posts|length > 0 %}`（正确）/ `{% if posts.length > 0 %}`（错误）
7. **不支持三元表达式**：用 `{% if %}...{% else %}...{% endif %}` 替代
8. **不支持 `~` 拼接**：在 `{{ }}` 中直接相邻输出 `{{ a }} | {{ b }}`
9. **否定包含用 `not "a" in b`**：不用 `"a" not in b`
10. **`include` 路径相对 `templates/` 根**：`{% include "partials/header.html" %}`
11. **标签内不可换行**：所有 `{% %}` 和 `{{ }}` 保持单行
12. **archives 分组键大写**：`{{ group.Year }}` / `{% for post in group.Posts %}`
13. **`theme_config` 数字比较前 `|to_int`**：`{% if loop.index <= theme_config.count|default:8|to_int %}`
14. **不支持 `macro`**：用 `{% include %}` 替代

### 3.3 重写顺序（按依赖关系）

**必须严格按以下顺序重写，确保每个阶段都能跑通验证：**

1. **`config.json`** → 映射 Hexo `_config.yml` 到 GTBS customConfig
2. **`assets/styles/main.css`** → 移植源主题 CSS，使用 GTBS 的 CSS 变量体系
3. **`templates/partials/head.html`** → `<head>` 区域（meta、OG、CSS 引用）
4. **`templates/partials/header.html`** → 导航栏（站点标题、菜单循环）
5. **`templates/partials/footer.html`** → 页脚（版权、社交链接、注入代码）
6. **`templates/partials/post-card.html`** → 文章卡片（封面图、标题、摘要、标签、日期）
7. **`templates/base.html`** → 全局骨架（`<html>` → `<head>` include → `<body>` → header → `{% block content %}` → footer → scripts）
8. **`templates/index.html`** → 首页（extends base，文章列表循环 + 分页）
9. **`templates/post.html`** → 文章详情页（extends base，文章内容 + 上下篇导航）
10. **`templates/archives.html`** → 归档页（年份分组，注意 `group.Year` 大写）
11. **`templates/tag.html`** → 单个标签页
12. **`templates/tags.html`** → 标签汇总页
13. **`templates/links.html`** → 友链页
14. **`templates/about.html`** → 关于页
15. **`templates/blog.html`** → 博客列表页（结构同 index）
16. **`templates/memos.html`** → 闪念页
17. **`templates/404.html`** → 404 页面

### 3.4 每完成一个模板的验证

每重写完一个模板文件，立即执行：

```bash
python scripts/validate_syntax.py ./themes/{THEME_NAME}
```

确保零新增错误。如果验证通过，继续下一个模板。如果失败，**只修当前模板**，不跳到其他文件。

---

## 阶段四：CSS 移植策略

### 4.1 不直接复制源 CSS

源主题的 CSS 可能包含 Hexo 特有类名、源模板引擎生成的特定选择器、以及硬编码的色值。移植策略：

1. **保留 GTBS 脚手架的 CSS 变量体系**（`:root` 中的 `--color-*` 变量）
2. **将源主题的色板映射到 CSS 变量**：把源主题的色值填入 `:root` 对应变量
3. **将源主题的字体栈合并到 `--font-sans`**：确保中文字体在正确位置
4. **将源主题的布局参数映射到变量**：`--content-width`、`--header-height`
5. **逐组件对照迁移**：对每个页面组件，对比源主题的 CSS 选择器和 `references/css-patterns.md` 提供的模式，选择最接近的方案
6. **暗色模式**：如果源主题有暗色模式，将暗色变量填入 `[data-theme="dark"]` 块

### 4.2 CSS 移植检查清单

- [ ] `:root` 中所有颜色变量已从源主题提取
- [ ] `--font-sans` 包含源主题的字体声明
- [ ] `--content-width` 与源主题布局一致
- [ ] `.post-content` 的 Markdown 元素样式完整（h1-h6、p、a、blockquote、code、pre、table、img、ul、ol、hr）
- [ ] 暗色模式变量完整映射
- [ ] 响应式断点与源主题一致
- [ ] 代码块样式移植完成
- [ ] 文章卡片样式移植完成
- [ ] 导航栏样式移植完成（含移动端汉堡菜单）
- [ ] 分页器样式移植完成
- [ ] 标签云样式移植完成
- [ ] 页脚样式移植完成

---

## 阶段五：自动化验证 + 内容核查

### 5.1 语法验证

```bash
python scripts/validate_syntax.py ./themes/{THEME_NAME}
```

**目标：零 ERROR，零 WARN。**

常见错误快速修复：

| 验证报错 | 原因 | 修复 |
|---------|------|------|
| filter 括号语法 | 用了 `filter(arg)` | 改为 `filter:arg` |
| 未闭合标签 | `{% for %}` 缺 `{% endfor %}` | 检查配对 |
| include 文件不存在 | 路径错误 | 路径相对 `templates/` 根 |
| 使用了 `macro` | Pongo2 不支持 | 改为 `include` |
| `date` filter 误用 | 对字符串用 `|date` | 改为 `dateFormat` 或 `relative` |
| `&&` / `||` / `!` | JavaScript 风格 | 改为 `and` / `or` / `not` |
| `not in` 写法 | Pongo2 不识别 | 改为 `not "a" in b` |

### 5.2 渲染测试

```bash
python scripts/render_test.py ./themes/{THEME_NAME} --output-dir ./test-output
```

**目标：所有页面渲染成功，无残留模板标签。**

### 5.3 内容级核查（渲染成功 ≠ 内容正确）

渲染测试通过后，必须逐页抽查输出 HTML：

**检查项：**

1. 打开 `test-output/index.html`：
   - 文章列表是否非空（有至少一篇文章标题可见）
   - 分页链接是否可点击
   - 导航菜单是否渲染了所有菜单项
   - 标签云是否显示标签

2. 打开 `test-output/post/{slug}/index.html`：
   - 文章标题、日期、内容是否完整
   - 标签列表是否显示
   - 上下篇导航是否出现
   - 封面图是否正常显示
   - `og:image` 等 meta 标签是否正确

3. 打开 `test-output/archives/index.html`：
   - 归档年份是否正确显示
   - 每篇文章链接是否可点击
   - **特别注意：年份标题是否显示（`group.Year` 大写检查）**

4. 打开 `test-output/tag/{slug}/index.html`：
   - 标签名是否正确显示
   - 该标签下的文章列表是否完整

5. 打开 `test-output/links/index.html`：
   - 友链列表是否显示
   - 友链名称和链接是否正确

6. 空状态检查：
   - 修改 `assets/mock-data-empty.json` 为无文章场景，重新渲染
   - 确认首页显示"暂无文章"而非崩溃

### 5.4 常见内容级问题速查

| 症状 | 可能原因 | 检查位置 |
|------|---------|---------|
| 整页空白 | `{% extends %}` 不是第一个标签 | 每个页面模板第一行 |
| 某 `if` 块完全不渲染 | `not x == y` 静默失效 | 全局搜索 `not .* ==` 改为 `!=` |
| 归档年份不显示 | 用了 `group.year` 小写 | 改为 `group.Year` |
| HTML 标签显示为文本 | 缺少 `|safe` | 所有 `post.content` 和 `post.abstract` 输出 |
| 日期显示为空 | 对字符串用了 `|date` | 改为 `post.dateFormat` |
| 循环体空 | 变量名写错 | 对照阶段二推导的映射表 |
| 友链名称不显示 | 用了 `link.name` 而非 `link.siteName` | 对照 `template-variables.md` 友链字段 |
| 分页不显示 | 用了 `pagination.prev` 但值为空 | 改用 `pagination.hasPrev` 判断 |
| 暗色模式切换无效 | CSS 变量未在 `[data-theme="dark"]` 中定义 | 检查 CSS 暗色变量块 |

---

## 阶段六：真机验证（Gridea Pro 桌面端）

### 6.1 安装主题

将主题目录复制到 Gridea Pro 数据目录：

```bash
# macOS
cp -r ./themes/{THEME_NAME} ~/Documents/Gridea\ Pro/themes/

# 注意：修改 customConfig 后必须重启 Gridea Pro 应用！
# 开发期绕法：每次改 config.json 后，复制主题为新名字的目录
cp -r ~/Documents/Gridea\ Pro/themes/{THEME_NAME} ~/Documents/Gridea\ Pro/themes/{THEME_NAME}-v2
```

### 6.2 真机验证清单

1. 在 Gridea Pro 中切换为新主题
2. 点击"渲染站点"
3. 检查 `output/` 目录中无 `fallback-banner`（黄色降级视图 = 渲染报错）
   ```bash
   grep -rl fallback-banner ~/Documents/Gridea\ Pro/output/ --include="*.html"
   ```
4. 逐页抽查：
   - 首页：文章列表、分页、导航
   - 文章页：标题、内容、标签、上下篇、封面图
   - 归档页：年份分组、文章列表
   - 标签页：标签列表、标签下文章
   - 友链页：友链卡片
   - 404 页：自定义 404 内容
5. 暗色模式切换：确认所有页面在暗色模式下可读
6. 移动端响应式：使用浏览器 DevTools 检查 375px 宽度

---

## 阶段七：映射积累（将本次迁移经验持久化）

> **这是让每次迁移真正产生复利效应的关键步骤。** 迁移完成后，将本次的变量映射关系提取并积累到 `references/hexo-to-gridea-mappings.md` 中，供后续迁移直接复用。

阶段七支持**两种使用模式**：

### 模式 A：在完整迁移流程中自动触发（阶段一→六→七）

完成阶段六真机验证后自动进入阶段七。无需额外操作。

### 模式 B：独立使用——对已有迁移主题进行"事后积累"

如果你手头有之前手动迁移完成的主题（不是通过本 Prompt 的流程迁移的），可以跳过阶段一至六，**单独执行阶段七**。AI 将先对目标主题进行前置预检（语法校验 + 渲染测试），通过后再执行交叉比对。向 AI 发送：

```
加载 gridea-theme-builder skill。

请严格按照 theme-builder-skill/hexo-pug-to-gridea-migration-prompt.md 中阶段七的流程，
对以下源主题和迁移后的主题执行交叉比对，将映射结果追加到 references/hexo-to-gridea-mappings.md。

源 Hexo 主题：{HEXO_THEME_PATH}
迁移后的 Gridea Pongo2 主题：{GRIDEA_THEME_PATH}
来源名称：{theme-name}（用于映射文件中的来源标记）

排除以下文件/组件（可选，如不需要排除可省略）：
- templates/post.html（原因：写得不好，后续可能大改）
- templates/partials/comments.html（原因：实现不完整）
```

AI 将直接跳转到 7.0 开始执行前置预检，不执行阶段一至六。

---

### 7.0 前置预检（交叉比对前必须执行）

> **在开始交叉比对之前，必须对迁移后的主题进行自动化校验，确保其本身没有已知错误。** 如果目标主题本身存在语法错误或渲染降级，从中提取的映射会被污染。

对迁移后的 Gridea 主题目录依次执行以下检查：

**步骤 1：语法校验**

```bash
python scripts/validate_syntax.py <GRIDEA_THEME_PATH>
```

要求：**零 ERROR**。如果存在 ERROR，告知用户具体问题并**停止流程**，要求用户修复后重新运行模式 B。WARN 可以放行，但需在比对报告中标注。

**步骤 2：渲染测试**

```bash
python scripts/render_test.py <GRIDEA_THEME_PATH>
```

要求：**零 FAIL**。如果存在 FAIL（含 fallback-banner 降级），告知用户具体问题并**停止流程**，要求用户修复后重新运行。WARN 可以放行，但需在比对报告中标注。

**步骤 3：结构完整性检查**

参照 `references/quality-checklist.md` 的 P0 级别，快速确认以下边界情况：

- 0 篇文章时不崩溃
- 无封面图文章不崩溃
- 特殊字符标题正确转义
- 每页有唯一 `<title>`

> 如果用户在步骤 1/2 中发现了问题但确认"已知且可接受，继续比对"，则 AI 可以继续，但必须在映射文件中额外标注"该来源存在已知问题：{具体问题}"，并将该来源的置信度降为 L2，即使来源为模式 B。

**步骤 4：输出预检报告**

AI 输出以下格式的预检报告：

```
===== 前置预检报告 =====
主题：{GRIDEA_THEME_PATH}
validate_syntax.py：PASS / WARN(N项) / ERROR(N项)
render_test.py：PASS / WARN(N项) / FAIL(N项)
结构完整性：通过 / 存在以下问题：(...)
结论：允许进入交叉比对 / 拒绝进入，需修复后重试
========================
```

只有结论为"允许进入交叉比对"时，才继续执行 7.1。

---

### 7.1 用户指定排除范围

在开始映射提取前，**用户必须明确告知**哪些页面或组件不参与比对。例如：

```
映射提取时，排除以下文件/组件：
- templates/post.html（这个页面我写得不好，后续可能大改，不要作为参考）
- templates/partials/comments.html（评论系统实现不完整）
```

如果用户没有排除声明，则全部文件参与比对。

### 7.2 交叉比对过程

AI 对**未被排除的**文件，逐一执行以下比对：

1. 从源 Hexo 主题中提取该文件对应的源模板，列出其中使用的所有变量和 Helper
2. 从迁移后的 Gridea 主题中提取对应的 Pongo2 模板，列出其中使用的所有变量和 Filter
3. 逐对匹配，确认本次迁移中实际生效的映射关系
4. 特别注意标注以下类型：
   - **字段名不同的映射**（如 `config.title` → `config.siteName`）
   - **需要特殊处理的映射**（如 `page.date` → `post.dateFormat`，因为 `|date` 不可用）
   - **语义反转的映射**（如 `page.prev` → `post.prevPost`，方向相反）
   - **无对应物的功能**（如 `__('key')` 多语言 → 硬编码）
   - **Helper → Filter/直接写法的转换**（如 `url_for()` → 相对路径）

### 7.3 写入映射文件

将所有新发现的映射关系**追加**到 `references/hexo-to-gridea-mappings.md`。**不修改项目中其他任何文件。** 映射文件格式如下：

```markdown
# Hexo → Gridea Pro 变量映射积累

> 本文件由阶段七交叉比对生成（需用户确认排除范围），累积所有已确认的跨系统变量映射关系。
> 后续迁移时，阶段二优先查阅本文件作为先验知识；如有冲突，以 `template-variables.md` 为准。

---

## 来源：{theme-name}（迁移日期：{YYYY-MM-DD}）

### 排除的组件
- `templates/post.html`（用户标记为不参与比对）
- `templates/partials/comments.html`（实现不完整）

### 全局变量映射

| Hexo | Gridea | 发现位置（源 → 目标） | 备注 |
|-------|--------|----------------------|------|
| `config.title` | `config.siteName` | `layout.pug` → `base.html` | 字段名不同 |
| `config.url` | `config.domain` | `layout.pug` → `base.html` | 含协议头 |
| `config.subtitle` | `theme_config.subtitle` | `layout.pug` → `base.html` | 需在 customConfig 中声明 |
| `theme.xxx` | `theme_config.xxx` | - | 通用规则 |

### 文章变量映射

| Hexo | Gridea | 发现位置（源 → 目标） | 备注 |
|-------|--------|----------------------|------|
| `page.cover` | `post.feature` | `post.pug` → `post.html` | 字段名不同，可能为空 |
| `page.permalink` | `post.link` | `post-card.pug` → `post-card.html` | 字段名不同 |
| `page.date` | `post.dateFormat` | `post.pug` → `post.html` | **禁止 `|date` filter** |
| `page.excerpt` | `post.content|excerpt` | `post-card.pug` → `post-card.html` | 需 `|safe` |
| `page.top` | `post.isTop` | `post-card.pug` → `post-card.html` | 字段名不同 |
| `page.prev` | `post.prevPost` | `post.pug` → `post.html` | **方向相反** |
| `page.next` | `post.nextPost` | `post.pug` → `post.html` | **方向相反** |

### 标签变量映射

| Hexo | Gridea | 发现位置（源 → 目标） | 备注 |
|-------|--------|----------------------|------|
| `tag.length` | `tag.count` | `tag.pug` → `tag.html` | 字段名不同 |
| `tag.permalink` | `tag.link` | `tag.pug` → `tag.html` | 字段名不同 |

### 分页变量映射

| Hexo | Gridea | 发现位置（源 → 目标） | 备注 |
|-------|--------|----------------------|------|
| `page.prev` | `pagination.prevURL` | `pagination.pug` → `pagination.html` |   |
| `page.next` | `pagination.nextURL` | `pagination.pug` → `pagination.html` |   |
| `page.current` | `pagination.currentPage` | `pagination.pug` → `pagination.html` |   |
| `page.total` | `pagination.totalPages` | `pagination.pug` → `pagination.html` |   |

### Helper 函数映射

| Hexo Helper | Gridea 替代 | 发现位置 | 备注 |
|-------------|------------|---------|------|
| `url_for(path)` | 直接写相对路径 | `layout.pug` → `base.html` | 无等价 helper |
| `date_xml(date)` | `post.date` | `head.pug` → `head.html` | RFC3339 即合法 |
| `strip_html(str)` | `|striptags` | `post-card.pug` → `post-card.html` | Pongo2 filter |
| `truncate(str, len)` | `|truncatechars:N` | `post-card.pug` → `post-card.html` | 冒号语法 |
| `__('key')` | 硬编码中文 | `footer.pug` → `footer.html` | 无多语言机制 |
| `partial('path', data)` | `{% include "path" %}` | `layout.pug` → `base.html` | 不传参，用上下文 |

### 陷阱记录

| 陷阱 | 描述 | 发现位置 |
|------|------|---------|
| prev/next 方向反转 | Hexo 的 `page.prev` = 更早；Gridea 的 `post.prevPost` = 更新 | `post.pug` → `post.html` |
| date filter 不可用 | `post.date` 在 Pongo2 中是 RFC3339 字符串，`|date` 报错 | `post.pug` → `post.html` |
| archives 大写键 | `group.Year` / `group.Posts` 而非小写 | `archive.pug` → `archives.html` |

---

```

### 7.4 映射文件维护规则

#### 来源的可信度等级

映射来源分为两个等级，**写入时必须在来源块中标注**：

| 等级 | 来源 | 可信度 | 说明 |
|------|------|--------|------|
| **L1（高置信度）** | 模式 B（仅积累映射） | ★★★ | 人工确认过的迁移主题，源和目标都是已知代码，交叉比对结果可靠 |
| **L2（自动生成）** | 模式 A（完整迁移自动写入） | ★★☆ | AI 自动推导的映射，未经人工逐条复核，可能存在错误 |

L1 来源的标记格式：`## 来源：{theme-name}（迁移日期：{YYYY-MM-DD}）[L1-高置信度]`

L2 来源的标记格式：`## 来源：{theme-name}（迁移日期：{YYYY-MM-DD}）[L2-自动生成]`

#### 冲突解决规则

当同一 Hexo 变量在多个来源中映射到**不同**的 Gridea 变量时：

1. **L1 永远覆盖 L2。** 如果 L1 来源的映射与 L2 冲突，以 L1 为准，并在备注中标注"L1 覆盖 L2-{旧主题名}"
2. **L1 之间冲突时，以最新的为准。** 两个 L1 来源的映射冲突，以最近日期的为准，并标注"覆盖自 L1-{旧主题名}"
3. **L2 之间冲突时，以最新的为准。** 两个 L2 来源的映射冲突，以最近日期的为准，并标注"覆盖自 L2-{旧主题名}"
4. **L2 写入时若发现已有 L1 记录，不覆盖。** L2 自动生成的结果不能覆盖 L1 人工确认的结果。如果 L2 的映射与 L1 不同，仅在 L2 的备注中标注"与 L1-{主题名} 不一致，保留 L1"

#### 其他规则

5. **每次成功迁移追加一个"来源"块**，不覆盖已有内容
6. 通用规则（如 `theme.xxx` → `theme_config.xxx`）在首次出现时记录，后续迁移无需重复
7. 陷阱记录在首次发现时记录，后续迁移确认仍然存在时更新发现位置
8. 映射文件仅由阶段七写入，**不修改项目中其他任何文件**

#### 阶段二消费时的优先级

阶段二推导变量映射时，读取 `hexo-to-gridea-mappings.md` 的优先级：

1. **优先采用 L1 映射**（高置信度，可直接复用）
2. **L2 映射作为参考**（需与 `template-variables.md` 交叉验证，不盲信）
3. 如果同一变量有多个 L1 映射且冲突，**以最新日期的 L1 为准**

---

## 速查卡片：Pongo2 最容易踩的 5 个坑

> **每写一个模板时，先看这 5 条，写完后再自查一遍。**

| # | 坑 | 错误 | 正确 |
|---|-----|------|------|
| 1 | Filter 冒号 | `default("x")` | `default:"x"` |
| 2 | date filter | `post.date|date:"2006-01-02"` | `post.dateFormat` |
| 3 | not == 陷阱 | `not x == y` | `x != y` |
| 4 | 长度 | `.length` | `|length` |
| 5 | safe | `post.content` | `post.content|safe` |

---

## 附录 A：源主题组件模式 → Pongo2 转换策略

不同源引擎的组件复用机制各不相同，但目标统一为 Pongo2 的 `{% include %}` 模式（Pongo2 不支持 `macro`）。

### A.1 Pug Mixin → Pongo2

Pug 的 `mixin` 是带参数的函数式组件。转换策略：

**策略 1：转为 `include` 组件 + `set` 变量**

```pug
// 源: _mixins/post-card.pug
mixin postCard(post, index)
  .post-card
    h2= post.title
    time= post.date
```

```html
<!-- 目标: templates/partials/post-card.html -->
<!-- 在调用前通过 {% set %} 设置变量，然后在循环中 include -->
{% set post = currentPost %}
<article class="post-card">
  <h2>{{ post.title }}</h2>
  <time>{{ post.dateFormat }}</time>
</article>
```

```html
<!-- 调用处 -->
{% for post in posts %}
  {% include "partials/post-card.html" %}
{% endfor %}
```

**策略 2：直接内联** — 如果 mixin 逻辑简单（< 5 行），直接内联到调用处。

**策略 3：用 `if` 条件分支替代 `case`**

```pug
// 源: 用 case 区分页面类型
case page.type
  when 'tags'
    include includes/tags.pug
```

```html
<!-- 目标: 不同页面类型已经是独立模板文件，不需要 case -->
{% if page_type == "tags" %}
  ...
{% endif %}
```

### A.2 EJS Function/Include → Pongo2

EJS 的组件复用通过 `include()` 或内联 JS 函数实现。

**策略 1：`<% include('path') %>` → `{% include "path" %}`**

```ejs
<!-- 源: <% include('_partial/post-card') %> -->
<!-- 目标: -->
{% include "partials/post-card.html" %}
```

**策略 2：内联辅助函数 → `{% include %}`**

```ejs
<!-- 源: 内联函数在 forEach 中调用 -->
<% posts.forEach(function(post) { %>
  <div class="card">
    <h2><%= post.title %></h2>
  </div>
<% }); %>
```

```html
<!-- 目标: 抽成独立组件 + for 循环 -->
{% for post in posts %}
  {% include "partials/post-card.html" %}
{% endfor %}
```

### A.3 Swig / Nunjucks Macro → Pongo2

Swig 和 Nunjucks 的 `macro` 与 Pug mixin 类似，但用 `{% %}` 语法。

```swig
{# 源: Swig macro #}
{% macro postCard(post) %}
  <div class="card">
    <h2>{{ post.title }}</h2>
  </div>
{% endmacro %}
```

```html
<!-- 目标: Pongo2 partial + for 循环 -->
{% for post in posts %}
  {% include "partials/post-card.html" %}
{% endfor %}
```

> **注意**：Swig/Nunjucks 的 macro 被调用的上下文变量（如 `post`）在执行 `{% include %}` 时自动可用，**不需要**像 Pug mixin 那样通过 `{% set %}` 显式传递。这是因为 Swig/Nunjucks macro 内部已经使用 `{{ post.title }}` 访问外部变量，而 Pongo2 的 `include` 同样继承了父级上下文——两者行为一致。

## 附录 B：发布前最终检查清单

参照 `references/quality-checklist.md` 的 P0 级别逐项确认：

- [ ] `validate_syntax.py` 零错误零警告
- [ ] `render_test.py` 所有页面渲染成功
- [ ] 输出 HTML 无残留模板标签
- [ ] 0 篇文章边界情况不崩溃
- [ ] 无封面图文章不崩溃
- [ ] 无标签文章不崩溃
- [ ] 特殊字符标题正确转义
- [ ] 真机验证无 `fallback-banner`
- [ ] 暗色模式所有元素可读
- [ ] 375px 移动端无水平滚动
- [ ] 每页有唯一 `<title>`
- [ ] 语义化 HTML 标签正确使用
- [ ] 所有 `<img>` 有 `alt` 属性