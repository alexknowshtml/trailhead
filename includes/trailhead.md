# Trailhead -- Session Tracking Protocol

Reusable protocol for skills that want to track which Claude Code sessions have used them. Every session where a tracked skill meaningfully contributes to the work leaves a trail marker, so the next session can find its way back.

Any skill can opt in by referencing this protocol and maintaining a `{skill-name}-sessions.json` file in its skill directory. Use `/trailhead {skill-name}` to install.

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

1. Create `{skill-name}-sessions.json` in the skill's directory with an empty array:

```json
[]
```

2. Add these two sections to the skill's SKILL.md:

**On Activation section** (near the top, after the frontmatter):
```markdown
## On Activation

Follow the Trailhead protocol in `/.claude/includes/trailhead.md`.
Session index: `/.claude/skills/{skill-name}/{skill-name}-sessions.json`
```

**Session History section** (wherever makes sense):
```markdown
## Session History

Past sessions with summaries and IDs for easy context recovery:
**Session index:** `/.claude/skills/{skill-name}/{skill-name}-sessions.json`
```

## Session Entry Format

Each entry in the JSON array is an object with these fields:

```json
{
  "date": "2026-04-01",
  "sessionId": "abc123def",
  "confidence": "high",
  "lastTouch": "2026-04-01T16:42:00-04:00",
  "summary": "Updated comparison model with new SALT cap logic"
}
```

- **date:** Session date (`TZ="America/New_York" date +"%Y-%m-%d"`)
- **sessionId:** The current session ID (from the session context or transcript path)
- **confidence:** How likely the skill is to contribute meaningfully to this session: `"high"`, `"medium"`, or `"low"`
- **lastTouch:** ISO 8601 timestamp of when the entry was last updated (`TZ="America/New_York" date +"%Y-%m-%dT%H:%M:%S%:z"`)
- **summary:** A brief, specific description of how the skill contributed to the session

### Confidence Levels

- **high** -- User explicitly asked to work on something the skill owns or manages
- **medium** -- User's request overlaps with the skill's domain but isn't specific to this skill
- **low** -- Keyword match triggered the skill but intent is unclear

## On Activation Protocol

When a skill with Trailhead is loaded:

### Guard 1: Deduplication

Read the skill's session index file. If the current session ID already appears in the index, **stop** -- no action needed. This prevents duplicate entries when a skill is reloaded within the same session.

### Guard 2: Confidence Assessment

Assess how likely the skill is to contribute meaningfully to this session. Instead of a binary log/skip decision, assign a confidence level based on the user's request:

- **high** -- The user's request directly targets something this skill owns
- **medium** -- The request is related to the skill's domain but could go either way
- **low** -- The skill was triggered by a keyword match but intent is unclear

All confidence levels get an entry written. Cleanup happens on the next activation (see below).

### Writing the Entry

After both guards pass, prepend a new entry to the JSON array (most recent first):

```json
{
  "date": "{today}",
  "sessionId": "{session_id}",
  "confidence": "{high|medium|low}",
  "lastTouch": "{now}",
  "summary": "{brief description based on user's request}"
}
```

## Cleanup Protocol (runs on next activation)

Session close is unreliable -- sessions end by running out of context, compacting, or the user moving on. Instead of a close protocol, cleanup happens at the start of the *next* activation, as a background task.

After writing the current session's entry, launch a **background subagent** (using `run_in_background: true`) to review and clean up previous entries:

1. **Scan** all entries in the session index
2. **Auto-remove unused lows:** If a previous entry has confidence `"low"` and `lastTouch` equals its `date` timestamp (meaning the skill was loaded but never touched after activation), remove it
3. **Flag stale entries:** If any entry's summary looks generic or placeholder-like (e.g., "session activated"), note it for potential removal on the next pass

The subagent runs in the background so the main session isn't blocked. If the subagent modifies the index, it writes the updated JSON file directly.

## File Naming Convention

- Session index: `{skill-name}-sessions.json`
- Location: Inside the skill's own directory (`.claude/skills/{skill-name}/`)
- This keeps session tracking co-located with the skill and makes files easy to find via `glob .claude/skills/**/*-sessions.json`

## Notes

- New entries go at the **beginning** of the array (most recent first)
- Summaries should be one line, specific enough to be useful for context recovery
- If a session loads multiple tracked skills, each skill's index gets its own entry
- The confidence field turns the significance filter from a binary gate into a signal -- write first, clean up on next activation
