# claude-skill-linkedin-post

**Turn curated Karakeep bookmarks into high-performing French LinkedIn posts — fully automated pipeline with human checkpoints.**

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg)](https://claude.ai/claude-code)
![Language: French](https://img.shields.io/badge/language-French-brightgreen.svg)
[![Dependency: humanize-fr](https://img.shields.io/badge/requires-humanize--fr-orange.svg)](https://github.com/hashbulla/claude-skill-humanize-fr)

---

## What It Does

A [Claude Code](https://claude.ai/claude-code) skill that runs a 6-phase pipeline transforming web bookmarks from [Karakeep](https://karakeep.app) into publication-ready LinkedIn posts in French. It processes bookmarks tagged `to-process`, clusters them by topic, generates 2-3 posts using proven copywriting frameworks, humanizes the output, and manages the full bookmark lifecycle.

```
Karakeep Bookmarks (tagged "to-process")
  → Phase 0: Environment Audit (API key, connectivity)
  → Phase 1: Ingest & Classify (fetch content, topic/signal tagging)
  → Phase 2: Plan Posts (count, frameworks, differentiation)
  → Phase 3: Generate Drafts (framework-strict, hook-last)
  → Phase 4: Humanize (/humanize-fr pipeline)
  → Phase 5: Finalize (save drafts, mutate bookmarks, log session)
```

Human approval gates at Phases 1, 2, 4, and 5.

## Quick Start

### Prerequisites

- [Claude Code CLI](https://claude.ai/claude-code)
- [Karakeep](https://karakeep.app) account with API key
- [humanize-fr skill](https://github.com/hashbulla/claude-skill-humanize-fr) (required dependency)
- Tavily MCP (optional — used for content extraction fallback)

### Installation

```bash
# 1. Install dependency
mkdir -p ~/.claude/skills/humanize-fr
curl -o ~/.claude/skills/humanize-fr/SKILL.md \
  https://raw.githubusercontent.com/hashbulla/claude-skill-humanize-fr/main/SKILL.md

# 2. Install this skill
mkdir -p ~/.claude/skills/linkedin-post
curl -o ~/.claude/skills/linkedin-post/SKILL.md \
  https://raw.githubusercontent.com/hashbulla/claude-skill-linkedin-post/main/SKILL.md

# 3. Set your API key (in your project's .env)
echo "KARAKEEP_API_KEY=your_key_here" >> .env
```

### First Run

```
/linkedin-post --dry-run
```

## Usage

| Command | Description |
|---------|-------------|
| `/linkedin-post` | Full pipeline — ingest, plan, generate, humanize, finalize |
| `/linkedin-post --dry-run` | Preview manifest and plan only — zero writes, zero mutations |
| `/linkedin-post --tag kubernetes` | Filter to bookmarks with an additional tag |

## Example Output

After running `/linkedin-post`, you get copy-paste-ready posts:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
POST 1 — PASTOR | AI | 2026-04-01 08:30 CET
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

J'ai automatisé 80% de ma veille tech.
Ça m'a pris 3 weekends. Et j'ai failli tout abandonner au 2ème.

Le problème, en fait, c'était pas l'outil.
C'était ma façon de trier l'information.

On scrolle, on bookmarke, on oublie.
J'avais 200+ signets non lus en janvier.

Alors j'ai construit un pipeline :
→ Karakeep pour capturer
→ Un tri par thème automatique
→ Et une synthèse hebdo en 3 posts

Résultat ? 4h de veille → 45 minutes.
Pas en sacrifiant la qualité — en supprimant le bruit.

Mais est-ce que le vrai gain, c'est le temps ?
Ou c'est la clarté de savoir exactement quoi partager ?

Si ta veille ressemble à un cimetière de bookmarks,
dis-moi en commentaire : c'est quoi ton plus gros frein ?

#IA #VeilleTech #Productivité #ContentStrategy

---
[PREMIER COMMENTAIRE]
🔗 Sources et ressources :
- How I Built My AI Reading Pipeline : https://example.com/article1
- Karakeep Documentation : https://docs.karakeep.app
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Draft saved to:
```
→ posts/drafts/2026-04-01-veille-tech-pipeline.md
```

## Pipeline Phases

### Phase 0 — Environment Audit

Verifies `KARAKEEP_API_KEY` is set, tests Karakeep API connectivity, checks Tavily MCP availability. Halts with diagnostic if any required check fails.

### Phase 1 — Bookmark Ingestion

Fetches non-archived bookmarks, filters to `to-process` tag, extracts content (Karakeep crawled HTML first, Tavily Extract as fallback). Classifies each by topic (AI / DevSecOps / Kubernetes / cross-cutting) and signal strength. Presents a manifest table for human pruning. Enforces a 60K-char content budget and 15-bookmark cap.

### Phase 2 — Post Planning

Decides post count (2-3) based on signal density and topic distribution. Assigns a copywriting framework per post. Enforces batch differentiation: no duplicate audiences, hook types, or angles. Presents plan for approval.

### Phase 3 — Content Generation

Writes one draft per planned post following the assigned framework. Hook written last (max 140 chars). Enforces 800-1,300 char target (1,500 ceiling, 1,800 for story-format). Includes `[PREMIER COMMENTAIRE]` block with source URLs. Runs forbidden pattern check.

### Phase 4 — Humanization

Invokes `/humanize-fr` on each draft. Presents humanized versions with diff summaries for review.

### Phase 5 — Finalization

Delivers copy-paste-ready posts. Saves drafts to `posts/drafts/` with YAML frontmatter. On approval, mutates Karakeep bookmarks (removes `to-process`, adds `processed`, adds to topic list) with per-bookmark error tracking. Logs session to `.claude/session-log.md`.

## Copywriting Frameworks

| Framework | When Used | Structure |
|-----------|----------|-----------|
| **PASTOR** | Multi-signal, complex topic | Problem → Amplify → Story → Transformation → Offer → Response |
| **PAS** | Single clear insight | Problem → Agitate → Solution |
| **Story-Question-Lesson** | Personal/terrain anecdote | Story → Reflective question → Actionable takeaway |
| **APP** | Digest of 3+ sources | Agree (shared pain) → Promise → Preview (key points) |

## Configuration

| Variable | Required | Description |
|----------|----------|-------------|
| `KARAKEEP_API_KEY` | Yes | API key in `.env` file (git-ignored) |
| Tavily MCP | No | User-scope MCP server for content extraction fallback |

### Karakeep Setup

- **Base URL**: `https://cloud.karakeep.app/api/v1`
- **Input tag**: `to-process` (attach to bookmarks you want processed)
- **Output tag**: `processed` (added automatically after processing)
- **Topic lists**: `AI`, `Kubernetes`, `DevSecOps` (created in Karakeep, resolved by name at runtime)

All tag and list IDs are resolved by name at runtime — never hardcoded.

## Output

- **Drafts**: `posts/drafts/YYYY-MM-DD-slug.md` with YAML frontmatter (topic, audience, framework, sources)
- **Session log**: `.claude/session-log.md` (YAML frontmatter, machine-readable)
- **Console**: Copy-paste-ready posts with first comment blocks

## Dependencies

| Dependency | Required | Link |
|-----------|----------|------|
| `/humanize-fr` | Yes | [hashbulla/claude-skill-humanize-fr](https://github.com/hashbulla/claude-skill-humanize-fr) |
| Karakeep API | Yes | [karakeep.app](https://karakeep.app) |
| Tavily MCP | No | Falls back to Karakeep crawled content |

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `KARAKEEP_API_KEY manquant` | Add key to `.env` in your project root |
| `Clé API invalide ou expirée` | Regenerate key in Karakeep settings |
| `Tag 'to-process' introuvable` | Create the tag in Karakeep and attach it to bookmarks |
| `Tavily MCP non disponible` | Non-blocking — pipeline uses Karakeep's crawled content instead |
| `Aucun bookmark avec le tag 'to-process'` | Tag at least one bookmark with `to-process` in Karakeep |
| Partial Karakeep mutation failures | Re-run pipeline — mutations are idempotent |
| Content budget exceeded (60K chars) | Pipeline caps extraction at 60K characters total. Remaining bookmarks are skipped. Run again with `--tag` to target a smaller subset |

## Contributing

Contributions are welcome — especially new copywriting frameworks, hook patterns, and Karakeep API edge cases.

1. Open an issue describing the change you'd like to make
2. Fork the repo and create a feature branch
3. Update SKILL.md with your changes
4. Submit a PR with a clear description of what changed and why

For framework additions: include the framework name, when to use it, its structure, and a sample post outline.

## License

[MIT](LICENSE)
