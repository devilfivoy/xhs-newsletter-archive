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
- 周次（如：2026 第 16 周）
- 板块 03、04 的笔记数据（每条：笔记 ID / 标题 / 封面图 URL / 赞藏评数量 / 店铺名 / 标签）
- 本期新增往期归档链接（可选）
- 板块 02 玩法内容是否沿用上期（默认沿用）

**⚠️ 日期范围规则（必须遵守）**：
- 数据时间范围 = **从生成当天往前推 7 天**（不含今天，含第 7 天前）
- 例：5月7日生成 → Apr 30–May 7（即 Apr 30、May 1、2、3、4、5、6、7，共 8 天跨度）
- 计算方式：`生成日期 - 7 天` 到 `生成日期`
- 例：Apr 28 生成 → Apr 21–28

### 2. 填充模板

复制 `assets/newsletter-template.html`，替换所有 `{{占位符}}`：

```
{{WEEK_LABEL}}     → 2026 第 16 周
{{DATE_RANGE}}     → Apr 30–May 7（当天往前推7天）
{{INDUSTRY}}       → 美食滋补行业
{{JIEQI_NAME}}     → 谷雨 · 4月20日前后
{{JIEQI_DESC}}     → ...
{{NOTE_XX_*}}      → 各笔记字段
{{ARCHIVE_LINKS}}  → 往期归档链接 HTML 片段
```

节气判断规则（参考 `references/jieqi.md`）。

### 3. 往期归档板块处理（⚠️ 严格按顺序执行）

- **Step 3a：读取上期 HTML** 中 `archive-grid` 内的所有 `<a>` 标签
- **Step 3b：查找上期小报 CDN 链接** — 从存档索引 `output/newsletter-archive.md` 中获取最近一期的 CDN 链接
- **Step 3c：插入最新链接** — 将上期小报 CDN 链接作为**编号 01**，插入到 `archive-grid` 的**最前面**
- **Step 3d：追加其余归档** — 将 Step 3a 读取的所有 `<a>` 标签原样追加到 Step 3c 之后（编号顺延）
- **Step 3e：更新计数** — 更新 `sec-meta` 和 `archive-week-label` 中的总数
- **核心原则：每期新生成的小报链接必须放在第 05 部分的最前面（编号 01），旧的自动后移顺延**

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

### 封面图获取流程（⚠️ 必须严格遵守，历史上反复出错）

**核心结论（2026-05-07 验证）**：
- `sns-webpic-qc.xhscdn.com` URL 含时间戳鉴权，**对外必然 404，绝对不能用**
- 正确方式：用 SSO Cookie 抓取笔记详情页 HTML，从 `imageList` 中提取 fileId，**去掉时间戳hash前缀**，直接拼 `sns-img-hw.xhscdn.com/{fileId}` 并 curl 验证

**强制流程（curl 命令行方式，无需浏览器）**：

```bash
# Step 1: 用 SSO Cookie 抓笔记详情页 HTML
TOKEN=$(jq -r '."common-internal-access-token-prod"' /home/node/.token/sso_token.json)
curl -s "https://www.xiaohongshu.com/discovery/item/{笔记ID}" \
  -H "Cookie: common-internal-access-token-prod=${TOKEN}" \
  -H "User-Agent: Mozilla/5.0" > /tmp/note.html

# Step 2: 从 HTML 中提取 urlPre 字段（含 Unicode 转义）
# 示例原始值: http:\\u002F\\u002Fsns-webpic-qc.xhscdn.com\\u002F202605071140\\u002F{hash}\\u002F{fileId}!nd_xxx
# 格式规律：时间戳/hash/[子目录/]fileId
grep -o '"urlPre":"[^"]*"' /tmp/note.html | head -1

# Step 3: 提取 fileId（去掉时间戳 hash，去掉 !nd_xxx 后缀）
# 时间戳格式: 14位数字，例如 202605071140
# hash 格式: 32位十六进制字符串
# fileId 格式: 1040xxx 开头，可能带 notes_pre_post/、spectrum/、ark_notes/ 子目录前缀
python3 -c "
import re, json
raw = open('/tmp/note.html').read()
m = re.search(r'\"urlPre\":\"(http[^\"]+)\"', raw)
if m:
    url = m.group(1).encode().decode('unicode_escape')
    # 提取 fileId：去掉 时间戳hash/ 前缀，去掉 !nd_xxx 后缀
    m2 = re.search(r'xhscdn\.com/\d+/[0-9a-f]+/(.+?)!', url)
    if not m2:
        m2 = re.search(r'xhscdn\.com/\d+/(.+?)!', url)
    if m2:
        fileid = m2.group(1)
        print(f'fileId: {fileid}')
        print(f'封面URL: https://sns-img-hw.xhscdn.com/{fileid}?imageView2/2/w/270/format/jpg')
"

# Step 4: curl 验证（必须 200 才能用）
curl -s -o /dev/null -w "%{http_code}" \
  "https://sns-img-hw.xhscdn.com/{fileId}?imageView2/2/w/270/format/jpg"
```

**子目录规律**（fileId 可能含前缀，原样保留）：
- 无前缀：`1040gxxx...`（最常见）
- `notes_pre_post/1040g3kxxx...`
- `spectrum/1040g0kxxx...`
- `ark_notes/104104eoxxx...`

**最终写入 HTML 的 URL 格式**：
```
https://sns-img-hw.xhscdn.com/{fileId}?imageView2/2/w/270/format/jpg
```

**三条铁律（绝对不能违反）**：
- ❌ `sns-webpic-qc.xhscdn.com` — 含时间戳鉴权，对外 404，**禁止使用**
- ❌ `ci.xiaohongshu.com` — 内网域名，对外不可访问，**禁止使用**
- ❌ 未经 curl 验证 200 就写入 HTML — **必须先验证**

**已删除笔记检测**：
- 若抓取到的 HTML 中无 `imageList` 或 `urlPre` 字段，说明笔记已删除/私密，**必须换其他笔记**

**板块05 往期优质笔记存档规则（⚠️ 必须遵守）**：
- 只能放：往期优质笔记**文档链接**（微信文档 doc.weixin.qq.com）和历次生成的**周报小报 CDN 链接**
- **绝对不能放**：单条笔记的 xiaohongshu.com/explore/ 链接
- 每期生成后，在存档中新增一行指向本期小报 CDN 链接，编号顺延
