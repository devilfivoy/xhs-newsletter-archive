# 部署指南

## 方案A：GitHub Pages（推荐，企微/微信可直接打开）

### Token获取

GitHub token 存储在 workspace 的 git remote URL 中：

```bash
cd /home/node/.openclaw/workspace
TOKEN=$(git remote get-url origin | grep -oP 'ghp_[^@]+')
REPO="devilfivoy/xhs-newsletter-archive"
```

### 上传单个文件

```bash
FILE="path/to/index.html"
DEST="xiushi-ka-xinpin/index.html"
B64=$(base64 -w0 "$FILE")

# 1. 检查文件是否已存在（获取sha用于更新）
SHA=$(curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO/contents/$DEST" | \
  python3 -c "import sys,json; print(json.load(sys.stdin).get('sha',''))" 2>/dev/null)

# 2. 上传（如有sha则更新，无则创建）
if [ -n "$SHA" ] && [ "$SHA" != "" ]; then
    CODE=$(curl -s -o /dev/null -w "%{http_code}" -X PUT \
      -H "Authorization: token $TOKEN" \
      "https://api.github.com/repos/$REPO/contents/$DEST" \
      -d "{\"message\":\"update\",\"content\":\"$B64\",\"sha\":\"$SHA\"}")
else
    CODE=$(curl -s -o /dev/null -w "%{http_code}" -X PUT \
      -H "Authorization: token $TOKEN" \
      "https://api.github.com/repos/$REPO/contents/$DEST" \
      -d "{\"message\":\"add\",\"content\":\"$B64\"}")
fi
echo "→ HTTP $CODE"  # 200=更新成功，201=创建成功
```

### 批量上传图片

```bash
for img in images/*.jpg; do
    FNAME=$(basename "$img")
    DEST="xiushi-ka-xinpin/hancard/$FNAME"
    B64=$(base64 -w0 "$img")
    SIZE=$(stat -c%s "$img")
    
    SHA=$(curl -s -H "Authorization: token $TOKEN" \
      "https://api.github.com/repos/$REPO/contents/$DEST" | \
      python3 -c "import sys,json; print(json.load(sys.stdin).get('sha',''))" 2>/dev/null)
    
    if [ -n "$SHA" ] && [ "$SHA" != "" ]; then
        CODE=$(curl -s -o /dev/null -w "%{http_code}" -X PUT \
          -H "Authorization: token $TOKEN" \
          "https://api.github.com/repos/$REPO/contents/$DEST" \
          -d "{\"message\":\"update $FNAME\",\"content\":\"$B64\",\"sha\":\"$SHA\"}")
    else
        CODE=$(curl -s -o /dev/null -w "%{http_code}" -X PUT \
          -H "Authorization: token $TOKEN" \
          "https://api.github.com/repos/$REPO/contents/$DEST" \
          -d "{\"message\":\"add $FNAME\",\"content\":\"$B64\"}")
    fi
    echo "✅ $FNAME (${SIZE}B) → HTTP $CODE"
done
```

### 验证部署

```bash
# Pages部署通常需要1-2分钟
sleep 60
curl -s -o /dev/null -w "%{http_code}" \
  "https://devilfivoy.github.io/xhs-newsletter-archive/xiushi-ka-xinpin/index.html"
# 应返回 200
```

### 触发Pages重建（如状态errored）

```bash
curl -s -X POST -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO/pages/builds"
```

### 目录结构

```
xhs-newsletter-archive/
└── xiushi-ka-xinpin/
    ├── index.html           # 主页面（~60KB）
    └── hancard/             # 手卡图片（每张<100KB）
        ├── hancard_1.jpg
        ├── hancard_2.jpg
        └── ...
```

### ⚠️ 注意事项

1. **图片必须同仓库**：HTML引用外部CDN图片（如COS）在GitHub Pages上可能跨域失败
2. **Pages部署延迟**：上传后1-2分钟才生效，不要急着验证
3. **Pages errored**：仓库较大时首次构建可能失败，触发rebuild即可
4. **手卡图先压缩**：Pillow quality=75，resize到max 800px宽，单张<100KB
5. **git push不通**：环境中git push可能超时，改用REST API逐文件上传

---

## 方案B：腾讯云COS CDN（内网上传快，但企微可能打不开）

使用 `aibibp-cdn-upload` skill 上传 HTML + 图片资源。

**域名**：`picasso-private-1251524319.cos.ap-shanghai.myqcloud.com`

### 上传流程

1. 把HTML和图片放到一个目录
2. 调用 `aibibp-cdn-upload` 上传整个目录
3. 返回COS公网URL

### ⚠️ 注意

- COS链接公网可访问，但**企微中可能被拦截无法打开**
- 如需企微分享，必须用GitHub Pages

---

## 方案C：Cowork（内部平台）

使用 `cowork-publish` 发布：

**域名**：`cowork.xiaohongshu.com/f/{alias}/`

### 纯前端要求

1. 根目录有 `index.html`
2. 所有资源用**相对路径**引用
3. 禁止有 `install.sh`、`package.json`、`Dockerfile` 等
4. 不需要 transform 脚本

### ⚠️ 注意

- HTML中引用外部CDN图片时不受跨域限制
- `cowork_redeploy` 可快速更新代码（不换alias）
- Cowork链接通常只有内网可访问

---

## 方案选择速查

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 发给外部买手/商务 | GitHub Pages | 企微/微信可直接打开 |
| 内部快速分享 | COS CDN | 上传快 |
| 内部长期使用 | Cowork | 有版本管理 |
| 需要两个渠道都覆盖 | GitHub Pages + COS CDN | 互为备用 |
