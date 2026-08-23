# 内容抓取策略详解

## X / Twitter

### syndication API（首选，无需认证）

```bash
# 提取 tweet ID（URL 中最后的数字串）
# https://x.com/user/status/1234567890 → ID = 1234567890

curl -s --proxy http://127.0.0.1:7890 \
  "https://cdn.syndication.twimg.com/tweet-result?id=<TWEET_ID>&lang=en&token=x" \
  -H "User-Agent: Mozilla/5.0"
```

返回 JSON 包含：
- `text`：推文正文
- `article.title`：文章标题（如有）
- `article.preview_text`：文章预览（截断）
- `user.name` / `user.screen_name`：作者信息
- `favorite_count` / `conversation_count`：互动数据

局限：article 正文被截断，只有 preview_text。需要全文时走下方策略。

### 中文转载源（获取完整翻译全文）

很多 X Article 会被中文平台全文转载。搜索策略：

```bash
# 用文章标题 + 作者名搜索
WebSearch: "<文章标题>" "<作者名>" 全文
WebSearch: "<关键短语>" site:163.com OR site:qq.com OR site:csdn.net
```

抓取时通过代理：
```bash
curl -s --proxy http://127.0.0.1:7890 -L "<URL>" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36"
```

用 Python 提取正文：
```python
import sys, re, html
content = sys.stdin.read()
body = re.findall(r'<div[^>]*class="[^"]*article[^"]*"[^>]*>(.*?)</div>', content, re.DOTALL)
for b in body:
    text = re.sub(r'<br\s*/?>', '\n', b)
    text = re.sub(r'<p[^>]*>', '\n', text)
    text = re.sub(r'<[^>]+>', '', text)
    text = html.unescape(text).strip()
    if len(text) > 200:
        print(text)
```

高频转载源：
- `m.163.com/dy/article/` — 网易号
- `new.qq.com/rain/a/` — 腾讯新闻
- `blog.csdn.net/` — CSDN 博客
- `m.toutiao.com/article/` — 头条号
- `view.inews.qq.com/wxn/` — 腾讯新闻（新版）

## 普通网页

### WebFetch 工具

直接调用 WebFetch，适合大多数非 SPA 页面。

### curl + 代理

```bash
curl -s --proxy http://127.0.0.1:7890 -L "<URL>" \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36"
```

### Jina Reader

```bash
curl -s "https://r.jina.ai/<URL>" -H "Accept: text/plain"
```

注意：Jina 对 x.com 有滥用封禁，Twitter 链接不要用此方式。

## 代理检测

抓取前验证代理可用：
```bash
curl -s --proxy http://127.0.0.1:7890 "https://httpbin.org/ip"
```
返回 JSON 含 `origin` 字段即代理正常。连接拒绝说明代理未启动。
