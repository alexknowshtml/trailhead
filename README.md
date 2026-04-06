# Trailhead

> **Work in progress.** Trailhead is actively being developed and tested. The protocol, format, and behavior are all subject to change. Use it, break it, share feedback -- but know that things will shift as we learn what works.

Your agent invokes a skill it hasn't touched in two weeks. Or one you were using yesterday. Either way, it has no idea what happened in prior sessions -- unless it reads full transcripts.

Trailhead turns [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills into bridges between sessions. Each skill maintains a lightweight index of every session where it contributed to the work. When the skill activates, the agent instantly sees what other sessions used it, how deeply, and what they accomplished -- rebuilding a knowledge graph in seconds without transcript reads.

Skills become nodes. Sessions become edges. The agent follows the links and reconstructs the full arc of a multi-session project from the skill alone.

## What it looks like

A JSON index that lives alongside your skill, updated automatically:

```json
[
  {
    "date": "2026-04-05",
    "sessionId": "discord-1490402407228641342",
    "confidence": "high",
    "lastTouch": "2026-04-05T20:00:00-04:00",
    "summary": "Added 18 new relationship edges + PIDC org actor. DB now 56 actors, 111 relationships. Built layer toggle buttons for graph readability. Ran betweenness centrality analysis -- Parker highest bridge score (0.224). Added centrality-scaled node radii and bridge scores in tooltips."
  },
  {
    "date": "2026-04-04",
    "sessionId": "e68c21a9",
    "confidence": "high",
    "lastTouch": "2026-04-04T18:00:00-04:00",
    "summary": "37 staff LinkedIn research via 6 parallel agents grouped by office. Surfaced 16 cross-faction connections. Established agent grouping pattern (by office, not alphabet)."
  },
  {
    "date": "2026-02-24",
    "sessionId": "fcbbd2b8",
    "confidence": "medium",
    "lastTouch": "2026-02-24T17:00:00-05:00",
    "summary": "Reviewed transition memo and calculator update scope. No sheet changes."
  },
  {
    "date": "2026-01-21",
    "sessionId": "c3df8fe7",
    "confidence": "high",
    "lastTouch": "2026-01-21T18:00:00-05:00",
    "summary": "Expanded calculator to 10 test scenarios. Refactored calculator.js. Shock year logic finalized."
  },
  {
    "date": "2026-01-15",
    "sessionId": "be046943",
    "confidence": "high",
    "lastTouch": "2026-01-15T19:00:00-05:00",
    "summary": "Initial deep dive into BIRT/NPT model. Built the first interactive calculator."
  }
]
```

These are from a real project skill tracking a multi-month policy analysis. Five entries tell you:
- The project started as a calculator build in January
- It evolved into a full relationship-mapping system by April
- One session was a light review with no changes (medium confidence)
- The April sessions ran across both CLI and Discord (mixed session ID formats)
- A single session coordinated 6 parallel research agents

An agent activating this skill for the first time reads 16 session entries and knows the full project arc in seconds -- no transcript reads needed.

## What emerges

After a few sessions, Trailhead indexes start showing patterns that no individual session captures:

**Session chains.** Work that builds across sessions becomes visible. A calculator built on Jan 15, bug-fixed on Jan 17, expanded on Jan 21, then restructured on Apr 1 -- each summary referencing discoveries from the one before. The agent sees the progression and picks up where the chain left off.

**Depth signals.** A `lastTouch` three hours after activation was a deep working session. One where `lastTouch` equals the activation time was a drive-by. When deciding which past session to recover full context from, that signal matters more than the date.

**Cross-surface continuity.** Session IDs come from CLI sessions (`227dd247`), Discord threads (`discord-1490402407228641342`), or any other surface that invokes the skill. The index doesn't care where work happened -- it tracks what the skill contributed.

**Coordination memory.** Summaries like "37 staff research via 6 parallel agents grouped by office" capture multi-agent patterns that aren't recorded anywhere else. The next session knows to use the same approach.

## Why each field matters

**`confidence`** solves the false activation problem. Skills can have broad triggers -- a keyword match might load a skill even when the session isn't really about that skill. Instead of guessing whether to log or skip, Trailhead always writes the entry with a confidence score. On next activation, unused low-confidence entries get auto-removed. The index stays clean without a psychic significance filter.

- **high** -- User explicitly asked to work on something this skill owns
- **medium** -- Related to the skill's domain, skill may contribute
- **low** -- Keyword match triggered it, intent unclear

**`lastTouch`** records the timestamp of the last meaningful interaction with the skill during that session. Depth, not presence.

**`summary`** captures *what the skill contributed*, not just that it was loaded. Good summaries include specific counts, entity names, bugs found, patterns established, and architectural decisions. They're written for an agent that needs to understand the project state without reading the session transcript.

## How it works

### On activation

When a skill with Trailhead is loaded, two guards run:

1. **Deduplication** -- If the current session ID is already in the index, stop. Handles skill reloads within the same session.
2. **Confidence assessment** -- Score how likely the skill is to contribute. Write the entry regardless of score.

### Cleanup (async, on next activation)

Session close is unreliable -- sessions end by running out of context, compacting, or the user walking away. Instead of relying on a close protocol, cleanup runs as a **background subagent** at the start of the next activation:

1. **Auto-remove unused lows** -- Any previous entry with confidence `"low"` and `lastTouch` equal to its activation time gets removed (skill was loaded but never used)
2. **Flag stale entries** -- Generic or placeholder summaries get noted for removal on the next pass

The subagent runs in the background so the main session isn't blocked.

### Skills that teach themselves

Every session where a skill contributes is an opportunity for that skill to get better. The protocol prompts updating the skill's own documentation with new learnings from the session. The session index is the trail of that growth.

## When to use Trailhead

**Good candidates:**
- Project skills where work compounds across sessions
- Domain skills with evolving state (databases, spreadsheets, relationship maps)
- Skills that manage external artifacts where you need to know what changed and when

**Skip it for:**
- Utility skills (credential lookups, tool wrappers)
- One-shot operations (creating a gist, sending a notification)
- Skills with no persistent state

**Rule of thumb:** If you'd never ask "what did we do with this skill last time?", it doesn't need Trailhead.

### Batch installation

Trailhead is a meta-skill -- it installs session tracking into other skills. You can install it on one skill at a time, or batch-install across a whole tier of skills in a single session:

```
/trailhead my-project-skill
```

The installer creates the session index file, adds On Activation and Session History sections to the target skill's SKILL.md, and commits the changes.

## Installation

Copy the three files into your Claude Code project:

```
.claude/skills/trailhead/SKILL.md    # The installer
.claude/includes/trailhead.md         # The runtime protocol
.claude/commands/trailhead.md         # The /trailhead slash command
```

### Manual setup

Create `{skill-name}-sessions.json` in the skill's directory with an empty array (`[]`), then add an On Activation section and Session History section to the skill's SKILL.md pointing to the protocol and index file. See the [full protocol](includes/trailhead.md) for details.

## File structure

```
.claude/
  skills/
    trailhead/
      SKILL.md                        # Installer skill
    my-project/
      SKILL.md                        # Your skill (modified by installer)
      my-project-sessions.json        # Session index (created by installer)
  includes/
    trailhead.md                      # Shared runtime protocol
  commands/
    trailhead.md                      # /trailhead slash command
```

## License

MIT
