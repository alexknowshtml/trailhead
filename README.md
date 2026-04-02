# Trailhead

Your agent invokes a skill it hasn't touched in two weeks. Or one you were using yesterday. Either way, it has no idea what happened in prior sessions -- unless it reads full transcripts.

Trailhead turns [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills into bridges between sessions. Each skill maintains a lightweight index of every session where it contributed to the work. When the skill is loaded, the agent can instantly see what other sessions used it, how deeply, and what they accomplished -- rebuilding a knowledge graph in seconds without transcript reads.

Skills become nodes. Sessions become edges. Follow the links and the agent reconstructs the full picture of a multi-session project from the skill alone.

## What it looks like

A JSON index that lives alongside your skill, updated automatically:

```json
[
  {
    "date": "2026-04-01",
    "sessionId": "227dd247",
    "confidence": "high",
    "lastTouch": "2026-04-01T17:42:00-04:00",
    "summary": "Used cell maps to update comparison model with new SALT cap logic"
  },
  {
    "date": "2026-03-18",
    "sessionId": "c3df8fe7",
    "confidence": "high",
    "lastTouch": "2026-03-18T15:20:00-04:00",
    "summary": "Expanded calculator to 10 test scenarios, refactored core logic"
  },
  {
    "date": "2026-03-15",
    "sessionId": "4c8908f1",
    "confidence": "medium",
    "lastTouch": "2026-03-15T11:05:00-04:00",
    "summary": "Rate verification against source data, corrected two values"
  }
]
```

In 10 seconds you know: what happened, how deep each session went, and which ones are worth recovering full context from.

## Why each field matters

**`confidence`** solves the false activation problem. Skills can have broad triggers -- a keyword match might load a skill even when the session isn't really about that skill. Instead of guessing whether to log or skip, Trailhead always writes the entry with a confidence score. On session close, unused low-confidence entries get auto-removed. The index stays useful without a psychic significance filter.

- **high** -- User explicitly asked to work on something this skill owns
- **medium** -- Related to the skill's domain but not specific
- **low** -- Keyword match triggered it, intent unclear

**`lastTouch`** tells you depth, not just presence. A session where `lastTouch` is three hours after activation was a deep working session. One where `lastTouch` equals the activation time was a drive-by. When deciding which past session to recover context from, that signal matters.

**`summary`** captures how the skill contributed, not just that it was loaded. Written on activation based on the user's request, then updated on close if scope changed.

## How it works

### On activation

When a skill with Trailhead is loaded, two guards run:

1. **Deduplication** -- If the current session ID is already in the index, stop. Handles skill reloads within the same session.
2. **Confidence assessment** -- Score how likely the skill is to contribute. Write the entry regardless of score.

### On session close

1. Update `lastTouch` to current time
2. Update `confidence` based on what actually happened -- bump lows that turned out useful, downgrade highs that went unused
3. Update `summary` if scope changed (started as "reviewing config" but ended up "rebuilding the pipeline")
4. **Auto-remove** any entry where confidence is "low" and `lastTouch` still equals activation time (loaded but never used)
5. Update the skill's own docs if new learnings emerged

### Skills that teach themselves

The close protocol doesn't just maintain the index. It prompts updating the skill's own documentation with new learnings from the session. Every session where a skill contributes is an opportunity for that skill to get better. The session index is the trail of that growth.

## When to use Trailhead

**Good candidates:**
- Project skills where work compounds across sessions
- Domain skills with evolving state
- Skills that manage external artifacts (spreadsheets, websites, databases)

**Skip it for:**
- Utility skills (credential lookups, tool wrappers)
- One-shot operations (creating a gist, sending a notification)
- Skills with no persistent state

**Rule of thumb:** If you'd never ask "what did we do with this skill last time?", it doesn't need Trailhead.

## Installation

Copy the three files into your Claude Code project:

```
.claude/skills/trailhead/SKILL.md    # The installer
.claude/includes/trailhead.md         # The runtime protocol
.claude/commands/trailhead.md         # The /trailhead slash command
```

Then run:

```
/trailhead my-skill-name
```

The installer creates the session index, adds On Activation and Session History sections to your skill's SKILL.md, and commits the changes.

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
