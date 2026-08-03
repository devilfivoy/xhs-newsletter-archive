---
name: snack-catalog-html
description: "从Excel货盘表或腾讯文档生成可公网访问的产品展示HTML页面，自动抓取小红书商品主图、匹配产品手卡、提取核心关键词、清洗差异点文本，生成带分类筛选的卡片式展示页面，最终上传至GitHub Pages生成对外可分享链接。适用场景：行业新品提报、买手或商务选品推荐、商品货盘可视化展示。触发词：产品展示页、新品提报HTML、商品货盘可视化、选品推荐页面、生成产品展示链接、Excel转展示页面、商品展示卡片页。"
---

# Product Showcase Page

从Excel/腾讯文档商品货盘表生成可对外分享的产品展示HTML页面。

## 核心流程概览

```
Excel/腾讯文档 → 解析数据 → 抓取商品主图 → 匹配手卡 → 关键词提取 → 文本清洗 → 生成HTML → 部署上线
```

---

## 第一步：需求确认（必做）

在动手前，必须逐轮确认以下关键问题（用户说"95%理解"才开工）：

| 轮次 | 确认项 |
|------|--------|
| 第1轮 | 产品图来源（Excel嵌入/小红书抓取/用户提供）、页面访问方式（公网URL/本地文件）、数据筛选条件（哪些AM/哪些品） |
| 第2轮 | 手卡文件处理（加下载链接/嵌入展示/忽略）、是否按商家分组、信息展示优先级（图/名称/链接/差异点/ID）、页面标题 |
| 第3轮 | 差异点截断长度（建议100字+展开）、手卡文件是否可获取 |

**经验**：用户最初说"用链接嵌入封面"通常指从小红书商品页自动抓主图，不需要额外提供图片。

---

## 第二步：解析数据源

### 方案A：Excel文件（推荐）

```python
import openpyxl
wb = openpyxl.load_workbook('货盘表.xlsx', data_only=True)
ws = wb.active
```

**标准列结构**（按实际表头自适应）：

| 列 | 字段 | 必需 |
|----|------|------|
| A | 提报AM | 否（用于筛选） |
| B | 商家ID | 否（可隐藏） |
| C | 商家名称 | 是 |
| D | 商品名称 | 是 |
| E | 商品ID | 否（可隐藏） |
| F | 商品链接 | 是 |
| G | 商品图片 | 否（可从链接抓取） |
| H | 差异点介绍 | 是 |
| I | 货盘/手卡附件名 | 否 |

**提取嵌入图片**：
```python
from openpyxl.drawing.image import Image as XlImage
for img_obj in ws._images:
    row = img_obj.anchor._from.row  # 0-indexed，+2=Excel行号
    col = img_obj.anchor._from.col  # 0-indexed
    # col=6 → 新品图片列，col=8 → 手卡列
```

⚠️ **踩坑**：`row`是0-indexed，Excel行号=row+1（含表头时+2）。

### 方案B：腾讯文档

腾讯文档用canvas渲染，无法直接DOM提取数据。策略：
1. **浏览器截图逐页读取**（Ctrl+Home → 逐PageDown截图 → 从截图识别文字）
2. **API尝试**（成功率低）：
   ```
   GET https://docs.qq.com/dop-api/opendoc?id={DOC_ID}&normal=1&outformat=1&wb=1
   ```
   返回metadata但通常不含实际单元格数据
3. **最佳实践**：让用户直接粘贴数据或提供Excel导出版

⚠️ **踩坑**：腾讯文档tab=参数可能导致只显示部分数据，切换tab后数据可能只有2行（筛选视图）。

---

## 第三步：抓取小红书商品主图

通过浏览器打开商品详情页，等待JS渲染后用evaluate提取图片URL：

```javascript
() => {
  const imgs = document.querySelectorAll('img');
  const r = [];
  for (const img of imgs) {
    const src = img.src || '';
    if (src.includes('xhscdn.com') && (img.naturalWidth > 100 || src.includes('800'))) {
      r.push(src);
    }
  }
  return r.slice(0, 2);
}
```

**主图特征**：
- 域名：`mall-i2.xhscdn.com/arkgoods/` 或 `mall-i1.xhscdn.com/arkgoods/`
- 参数：`?imageView2/2/w/800/q/80/format/webp`
- 取第一张作为封面

**流程**：
1. `browser navigate` 到商品链接
2. `wait 3000ms` 等待JS渲染
3. `evaluate` 提取图片URL
4. 逐个商品循环

⚠️ **踩坑**：
- 页面偶尔白屏（JS未加载），需要 `open` 新tab重试
- 商品链接中的 `xsec_token` 参数可省略，只保留 `goods-detail/{ID}` 即可
- 某些商品图宽度不是800而是750或4179，用 `naturalWidth > 100` 而非固定值判断

---

## 第四步：处理产品手卡

### 手卡来源优先级

