---
name: xhs-industry-newsletter
description: 生成小红书行业优质笔记周报（HTML 小报）并上传 CDN。适用场景：运营同学每周定期输出行业笔记精选周报，包含节气健康指南、平台核心玩法、优质笔记 TOP 榜、往期笔记归档等模块。触发词：生成周报、生成小报、输出本周笔记周报、更新行业小报、笔记周报。支持任意行业类目（滋补、美妆、母婴、食品等），用户提供数据后自动填充模板并发布。
---

# xhs-industry-newsletter · 行业优质笔记周报生成器

## 产出物

一个独立 HTML 页面，包含五个板块，上传公网 CDN 后返回链接。

## 模板与资产

- **HTML 模板**：`assets/newsletter-template.html`（含完整 CSS 样式，占位符用 `{{变量名}}` 标记）
- **存档索引**：`output/newsletter-archive.md`（每期发布后追加一行）

## 五个板块结构

| 板块编号 | 板块名称 | 数据来源 | 类目要求（强制） |
|----------|----------|----------|----------------|
| 01 | 当月节气健康指南 | 根据发布日期自动判断节气，结合行业特性生成养生/食补建议 | — |
| 02 | 小红书经营核心玩法 | 用户提供或沿用上期（笔记运营/直播运营/私域运营等） | — |
| 03 | 本周优质笔记 TOP 5（主类目） | 鹰眼平台数据（笔记ID、标题、封面图、互动数据） | **只能来源于：传统滋补营养品 / 精制中药材 / 茶** |
| 04 | 近期其他类目优质笔记参考 | 鹰眼平台数据（副类目笔记数据） | **只能来源于：休食（排除传统滋补品子类目）** |
| 05 | 往期优质笔记内容 | 自动从上期继承，并追加用户本期新增链接 | — |

### ⛔ 板块类目硬规定（最高优先级，不得违反）

**板块 03 — 传统滋补营养品 / 精制中药材 / 茶**
- 每条笔记的 `goodsFirstCategoryName` 必须匹配以下三类之一：
  - `传统滋补营养品`
  - `精制中药材`
  - `茶`（含茶叶、茶饮等子类目）
- 不符合以上类目的笔记，无论互动数据多高，**一律不得放入板块 03**

**板块 04 — 休食**
- 每条笔记的 `goodsIndustry` 必须 = `休食`
- 且 `goodsFirstCategoryName` **不能**是「传统滋补营养品」「精制中药材」「茶」（避免与板块 03 重复）
- 不符合以上条件的笔记，**一律不得放入板块 04**

**鹰眼 API 取数时，必须在请求参数中加入对应类目过滤条件，禁止手动从全量结果中随意挑选。**

## 生成流程

### 1. 收集数据

向用户确认以下信息（未提供时询问）：
- 行业类目（如：美食滋补）
- 数据时间范围（如：Apr 9–15）
- 周次（如：2026 第 16 周）
- 板块 03、04 的笔记数据（每条：笔记 ID / 标题 / 封面图 URL / 赞藏评数量 / 店铺名 / 标签）
- 本期新增往期归档链接（可选）
- 板块 02 玩法内容是否沿用上期（默认沿用）

### 2. 填充模板

复制 `assets/newsletter-template.html`，替换所有 `{{占位符}}`：

```
{{WEEK_LABEL}}     → 2026 第 16 周
{{DATE_RANGE}}     → Apr 9–15
{{INDUSTRY}}       → 美食滋补行业
{{JIEQI_NAME}}     → 谷雨 · 4月20日前后
{{JIEQI_DESC}}     → ...
{{NOTE_XX_*}}      → 各笔记字段
{{ARCHIVE_LINKS}}  → 往期归档链接 HTML 片段
```

节气判断规则（参考 `references/jieqi.md`）。

### 3. 往期归档板块处理

- 读取上期 HTML 文件中 `archive-grid` 内的所有 `<a>` 标签，完整复制
- 在末尾追加本期用户新增的链接（编号顺延）
- 更新 `sec-meta` 和 `archive-week-label` 中的总数

