# Trailhead

Session tracking for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills. Every session where a tracked skill meaningfully contributes to the work leaves a trail marker, so the next session can find its way back.

## The Problem

Claude Code sessions are stateless. When you return to a skill after days or weeks, there's no record of what happened before. You lose context, repeat work, and miss learnings from prior sessions.

## What Trailhead Does

Trailhead adds a lightweight JSON session index to any skill. Each time a skill contributes to a session's work, it logs:

- **Date** of the session
- **Session ID** for context recovery
- **Confidence** level -- how useful the skill was expected to be (updated on close with what actually happened)
- **Last touch** timestamp -- when the skill was last actively used in the session
- **Summary** of how the skill contributed

The index lives alongside the skill, so future sessions can scan it and pick up where the last one left off.

### Example session entry

```json
{
  "date": "2026-04-01",
  "sessionId": "abc123def",
  "confidence": "high",
  "lastTouch": "2026-04-01T16:42:00-04:00",
  "summary": "Used cell maps to update comparison model with new SALT cap logic"
}
```

### Three Guard Layers

Trailhead doesn't log every session that touches a skill. Three guards keep the index useful:

1. **Deduplication** -- If the current session is already in the index, skip it (handles skill reloads within a session)
2. **Confidence scoring** -- Every activation gets a confidence level (high/medium/low). On session close, unused low-confidence entries are auto-removed.
3. **Installer Gate** -- Warns before installing on utility skills where session history doesn't add value

## Installation

### Option 1: Use the installer skill

Copy the three files into your Claude Code project:

```
.claude/skills/trailhead/SKILL.md    # The installer skill
.claude/includes/trailhead.md         # The runtime protocol
.claude/commands/trailhead.md         # The /trailhead slash command
```

Then run:

```
/trailhead my-skill-name
```

This creates the session index and wires up the On Activation and Session History sections in your skill's SKILL.md automatically.

### Option 2: Manual setup

If you prefer to set it up by hand:

1. Create `{skill-name}-sessions.json` in the skill's directory:

```json
[]
```

2. Add an **On Activation** section to the skill's SKILL.md (after frontmatter):

```markdown
## On Activation

Follow the Trailhead protocol in `/.claude/includes/trailhead.md`.
Session index: `/.claude/skills/{skill-name}/{skill-name}-sessions.json`
```

3. Add a **Session History** section (wherever makes sense):

```markdown
## Session History

Past sessions with summaries and IDs for easy context recovery:
**Session index:** `/.claude/skills/{skill-name}/{skill-name}-sessions.json`
```

## When to Use Trailhead

**Good candidates:**
- Project skills where work builds on itself across sessions
- Domain skills with evolving state
- Skills that manage external artifacts (spreadsheets, websites, databases)

**Skip it for:**
- Utility/reference skills (credential lookups, tool wrappers)
- One-shot operation skills (creating a gist, sending a message)
- Skills with no persistent state

**Rule of thumb:** If you'd never say "what did we do with this skill last time?", it doesn't need Trailhead.

## How It Works at Runtime

When a skill with Trailhead is loaded, the On Activation protocol runs:

1. **Guard 1 (Deduplication):** Read the session index. If the current session ID is already there, stop.
2. **Guard 2 (Confidence):** Assess how likely the skill is to contribute to this session. Assign high/medium/low.
3. **Write entry:** Prepend a new JSON entry with date, session ID, confidence, timestamp, and summary.

When the session wraps up, the On Close protocol:

1. Updates `lastTouch` to the current time
2. Updates `confidence` based on what actually happened (a "low" that turned out useful gets bumped; a "high" that was never used gets downgraded)
3. Updates `summary` if scope changed
4. **Auto-removes** any entry where confidence is "low" and `lastTouch` still equals the original activation time (skill was loaded but never used)
5. Updates the skill's includes or SKILL.md if new learnings emerged

## File Structure

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
