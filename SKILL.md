---
name: linkedin-post
description: >
  Karakeep-to-LinkedIn content pipeline. Ingests bookmarks tagged "to-process"
  from Karakeep, clusters them by topic, generates 2-3 LinkedIn posts in French
  using proven copywriting frameworks, humanizes the output, and manages bookmark
  lifecycle (tag/list mutations).

  Activate on ANY of these triggers:
  French: "génère des posts LinkedIn", "crée des posts depuis Karakeep",
  "linkedin post", "posts LinkedIn", "traite les bookmarks",
  "génère du contenu LinkedIn", "transforme les bookmarks en posts",
  "posts depuis mes signaux", "lance le pipeline LinkedIn"
  English: "generate LinkedIn posts", "create posts from Karakeep",
  "linkedin post", "process bookmarks", "run the LinkedIn pipeline",
  "turn bookmarks into posts"

  Do NOT activate for: writing a single post from a topic (use /write-post),
  reviewing an existing post (use /review-post), humanizing text (use /humanize-fr),
  deck generation, or any task not involving the Karakeep-to-LinkedIn pipeline.
user-invocable: true
disable-model-invocation: false
argument-hint: "[--dry-run] [--tag <tagname>]"
---

# LinkedIn Post Pipeline (Karakeep → LinkedIn)

You are an expert LinkedIn content strategist and copywriter operating a 6-phase
pipeline that transforms curated web bookmarks into high-performing French LinkedIn posts.

## Arguments

Parse from $ARGUMENTS:
- `--dry-run`: Run Phases 0-2 only. Zero writes, zero API mutations. Output manifest + plan, then exit.
- `--tag <tagname>`: Additional tag filter — only process bookmarks that also have this tag.

## Phase 0 — Environment Audit

**Goal**: Verify all dependencies before doing any work.

### 0.1 Load Karakeep API Key

```bash
# Try .env in current project, then global
if [ -f .env ]; then source .env; fi
echo "${KARAKEEP_API_KEY:0:8}..."
```

If `KARAKEEP_API_KEY` is empty or unset:
```
✦ KARAKEEP_API_KEY manquant.
  Ajoutez-le dans .env (projet) ou exportez-le dans votre shell.
  Voir : .claude/rules/security.md
```
→ **HALT**

### 0.2 Test Karakeep Connectivity

```bash
rtk proxy curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  "https://cloud.karakeep.app/api/v1/bookmarks?limit=1"
```

- `200` → proceed
- `401` → "Clé API invalide ou expirée." → **HALT**
- Other → "Karakeep injoignable (HTTP {code})." → **HALT**

### 0.3 Verify Tavily MCP

Check that `mcp__tavily__tavily_extract` is in available tools.
- Present → proceed
- Absent → warn "Tavily MCP non disponible — fallback sur contenu Karakeep uniquement." → continue (non-blocking)

---

## Phase 1 — Bookmark Ingestion

**Goal**: Fetch, filter, extract content, classify.

### 1.1 Resolve Tag & List IDs

Fetch all tags and lists to resolve names → IDs at runtime. **Never hardcode IDs.**

```bash
# Get all tags
rtk proxy curl -s -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  "https://cloud.karakeep.app/api/v1/tags"

# Get all lists
rtk proxy curl -s -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  "https://cloud.karakeep.app/api/v1/lists"
```

From the responses, resolve:
- `to-process` tag → `TO_PROCESS_TAG_ID`
- `processed` tag → `PROCESSED_TAG_ID`
- Lists: `AI`, `Kubernetes`, `DevSecOps` → their respective IDs

If `to-process` tag not found → "Tag 'to-process' introuvable dans Karakeep." → **HALT**
If `processed` tag not found → warn, continue (will skip tag mutation in Phase 5)

### 1.2 Fetch Bookmarks

```bash
rtk proxy curl -s -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  "https://cloud.karakeep.app/api/v1/bookmarks?limit=30&archived=false"
```

If response has `nextCursor`, fetch additional pages until all non-archived bookmarks are retrieved (up to 100 max).

### 1.3 Filter

From all fetched bookmarks:
1. Keep only those with `to-process` tag (match by tag ID)
2. If `--tag <tagname>` was provided: further filter to bookmarks also having that tag
3. If zero bookmarks match → "Aucun bookmark avec le tag 'to-process' trouvé." → **HALT**

### 1.4 Content Extraction

Process bookmarks in order, tracking a **60,000 character content budget**.

For each bookmark (max 15):
1. **Check budget**: If total extracted content ≥ 60,000 chars → stop fetching, log remaining as "skipped (budget)"
2. **Karakeep crawled content** (primary): If `crawlStatus === "success"` and `contentAssetId` exists:
   ```bash
   rtk proxy curl -s -H "Authorization: Bearer $KARAKEEP_API_KEY" \
     "https://cloud.karakeep.app/api/v1/assets/{contentAssetId}"
   ```
3. **Tavily Extract** (fallback): If Karakeep didn't crawl, or for GitHub URLs (README extraction):
   Use `mcp__tavily__tavily_extract` with the bookmark URL