### 4. 输出文件命名

```
output/viz-{industry-slug}-weekly-YYYYMMDD.html
```

例：`output/viz-zibu-weekly-20260416.html`

### 5. 上传 CDN

```bash
cd /home/node/.openclaw/workspace
python3 skills/aibibp-cdn-upload/aibibp-cdn-upload/scripts/upload_to_cdn.py output/<文件名> --production
```

### 6. 更新存档索引

在 `output/newsletter-archive.md` 表格顶部插入新一行：

```markdown
| 2026 第XX周 | Apr X–X | 2026-04-XX | [查看](CDN链接) | `文件名.html` |
```

### 7. 推送结果

向用户发送 CDN 链接，格式：
```
✅ 第XX周小报已发布！
📎 https://fe-static.xhscdn.com/...
📁 本地存档：output/viz-xxx-weekly-YYYYMMDD.html
```

## 注意事项

### ⚠️ 封面图域名（必须严格遵守）
- **鹰眼 `discoveryLink` 返回的是 `ci.xiaohongshu.com`，这是内网域名，外部完全无法访问**
- **必须将所有封面图域名替换为 `sns-img-hw.xhscdn.com`（对外公网可访问）**
- 替换规则：
  ```
  http://ci.xiaohongshu.com/  →  https://sns-img-hw.xhscdn.com/
  https://ci.xiaohongshu.com/ →  https://sns-img-hw.xhscdn.com/
  ```
- 路径保持不变，只换域名即可（`spectrum/`、`notes_pre_post/`、`note_pre_post_uhdr/` 等子路径均适用）
- 生成 HTML 后上传前，必须执行检查：**文件中不能出现任何 `ci.xiaohongshu.com`**

### 其他规则
- 封面图加 `onerror` 兜底 emoji 占位符（防止极少数图片加载失败）
- 亮点分析中**不得透露 DGMV、GPM 等内部数据指标**（这是内部运营数据，不对外）
- 板块03/04**不可出现涉及明星、头部达人的笔记**（如某明星种草、某知名达人推荐）
- 链接卡片样式：4列网格，粉色系，悬停变深
- 每期小报链接永久有效（CDN 公网可分享）
- 定时任务已配置（每周三 10:30），自动生成时读取上期文件继承归档

### 封面图获取流程（必须从笔记客户端取，保证对外可见）

**背景**：鹰眼 API `discoveryLink` 返回的封面 key 有时不完整或已过期（返回 404）。**必须从小红书笔记客户端页面实时抓取当前有效封面**，确保对外可见。

**强制流程**：
1. 对每条笔记，用浏览器访问 `https://www.xiaohongshu.com/explore/{笔记ID}`
2. 执行 JS 取图片：
   ```js
   Array.from(document.querySelectorAll('img'))
     .map(i => i.src)
     .filter(s => s.includes('xhscdn') && !s.includes('avatar') && !s.includes('picasso'))
   ```
3. 取第一张非头像的图片 URL，提取 `sns-webpic-qc.xhscdn.com/.../{路径key}!nd_...` 中的 `{路径key}`
4. 拼接为：`https://sns-img-hw.xhscdn.com/{路径key}?imageView2/2/w/270/format/jpg`
5. 用 `curl -o /dev/null -w "%{http_code}"` 验证返回 200
   - 若 404，尝试加 `notes_pre_post/`、`note_pre_post_uhdr/`、`spectrum/` 前缀
6. **只有验证 200 的 URL 才能写入 HTML**

**禁止行为**：
- ❌ 不能直接用鹰眼 API 返回的 `discoveryLink`（不验证直接写入，是封面加载失败的根本原因）
- ❌ 不能使用 `ci.xiaohongshu.com` 域名（内网域名，对外不可访问）
- ❌ 不能使用 `sns-webpic-qc.xhscdn.com` 域名（带时间戳鉴权，过期后 403）

**已删除笔记检测**：
- 访问笔记页面时，若页面无图片（imgs 为空或仅有系统图片），说明笔记已被删除，**必须从候选池中选其他笔记替换，不得放入小报**
