---
name: feishu-clipper
description: "Clip web pages, articles, or videos into Feishu wiki with reading notes and original text/transcript. Use when user shares a URL and wants it saved to a Feishu knowledge base (wiki). Supports X/Twitter articles, general web pages, Chinese platforms, and YouTube/Bilibili video transcripts. Default save location is 'Area' wiki space; user can specify another space name. Triggers: user shares a URL and says 'clip', 'save to feishu', '写入飞书', '剪藏', '收藏到知识库', or wants to archive a web page or video into their wiki with notes."
---

# Feishu Clipper

将网页内容剪藏到飞书知识库，自动生成阅读笔记 + 原文两个 section。

## 配置

首次使用时读取 `config.yaml`（与本文件同目录）：

```yaml
# ~/.qoder-cn/skills/feishu-clipper/config.yaml
default_wiki_space: "Area"    # 默认保存的知识库（用户 prompt 可覆盖）
proxy: "http://127.0.0.1:7890"  # 代理地址
notes_language: "zh"           # 笔记语言
```

如果 config.yaml 不存在，使用上述默认值。用户 prompt 中指定的知识库名称优先于配置文件。

## 用户输入解析

从用户 prompt 中提取：

| 参数 | 必填 | 默认值 | 示例 |
|------|------|--------|------|
| URL | 是 | — | `https://x.com/andrewyng/status/...` |
| 目标知识库 | 否 | `Area` | "保存到 Project 知识库" |
| 文档标题 | 否 | 从页面提取 | "Andrew Ng: AI Engineering Skills Map" |
| 额外指令 | 否 | — | "重点分析第三部分" |

## 执行流程

### 1. 抓取内容（多策略重试）

按 URL 域名选择首选策略，失败时按重试链降级：

**X / Twitter 链接** (`x.com`, `twitter.com`)：

1. Twitter syndication API（无需认证，需代理）：
   ```bash
   curl -s --proxy http://127.0.0.1:7890 \
     "https://cdn.syndication.twimg.com/tweet-result?id=<TWEET_ID>&lang=en&token=x"
   ```
   从返回 JSON 提取 `text`、`article.preview_text`、`article.title`。
2. agent-reach twitter-cli（需 cookie + 代理）：
   ```bash
   # 先预热 client transaction
   TWITTER_PROXY="http://127.0.0.1:7890" twitter-warm-ct
   # 读取推文
   twitter tweet <URL> --yaml
   # 读取 X Article
   twitter article <URL> --yaml
   ```
3. 中文转载源（通过代理 curl）：搜索 `"关键标题关键词" site:163.com OR site:qq.com OR site:csdn.net`，用 `curl --proxy` 抓取全文。

**普通网页**：

1. `WebFetch` 工具直接抓取
2. `curl --proxy http://127.0.0.1:7890` + Python 提取正文
3. Jina Reader：`curl -s "https://r.jina.ai/<URL>"`

**通用降级**：WebSearch 搜索原文标题 + 关键词，从转载源获取完整内容。

**视频链接**（YouTube / Bilibili）— 提取字幕/转录作为原文：

检测到 `youtube.com`、`youtu.be`、`bilibili.com`、`b23.tv` 域名时，走视频转录流程而非普通网页抓取。

YouTube 字幕提取（按序重试，拿到内容即停）：
```bash
# 1. yt-dlp 下载字幕（需代理）
yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" \
  --skip-download -o "/tmp/%(id)s" "URL"
# 读取生成的 .vtt 文件，去除时间轴只保留文本

# 2. yt-dlp 失败时，用 agent-reach transcribe 兜底（下载音频 + Whisper 转写）
agent-reach transcribe "URL"
```

Bilibili 字幕提取：
```bash
# 1. 获取视频元数据（确认字幕可用性）
bili video BVxxx

# 2. 提取字幕（需 OpenCLI + 桌面 Chrome）
opencli bilibili subtitle BVxxx

# 3. 无字幕时，下载音频再转写
bili audio BVxxx
agent-reach transcribe /tmp/音频文件路径
```

转录文本处理：
- 去除时间轴标记，只保留纯文本
- 合并重复行（自动字幕常有行间重复）
- 按语义分段，每段不超过 2000 字（适配飞书文档写入限制）
- 在文档"原文"section 标注来源为视频转录，附视频链接和时长信息

### 2. 生成阅读笔记

基于抓取内容撰写笔记。如果是视频转录，额外关注视频特有的信息（演示、代码展示、屏幕操作等）。笔记包含：

- **背景与动机**：作者/UP主是谁、何时发布、为什么做这个内容
- **核心观点**：1-2 句话总结
- **要点详解**：按内容结构逐点展开分析（视频按时间线或主题分段）
- **个人思考**：对内容的独立评价和延伸思考

笔记用中文撰写，保持分析深度，不是简单复述。

### 3. 创建飞书文档

```bash
# 查找目标知识库 space_id
lark-cli wiki +space-list --as user --format json
# 从返回结果中按 name 匹配目标知识库

# 创建文档节点
lark-cli wiki +node-create \
  --space-id <SPACE_ID> \
  --title "<文档标题>" \
  --as user --format json
```

记录返回的 `obj_token`（即 doc_token）用于后续写入。

### 4. 写入文档内容

分多次 `append` 写入，避免单次内容过长：

```bash
# 第一批：阅读笔记
lark-cli docs +update --doc "<doc_token>" \
  --command append --as user \
  --content '<h1>阅读笔记</h1><h2>背景与动机</h2>...' \
  --format xml

# 第二批：笔记续
lark-cli docs +update --doc "<doc_token>" \
  --command append --as user \
  --content '<h2>要点详解</h2>...' \
  --format xml

# 分隔线
lark-cli docs +update --doc "<doc_token>" \
  --command append --as user \
  --content '<hr/>' \
  --format xml

# 第三批：原文
lark-cli docs +update --doc "<doc_token>" \
  --command append --as user \
  --content '<h1>原文</h1><callout emoji="📌">来源说明...</callout>...' \
  --format xml
```

XML 格式规范见 `lark-doc` skill 的 `lark-doc-xml.md`。关键标签：
- `<h1>` / `<h2>` / `<h3>`：标题层级
- `<p>`：段落（默认连贯段落，不要一切皆列表）
- `<b>`：加粗
- `<callout emoji="📌">`：来源信息框
- `<hr/>`：分隔线
- `<a href="...">`：链接

### 5. 输出结果

向用户报告：
- 文档链接（wiki URL）
- 内容摘要（笔记要点 + 原文来源）
- 抓取过程中遇到的问题（如有）

## 注意事项

- **代理**：抓取外网内容时使用 `--proxy http://127.0.0.1:7890`（用户可指定其他端口）
- **分段写入**：每次 `append` 控制在 2000 字以内，避免触发参数限制
- **原文标注**：如果无法获取逐字原文，明确标注"基于多个信源还原"
- **写作风格**：遵循 `lark-doc-style.md`——默认连贯段落，标题只给章节，列表只给并列项，高亮框克制使用
- **依赖 skill**：`lark-wiki`（创建节点）、`lark-doc`（写入内容）、`lark-shared`（认证）、`agent-reach`（抓取）