4. **Neither available**: Mark as "content unavailable" — still include in manifest with URL and title only

### 1.5 Classification

For each bookmark with content, determine:
- **Topic**: `AI` | `DevSecOps` | `Kubernetes` | `cross-cutting`
- **Signal strength**: `strong` (clear actionable insight) | `moderate` (useful context) | `weak` (tangential)
- **Summary**: One-line description of the key insight

### 1.6 Present Manifest

Display a table to the user:

```
## Manifest — Bookmarks ingérés

| # | Titre | URL | Topic | Signal | Résumé |
|---|-------|-----|-------|--------|--------|
| 1 | ...   | ... | AI    | strong | ...    |

Total: N bookmarks | Budget utilisé: X/60,000 chars
```

Ask: **"Voulez-vous retirer des bookmarks avant de planifier les posts ? (numéros à exclure, ou Entrée pour continuer)"**

If `--dry-run`: continue to Phase 2, then exit after plan display.

---

## Phase 2 — Post Planning

**Goal**: Decide how many posts, assign frameworks, define differentiation.

### 2.1 Post Count Decision

- **2 posts** if: <8 strong/moderate signals OR topics are highly homogeneous
- **3 posts** if: clearly distinct clusters across ≥2 themes

### 2.2 Framework Assignment

Available frameworks (assign one per post):

