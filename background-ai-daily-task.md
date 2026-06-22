# AI Daily — Background Task

You are running the AI Daily workflow as a background sub-agent. Carry out ALL steps below.

## Step 1: Date
Run: `TZ=Asia/Shanghai date +%Y-%m-%d` — store as YYYY-MM-DD.
Also compute MMDD (month+day, e.g. 0519) and a 1-2 Chinese word topic keyword summarizing the main theme.

## Step 2: Research
Run the `verysmallwoods-research` skill to fetch the past 24 hours of research entries (since_hours=24, min_importance=3, limit=50). For every entry with importance 5, WebFetch the url_original to get full article text.

## Step 3: Cover Image
Read the cover-image skill at `/home/ubuntu/.openclaw/workspace/01coder-agent-skills/skills/cover-image/SKILL.md` for full instructions.

Use the **AI Daily Series** style template (hand-drawn editorial ink, warm off-white paper #F7F3EA).

IMPORTANT: Do NOT use the `Prefer: wait` header. Use async creation + polling instead, to avoid curl timeout losing the prediction ID.

### 3a. Create prediction (async — no Prefer: wait)
Extract prediction ID from the response immediately.

### 3b. Poll until complete (max 15 retries, sleep 10s between each)

If cover was not saved, do NOT include the `![cover]` line in the article.

## Step 4: Write Article
Write an AI Daily blog post in Chinese as a markdown file. Save to `/home/ubuntu/.openclaw/workspace/01-ai-daily/ai-daily-YYYY-MM-DD.md`

Title format in frontmatter: `"【AI早读 MMDD】topic"` — where topic is ~10-15 Chinese characters summarizing the day's main theme. Do NOT use `"AI Daily -"` as part of the title.

Before the first section, insert the cover image using local path:
```
![cover](images/ai-daily-YYYY-MM-DD.jpg)
```

Each section must include a reference link to the source article right after the first paragraph:
```
链接：[Article Title](url_original)
```

## Step 5: Apply personal-writing-style — MANDATORY
Load the personal-writing-style skill, then load ALL of these reference files and apply them completely:

1. personal-writing-style/references/punctuation.md
2. personal-writing-style/references/article-structure.md
3. personal-writing-style/references/voice-and-phrasing.md

Non-negotiable checks before saving:
- Quotes: use " and ". No straight quotes " around Chinese prose.
- Dash: use space-hyphen-space ( - ), never --, ——, or —.
- Ellipsis: use ......, never … or ...
- Chinese sentence punctuation: ，。：；？！、 not ASCII ,.:;?!
- No H1 (frontmatter has title)
- No summary / conclusion sections
- No translation-ese or net-slang
- No business cliches: 闭环、抓手、颗粒度、对齐、赋能、赛道、弯道超车、心智
- No trust-me lines
- Let facts carry the point

## Step 6: Frontmatter + Footer
Frontmatter should have: title (as described above), date, and tags.

Footer at the end: a horizontal rule, then a line in italics:
```
*来源：VerySmallWoods Research Feed - YYYY-MM-DD UTC*
```

No other footer or content should expose the underlying API worker URL.

## Step 7: Write WeChat Version
Generate a WeChat Official Account (微信公众号) version of the same article. Save to `/home/ubuntu/.openclaw/workspace/01-ai-daily/wechat-YYYY-MM-DD.md`

Differences from the markdown version:
- **No markdown links** — WeChat does not support external hyperlinks. Write URLs as plain text on their own line, e.g.:
  ```
  原文链接：https://example.com/article
  ```
- **No frontmatter** — WeChat won't use YAML frontmatter. Title goes as an H1 (`# `) at the top.
- **Cover image** — Include as a blockquote line with the relative path: `> 封面图：images/ai-daily-YYYY-MM-DD.jpg`
- Content is otherwise the same text (same sections, same voice, same punctuation rules from Step 5).

## Step 8: Git Push
```
cd /home/ubuntu/.openclaw/workspace/01-ai-daily
git add --all
git commit -m "ai daily + wechat YYYY-MM-DD"
git push
```