1. **用户直接提供图片URL**（最常见）→ 下载+压缩+上传
2. **Excel嵌入图片**（col=8/9的_images）→ 提取+压缩
3. **配套PPTX/PDF文件** → 需要libreoffice转换（环境通常没有）

### 手卡压缩（必做）

```python
from PIL import Image
img = Image.open(path)
if img.mode == 'RGBA':
    img = img.convert('RGB')
if img.width > 800:
    ratio = 800 / img.width
    img = img.resize((800, int(img.height * ratio)), Image.LANCZOS)
img.save(output, format='JPEG', quality=75, optimize=True)
```

**压缩效果**：通常 85-90% 压缩率（如 8张手卡 4.3MB → 464KB）。

### 手卡匹配逻辑

用户通常以"图片1：商品A，图片2：商品B"的形式提供对应关系。维护 `hancard_map` 字典：

```python
hancard_map = {
    '陈阿炳老卤鸭脖': 'hancard_1.jpg',
    '凤梨干': 'hancard_20.jpg',
    # ...
}
```

⚠️ **踩坑**：
- 用户多次修正手卡对应关系很常见（删掉某个、替换某个），需要逐条确认
- 手卡图来自不同来源（COS URL、用户上传图片、Excel嵌入），需统一处理
- PPTX文件中提取的是单个图片元素而非完整slide渲染，效果可能不好

---

## 第五步：关键词提取与文本清洗

### 关键词提取

每个商品提取3个核心关键词标签。用正则匹配行业卖点词：

```python
patterns = [
    r'高蛋白', r'0脂肪', r'0蔗糖', r'非油炸', r'独立包装',
    r'低温慢烘', r'无添加', r'无防腐剂', r'原切', r'手工',
    r'富硒', r'配料干净', r'酥脆', r'老醋', r'椰浆',
    r'无蔗糖', r'天然代糖', r'0色素', r'0添加剂',
    # 按行业持续扩充
]
```

⚠️ 避免把"产品介绍""主要成分"等无意义标题词混入关键词。

### 差异点文本清洗

统一为 `1. 2. 3.` 有序编号格式：

```python
import re
# 去除各类编号和特殊符号
line = re.sub(r'^[①②③④⑤⑥⑦⑧⑨⑩✅▪️√·•\-—]+\s*', '', line)
line = re.sub(r'^[\d]+[、\.．\)）]\s*', '', line)
line = re.sub(r'^\【[^】]*\】\s*', '', line)
# 重新编号
'\n'.join(f"{i+1}. {item}" for i, item in enumerate(items))
```

---

## 第六步：生成HTML页面

### 设计规范

- **主色调**：淡雅绿（`#a8d5a2 → #5ab55e` 渐变），可按用户要求换色
- **布局**：CSS Grid，`repeat(auto-fill, minmax(220px, 1fr))`
- **卡片**：圆角10px + 阴影 + hover上浮3px
- **产品图**：1:1正方形（`padding-top:100%`），object-fit:cover
- **差异点**：默认截断100字 + 点击展开/收起
- **手卡**：lazy loading + 点击放大
- **筛选**：sticky顶部筛选栏，JS切换 `data-cat` 属性
- **移动端**：2列布局，字号适配

### 筛选按钮JS

```javascript
function filterCards(cat, btn) {
  document.querySelectorAll('.filter-btn').forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
  document.querySelectorAll('.card').forEach(card => {
    card.classList.toggle('hidden', cat !== 'all' && card.dataset.cat !== cat);
  });
}
```

### 模板生成脚本

详见 `scripts/generate_showcase.py`，可命令行调用：

```bash
python generate_showcase.py \
  --excel 货盘表.xlsx \
  --title "卤味果干类新品推荐" \
  --output ./output \
  --categories '{"卤味":["鸭脖"], "坚果果干零食":["凤梨干"]}'
```

详细CSS规范见 `references/style-guide.md`。

---

## 第七步：部署发布

### 方案A：GitHub Pages（推荐，企微/微信可打开）

**前置**：GitHub token 存储在 git remote URL 中：
```bash
TOKEN=$(git remote get-url origin | grep -oP 'ghp_[^@]+')
```

**上传流程**：
```bash
# 1. 检查文件是否已存在（获取sha用于更新）
SHA=$(curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO/contents/$DEST" | \
  python3 -c "import sys,json; print(json.load(sys.stdin).get('sha',''))")

# 2. 上传（创建或更新）
B64=$(base64 -w0 "$FILE")
curl -s -X PUT -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO/contents/$DEST" \
  -d "{\"message\":\"deploy\",\"content\":\"$B64\",\"sha\":\"$SHA\"}"
```

**⚠️ 关键注意**：
- Pages部署延迟1-2分钟，上传后等待再验证
- **手卡图必须上传到同仓库**（如 `hancard/` 子目录），外部CDN图片跨域加载会失败
- 手卡图先压缩再上传（单张<100KB）
- Pages状态 `errored` 时，可通过API触发rebuild
- 仓库：`devilfivoy/xhs-newsletter-archive`，目录：`xiushi-ka-xinpin/`