| Framework | When to use | Structure |
|-----------|------------|-----------|
| **PASTOR** | Multi-signal, complex topic | Problem → Amplify → Story → Transformation → Offer → Response |
| **PAS** | Single clear insight | Problem → Agitate → Solution |
| **Story-Question-Lesson** | Personal/terrain anecdote available | Story → Reflective question → Actionable takeaway |
| **APP** | Digest of 3+ sources | Agree (shared pain) → Promise (what you'll deliver) → Preview (key points) |

### 2.3 Batch Differentiation Rules

Enforce ALL of these — no exceptions:
- No two posts target the same audience segment
- No two posts use the same hook type (priority: personal failure > specific number > contrarian > curiosity gap > story > question > bold claim)
- If posts share source bookmarks, each must extract a distinct angle
- Each post gets a recommended posting date: spaced ≥48h apart, Tue-Thu 8-10h or 12-14h CET

### 2.4 Present Plan

Display:

```
## Plan de publication

### Post 1
- **Topic cluster**: [topic]
- **Framework**: [PASTOR/PAS/Story-Question-Lesson/APP]
- **Hook direction**: [type + brief description]
- **Sources**: [bookmark #s from manifest]
- **Audience**: [specific persona]
- **Date recommandée**: [YYYY-MM-DD HH:MM CET]

### Post 2
[...]
```

Ask: **"Ce plan vous convient ? (oui / modifications souhaitées)"**

If `--dry-run`: display manifest + plan, then:
```
✦ Mode dry-run : aucune écriture effectuée. Pipeline terminé.
```
→ **EXIT**

---

## Phase 3 — Content Generation

**Goal**: Write one draft per planned post.

### 3.1 Drafting Rules

For each post, follow the assigned framework structure strictly.

**Length contract**:
- Target: 800–1,300 characters (post body, excluding hashtags and first comment)
- Hard ceiling: 1,500 characters for standard posts
- Exception: 1,800 characters ONLY for Story-Question-Lesson format
- Count characters after writing, before delivering

**Hook rules** (write the hook LAST, after the body):
- Maximum 140 characters
- Priority order for hook type (use the highest available that fits):
  1. Personal failure or vulnerability
  2. Specific number or statistic
  3. Contrarian take
  4. Curiosity gap
  5. Story opening
  6. Question
  7. Bold claim

**Formatting**:
- 1-2 sentence paragraphs maximum
- Generous line breaks (mobile-first)
- Emojis: 1-2 max, as visual anchors only (not decoration)
- No external links in post body

**Hashtags**:
- 3-5 hashtags at the end
- Mix broad (#LinkedIn, #IA) and niche (#RecrutementIA, #DevSecOps)

### 3.2 First Comment Block

Below each post, include:

```
[PREMIER COMMENTAIRE]
🔗 Sources et ressources :
- [Source title 1] : [URL]
- [Source title 2] : [URL]
```

### 3.3 Forbidden Pattern Check

Before outputting any draft, verify NONE of these appear:

- "Il est important de noter que"
- "Dans le paysage actuel"
- "En conclusion"
- "Il convient de souligner"
- "Force est de constater"
- "À l'ère du numérique"
- "N'hésitez pas à"
- "Cela étant dit"
- "Il est essentiel de"
- "En d'autres termes"
- Any sentence starting with "Il est" + adjective + "de"
- "paradigme", "synergie", "levier" (corporate jargon)
- "Like if you agree", "Tag someone" (engagement bait)
- "What do you think?" as sole CTA

If any are found → rewrite the offending passage before continuing.

---

## Phase 4 — Humanization

**Goal**: Make each draft sound authentically human.

### 4.1 Invoke Humanizer

For each draft, invoke the `/humanize-fr` skill with the post body text.

Pass `story-format` flag if the post uses Story-Question-Lesson framework (allows 1,800 char ceiling).

### 4.2 Review & Present

After humanization, present each post to the user with:
- The humanized version (ready to copy-paste)
- The diff summary from humanize-fr
- Character count
- Any verification flags

Ask: **"Posts prêts pour finalisation ? (oui / retouches souhaitées)"**

---

## Phase 5 — Finalization

**Goal**: Deliver copy-paste-ready posts, mutate Karakeep bookmarks, log session.

### 5.1 Final Output

For each post, display in copy-paste-ready format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POST [N] — [Framework] | [Topic] | [Date recommandée]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Post content — ready to paste into LinkedIn]

---
[PREMIER COMMENTAIRE]
🔗 Sources et ressources :
- [Source 1] : [URL]
- [Source 2] : [URL]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 5.2 Save Drafts

Save each post as a markdown file in the project's `posts/drafts/` directory:

File name: `YYYY-MM-DD-slug.md` (using the recommended posting date)

Include YAML frontmatter:
```yaml
---
topic: "[topic]"
audience: "[target persona]"
objective: "engagement"
cta_type: "[comment|share|dm|link]"
status: "draft"
scheduled: "YYYY-MM-DD HH:MM"
framework: "[PASTOR|PAS|Story-Question-Lesson|APP]"
sources:
  - title: "[bookmark title]"
    url: "[bookmark URL]"
    karakeep_id: "[bookmark ID]"
author: "AgenceIA"
---
```

### 5.3 Karakeep Bookmark Mutations

Ask: **"Marquer les bookmarks comme traités dans Karakeep ? (oui/non)"**

If yes, process each source bookmark **one at a time** with error tracking:

```bash
# 1. Remove "to-process" tag
rtk proxy curl -s -X DELETE \
  -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tags":[{"tagId":"TO_PROCESS_TAG_ID"}]}' \
  "https://cloud.karakeep.app/api/v1/bookmarks/{BOOKMARK_ID}/tags"

# 2. Add "processed" tag
rtk proxy curl -s -X POST \
  -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tags":[{"tagId":"PROCESSED_TAG_ID"}]}' \
  "https://cloud.karakeep.app/api/v1/bookmarks/{BOOKMARK_ID}/tags"

# 3. Add to topic list (AI, Kubernetes, or DevSecOps)
rtk proxy curl -s -X PUT \
  -H "Authorization: Bearer $KARAKEEP_API_KEY" \
  "https://cloud.karakeep.app/api/v1/lists/{TOPIC_LIST_ID}/bookmarks/{BOOKMARK_ID}"
```

**Error handling per bookmark**:
- If any mutation fails: log the bookmark ID and which operation failed, continue with remaining bookmarks
- After all mutations: report successes and failures
- Idempotency: adding an already-present tag or list membership is a no-op — safe to re-run

### 5.4 Session Log

Append an entry to `.claude/session-log.md` in the project directory:

```markdown
---
date: "YYYY-MM-DD HH:MM"
post_count: N
frameworks: ["PASTOR", "PAS"]
sources:
  - id: "bookmark_id"
    title: "Bookmark title"
    url: "https://..."
    topic: "AI"
  - id: "..."
    title: "..."
    url: "..."
    topic: "DevSecOps"
mutations:
  succeeded: N
  failed: N
  details: ["bookmark_id: tag removal failed", ...]
---

## Session [YYYY-MM-DD]

Generated [N] posts from [M] bookmarks.
Frameworks used: [list].
Topics covered: [list].
```

If `--dry-run` was active: do NOT write session log.

### 5.5 Completion

```
✦ Pipeline terminé.
  [N] posts générés → posts/drafts/
  [M] bookmarks traités dans Karakeep ([F] échecs le cas échéant)
  Session enregistrée → .claude/session-log.md
```

---

## Reference: Karakeep API

| Operation | Method | Endpoint | Body |
|-----------|--------|----------|------|
| List bookmarks | GET | `/bookmarks?limit=30&archived=false` | — |
| Get asset | GET | `/assets/{contentAssetId}` | — |
| List tags | GET | `/tags` | — |
| List lists | GET | `/lists` | — |
| Add tags | POST | `/bookmarks/{id}/tags` | `{"tags":[{"tagId":"..."}]}` |
| Remove tags | DELETE | `/bookmarks/{id}/tags` | `{"tags":[{"tagId":"..."}]}` |
| Add to list | PUT | `/lists/{listId}/bookmarks/{bookmarkId}` | — |
| Remove from list | DELETE | `/lists/{listId}/bookmarks/{bookmarkId}` | — |

Base URL: `https://cloud.karakeep.app/api/v1`
Auth: `Authorization: Bearer $KARAKEEP_API_KEY`
Pagination: cursor-based via `nextCursor` field

---

## Error Recovery

- If Phase 0 fails: diagnose and halt with clear message
- If content extraction fails for one bookmark: skip it, continue with others
- If humanization produces unexpected output: present raw draft, let user decide
- If Karakeep mutations partially fail: report per-bookmark status, user can re-run
- At any human checkpoint: user can abort, modify, or continue
