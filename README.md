# feishu-clipper

A Qoder CLI skill that clips web pages and articles into Feishu (Lark) wiki with auto-generated reading notes and original text.

## Features

- **Multi-strategy content fetching**: Twitter/X syndication API, agent-reach, Jina Reader, proxy curl, Chinese repost sources
- **Auto reading notes**: Generates structured reading notes with background, key points, analysis, and personal insights
- **Feishu wiki integration**: Creates documents directly in your Feishu knowledge base via lark-cli
- **Configurable**: Default wiki space, proxy, and language via `config.yaml`

## Installation

```bash
# Standard skill install (works with qodercn and compatible agents)
skills install https://github.com/bailynlove/feishu-clipper-skill

# Or link from local path (for development)
skills link /path/to/feishu-clipper-skill
```

## Configuration

Copy and edit the config file:

```bash
cp config.yaml.example config.yaml
# Edit ~/.qoder-cn/skills/feishu-clipper/config.yaml
```

```yaml
# Default wiki space name (overridable per invocation)
default_wiki_space: "Area"

# Proxy for fetching external content
proxy: "http://127.0.0.1:7890"

# Notes language
notes_language: "zh"
```

## Usage

Give the agent a URL and optionally specify a target wiki space:

```
https://x.com/andrewyng/status/2090840747738374568
```

```
把这个剪藏到 Project 知识库
```

```
Clip this to my Feishu wiki: https://example.com/article
```

### Document Structure

The generated document contains two sections:

1. **Reading Notes** (top) - Background, core insights, detailed analysis, personal reflections
2. **Original Text** (bottom) - Full article text with source attribution

## Dependencies

- [lark-cli](https://github.com/nicholaschenai/lark-cli) - Feishu/Lark CLI tool
- [agent-reach](https://github.com/Panniantong/agent-reach) - Internet access for Qoder CLI (optional, for Twitter)
- Qoder CLI with `lark-wiki`, `lark-doc`, `lark-shared` skills installed

## License

MIT