### 方案B：腾讯云COS CDN（内网快，但企微可能打不开）

使用 `aibibp-cdn-upload` skill 上传：
- 域名：`picasso-private-1251524319.cos.ap-shanghai.myqcloud.com`
- 公网可访问，但企微中可能无法打开

### 方案C：Cowork（内部平台）

使用 `cowork-publish` 发布：
- 域名：`cowork.xiaohongshu.com/f/{alias}/`
- 需要符合Cowork纯前端规范（根目录 index.html + 相对路径资源）
- 注意：不能有 install.sh/package.json 等

### 方案选择

| 场景 | 推荐方案 |
|------|----------|
| 发给外部买手/商务（企微/微信分享） | GitHub Pages |
| 内部同事快速分享 | COS CDN 或 Cowork |
| 快速预览 | 本地HTTP server |

详见 `references/deploy-guide.md`。

---

## 迭代修改速查

用户最常见的修改需求及对应操作：

| 需求 | 操作 |
|------|------|
| 改标题/删emoji | 修改 `<h1>` 内容 |
| 改色调 | CSS中 `linear-gradient` + `.tag` 背景色 |
| 删除统计栏/副标题 | 删除 `.stats-bar` / `.subtitle` 相关HTML和CSS |
| 图片更大/更小 | `.img-wrapper` 的 `padding-top`（100%=正方形，125%=4:5竖图） |
| 文字更大/更小 | `.card-body h3` / `.diff-text` / `.tag` 的 `font-size` |
| 卡片更大/更小 | `grid-template-columns` 的 `minmax` 值 |
| 按钮位置（手卡上/下） | 调换HTML中 `.hancard-section` 和 `.btn` 的顺序 |
| 加/删手卡 | `hancard_map` + 上传/删除图片 |
| 加/改分类 | `category_map` + filter-bar按钮数量文字 |
| 新增商品 | 在最后一个card后添加新card HTML + 更新筛选计数 |
| 取消商家分组 | 去掉 `.shop-group` 包裹，card平铺 |
| 标题加宽 | `font-size` 增大 + `letter-spacing` 增大 |
| 标题全宽居中 | header不设 `max-width`，保持 `width:100%` |

---

## 踩坑记录

### 1. Excel嵌入图片row/col对应错误
`_images` 的 `anchor._from.row` 是0-indexed。Row=0对应Excel第1行（表头），Row=1对应第2行。务必 +1 或 +2 转换。

### 2. GitHub Pages引用外部CDN图片失败
HTML中引用 `qa-cos-*.cos.ap-shanghai.myqcloud.com` 的手卡图，在GitHub Pages上可能因跨域策略加载不出来。解决：把图片也传到同一个GitHub仓库。

### 3. COS链接企微打不开
`picasso-private-*.cos.ap-shanghai.myqcloud.com` 的链接在企微中可能被拦截。需用 GitHub Pages 作为备用分享渠道。

### 4. 腾讯文档canvas渲染无法DOM提取
腾讯文档用canvas渲染表格，document.querySelectorAll无法获取单元格内容。只能通过截图+OCR或让用户导出Excel。

### 5. PPTX提取的不是完整手卡
`python-pptx` 提取PPTX会拆成单个图片/文字元素，不是完整的slide渲染图。环境中没有libreoffice无法渲染。解决：让用户截图或提供图片。

### 6. 商品页偶尔白屏
小红书商品详情页有时JS未加载导致白屏，evaluate返回空数组。解决：重新open一个tab或多等几秒。

### 7. base64内联导致HTML过大
Excel嵌入图片转base64会导致HTML从几十KB膨胀到10MB+。解决：图片保存为独立文件，HTML用相对路径/CDN URL引用。

### 8. GitHub API token位置
token不在固定文件里，而是存在 git remote URL 中：
```bash
TOKEN=$(git remote get-url origin | grep -oP 'ghp_[^@]+')
```

---

## 文件结构

```
skills/product-showcase-page/
├── SKILL.md                          # 本文件：核心指引
├── scripts/
│   └── generate_showcase.py          # 模板生成脚本（可直接运行）
└── references/
    ├── style-guide.md                # 视觉设计规范（色彩/字号/卡片/响应式）
    └── deploy-guide.md               # 部署指南（GitHub Pages / COS CDN / Cowork）
```

## 实际案例

**休食KA行业新品提报**（2026年7-8月）：
- 24款商品 · 13卤味 · 11坚果果干零食
- 19张手卡
- GitHub Pages链接：`devilfivoy.github.io/xhs-newsletter-archive/xiushi-ka-xinpin/`
- 经历了12轮迭代：数据解析 → 视觉优化 → 手卡匹配修正 → 分类筛选 → 新品增补
