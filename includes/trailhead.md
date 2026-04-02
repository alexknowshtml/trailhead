# Trailhead -- Session Tracking Protocol

Reusable protocol for skills that want to track which Claude Code sessions have worked on them. Every session that does substantive work on a tracked skill leaves a trail marker, so the next session can find its way back.

Any skill can opt in by referencing this protocol and maintaining a `{skill-name}-sessions.md` file in its skill directory. Use `/trailhead {skill-name}` to install.

## When to Install Trailhead

Session tracking is for **project-oriented skills** where work builds on itself across sessions. Install it on skills where future sessions will benefit from knowing what happened in past sessions.

**Good candidates:**
- Project skills (e.g., a website, a campaign, a product build)
- Domain skills with evolving state (e.g., a CRM, analytics, content pipeline)
- Skills that manage external artifacts (spreadsheets, websites, databases)

**Do NOT install on:**
- Utility/reference skills (credential lookups, API wrappers)
- One-shot operation skills (creating a gist, sending a notification)
- Skills that are just tool wrappers with no persistent state

**Rule of thumb:** If you'd never say "what did we do with this skill last time?", it doesn't need Trailhead.

## Setup

To add Trailhead to a skill, run `/trailhead {skill-name}`. Or manually:

1. Create `{skill-name}-sessions.md` in the skill's directory with this template:

```markdown
# {Skill Name} -- Session Index

Sessions where significant work was done. Most recent first.

| Date | Session ID | Summary |
|------|-----------|---------|
```

2. Add these two sections to the skill's SKILL.md:

**On Activation section** (near the top, after the frontmatter):
```markdown
## On Activation

Follow the Trailhead protocol in `/.claude/includes/trailhead.md`.
Session index: `/.claude/skills/{skill-name}/{skill-name}-sessions.md`
```

**Session History section** (wherever makes sense):
```markdown
## Session History

Past sessions with summaries and IDs for easy context recovery:
**Session index:** `/.claude/skills/{skill-name}/{skill-name}-sessions.md`
```

## On Activation Protocol

When a skill with Trailhead is loaded:

### Guard 1: Deduplication

Read the skill's session index file. If the current session ID already appears in the index, **stop** -- no action needed. This prevents duplicate entries when a skill is reloaded within the same session.

### Guard 2: Significance Filter

Before writing an entry, assess whether this session is doing substantive work related to the skill. **Only create an entry if:**

- The session is likely to **modify files** the skill owns or manages (code, config, spreadsheets, content)
- The session is **investigating or debugging** something that will produce learnings worth recording
- The user is doing **planning or design work** that shapes future sessions on this skill

**Skip the entry if:**
- The skill was loaded incidentally (e.g., triggered by a keyword match but the session isn't really about this skill)
- The session is only **reading** data through the skill without changing anything
- The session is using the skill as a **utility** for a task that belongs to a different skill

When in doubt, ask: "Would a future session working on this skill benefit from knowing this session happened?" If no, skip.

### Writing the Entry

If both guards pass, append a new row at the top of the table (below the header):

- **Date:** Today's date (`TZ="America/New_York" date +"%Y-%m-%d"`)
- **Session ID:** The current session ID (from the session context or transcript path)
- **Summary:** A brief, specific description based on the user's request or task. Not a placeholder -- write something useful for future context recovery.
- Format: `| {date} | \`{session_id}\` | {summary of what triggered this session} |`

## On Session Close Protocol

When wrapping up a session that used a tracked skill:

1. **Re-read** the session index
2. **Find** the current session's row
3. **Update the summary** if the scope changed significantly from what was written on activation (e.g., started as "reviewing rates" but ended up "rebuilding the calculator")
4. If the session produced new learnings (new cells, formulas, gotchas, resolved questions), update the skill's relevant includes or SKILL.md per that skill's own maintenance instructions

## File Naming Convention

- Session index: `{skill-name}-sessions.md`
- Location: Inside the skill's own directory (`.claude/skills/{skill-name}/`)
- This keeps session tracking co-located with the skill and makes files easy to find via `glob .claude/skills/**/*-sessions.md`

## Notes

- New entries go at the **top** of the table (most recent first)
- Session IDs should be wrapped in backticks for readability
- Summaries should be one line, specific enough to be useful for context recovery
- If a session loads multiple tracked skills, each skill's index gets its own entry (subject to the significance filter for each)
