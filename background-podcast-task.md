# AI Daily Podcast — Background Task

You are running the AI Daily Podcast workflow as a background sub-agent.

## Step 1: Determine Date
Run: `TZ=Asia/Shanghai date +%Y-%m-%d` — store as YYYY-MM-DD.

## Step 2: Fetch Research
Run the `verysmallwoods-research` skill to fetch:
1) Past 24 hours of research entries (since_hours=24, min_importance=2, limit=50)
2) Past 24 hours of trends/news headlines (since_hours=24, limit=30)

## Step 3: Generate Podcast Script
Generate a continuous oral script (口播稿) in Chinese:
- No section headers, no bullet lists, no markdown tables in the body
- Aggregate BOTH research (importance≥2) AND trends for richer coverage
- Every topic must be a complete, self-contained story — weave research depth and trend news together
- No 网感词 (internet slang like 走起、包场了、最炸的、拐点)
- Just clean, natural spoken Chinese prose meant to be read aloud

## Step 4: Apply personal-writing-style — MANDATORY
Load the personal-writing-style skill, then load ALL reference files:

1. personal-writing-style/references/punctuation.md
2. personal-writing-style/references/article-structure.md  
3. personal-writing-style/references/voice-and-phrasing.md

Non-negotiable checks:
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

## Step 5: Save & Git Push
Save as: `01-ai-daily/podcast-YYYY-MM-DD.md`

Then:
```
cd /home/ubuntu/.openclaw/workspace/01-ai-daily
git add --all
git commit -m "podcast YYYY-MM-DD"
git push
```
