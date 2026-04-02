# Trailhead

Session tracking for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills. Every session that does substantive work on a tracked skill leaves a trail marker, so the next session can find its way back.

## The Problem

Claude Code sessions are stateless. When you return to a skill after days or weeks, there's no record of what happened before. You lose context, repeat work, and miss learnings from prior sessions.

## What Trailhead Does

Trailhead adds a lightweight session index to any skill. Each time a session does meaningful work on that skill, it logs:

- **Date** of the session
- **Session ID** for context recovery
- **Summary** of what was done

The index lives alongside the skill, so future sessions can scan it and pick up where the last one left off.

### Three Guard Layers

Trailhead doesn't log every session that touches a skill. Three guards keep the index useful:

1. **Deduplication** -- If the current session is already in the index, skip it (handles skill reloads within a session)
2. **Significance Filter** -- Only log sessions that modify files, investigate issues, or do design/planning work. Skip incidental loads and read-only access.
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

1. Create `{skill-name}-sessions.md` in the skill's directory:

```markdown
# {Skill Name} -- Session Index

Sessions where significant work was done. Most recent first.

| Date | Session ID | Summary |
|------|-----------|---------|
```

2. Add an **On Activation** section to the skill's SKILL.md (after frontmatter):

```markdown
## On Activation

Follow the Trailhead protocol in `/.claude/includes/trailhead.md`.
Session index: `/.claude/skills/{skill-name}/{skill-name}-sessions.md`
```

3. Add a **Session History** section (wherever makes sense):

```markdown
## Session History

Past sessions with summaries and IDs for easy context recovery:
**Session index:** `/.claude/skills/{skill-name}/{skill-name}-sessions.md`
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
2. **Guard 2 (Significance):** Assess whether this session will do substantive work on the skill. If not, skip.
3. **Write entry:** Append a new row with date, session ID, and a brief summary.

When the session wraps up, the On Close protocol:

1. Re-reads the index
2. Updates the summary if scope changed (e.g., started as "reviewing config" but ended up "rebuilding the pipeline")
3. Updates the skill's includes or SKILL.md if new learnings emerged

## File Structure

```
.claude/
  skills/
    trailhead/
      SKILL.md                        # Installer skill
    my-project/
      SKILL.md                        # Your skill (modified by installer)
      my-project-sessions.md          # Session index (created by installer)
  includes/
    trailhead.md                      # Shared runtime protocol
  commands/
    trailhead.md                      # /trailhead slash command
```

## License

MIT
