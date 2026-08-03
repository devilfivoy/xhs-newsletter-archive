# Product Showcase Page

> 从Excel/腾讯文档商品货盘表，一键生成可对外分享的产品展示HTML页面

## 效果展示

![示例页面](https://devilfivoy.github.io/xhs-newsletter-archive/xiushi-ka-xinpin/index.html)

**在线预览**：https://devilfivoy.github.io/xhs-newsletter-archive/xiushi-ka-xinpin/

## 功能特性

- 🖼️ **自动抓取商品主图**：通过浏览器打开小红书商品详情页，JS自动提取CDN图片URL
- 📋 **产品手卡展示**：支持多来源手卡图片（用户提供/Excel嵌入/PPTX提取），自动压缩（85-90%压缩率）
- 🏷️ **智能关键词提取**：正则匹配行业卖点词，每款商品自动提取3个核心标签
- 📝 **文本统一清洗**：杂乱编号（①②③、1)、一、等）统一为 `1. 2. 3.` 有序格式
- 🔍 **分类筛选**：顶部sticky筛选栏，一键切换品类（如卤味/坚果果干零食）
- 📱 **响应式设计**：PC端4-5列网格，移动端自适应2列
- 🎨 **淡雅绿色调**：清爽专业，适合商务场景分享
- 🔗 **一键部署**：支持GitHub Pages / 腾讯云COS CDN / Cowork三种发布方式

## 适用场景

| 场景 | 说明 |
|------|------|
| 行业新品提报 | AM给买手/商务推荐新品时的可视化展示页 |
| 选品推荐 | 品类筛选+产品手卡+商品链接，一站式选品参考 |
| 商品货盘可视化 | 把Excel表格变成图文并茂的在线展示页 |
| 买手商务沟通 | 直接转发链接，企微/微信可打开 |

## 快速开始

### 1. 准备数据

Excel文件需包含以下列：

| 列 | 字段 | 必需 |
|----|------|------|
| C | 商家名称 | ✅ |
| D | 商品名称 | ✅ |
| F | 商品链接 | ✅ |
| H | 差异点介绍 | ✅ |

### 2. 生成页面

```bash
python scripts/generate_showcase.py \
  --excel 货盘表.xlsx \
  --title "卤味果干类新品推荐" \
  --output ./output
```

### 3. 部署上线

推荐使用GitHub Pages（企微/微信可直接打开）：

```bash
# 从git remote获取token
TOKEN=$(git remote get-url origin | grep -oP 'ghp_[^@]+')
# 上传到GitHub
B64=$(base64 -w0 output/index.html)
curl -X PUT -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/USER/REPO/contents/path/index.html" \
  -d "{\"message\":\"deploy\",\"content\":\"$B64\"}"
```

## 文件结构

```
product-showcase-page/
├── README.md                         # 本文件
├── SKILL.md                          # OpenClaw Skill指引（完整流程+踩坑记录）
├── scripts/
│   └── generate_showcase.py          # 模板生成脚本（CLI可直接运行）
└── references/
    ├── style-guide.md                # 视觉设计规范（色彩/字号/卡片/响应式）
    └── deploy-guide.md               # 部署指南（GitHub Pages/COS CDN/Cowork）
```

## 技术栈

- **数据解析**：openpyxl（Excel）、浏览器截图+OCR（腾讯文档）
- **图片抓取**：Playwright/浏览器 evaluate + 小红书商品页JS渲染
- **图片压缩**：Pillow（JPEG quality=75, max 800px宽）
- **前端**：纯HTML/CSS/JS，无框架依赖
- **部署**：GitHub REST API / aibibp-cdn-upload / cowork-publish

## 实际案例

**休食KA行业新品提报**（2026年7-8月）：
- 📦 24款商品 · 2大品类（卤味13 + 坚果果干零食11）
- 📸 19张产品手卡
- 🔄 12轮迭代优化
- 🏪 涵盖陈阿炳、无趣的店、隐谷野、臻味、蜀西奇胜、水一方、三关六码头、玄小食、失重森林、食味的初相等10家商家

## 迭代记录

| 版本 | 改动 |
|------|------|
| v1 | 基础版：按商家分组，红色头部 |
| v2 | 取消分组平铺，渐变头部+卡片布局 |
| v3 | 淡雅绿色调，卡片紧凑化，文字编号统一 |
| v4 | 删emoji，图片放大，文字缩小 |
| v5 | 手卡匹配修正，新增分类筛选（卤味/坚果果干零食） |
| v6 | 手卡补全（从用户图片+PPTX+PDF多源匹配） |
| v7 | 手卡图片压缩（4.3MB→464KB），加速加载 |
| v8 | 标题头部全宽+居中，字号字距优化 |
| v9 | 新增6款商品（失重森林+食味的初相），总计24款 |

## 依赖

```
openpyxl>=3.1.0
Pillow>=9.0.0
```

## License

Internal use only.
