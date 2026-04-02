---
name: trailhead
description: Meta skill that installs session tracking into any other skill. Creates the session index file and wires up the On Activation and Session History sections in the target skill's SKILL.md. Every session that works on a tracked skill leaves a trail marker so the next session can find its way back. Invoke via /trailhead or when the user asks to add session tracking to a skill.
triggers:
  - "install session tracking"
  - "add session tracking"
  - "track sessions for"
  - "trailhead"
---

# Trailhead

Meta skill that installs session tracking into any target skill. Every session that works on a tracked skill leaves a trail marker -- so the next session can find its way back.

## What It Does

1. **Creates** `{skill-name}-sessions.json` in the target skill's directory with an empty array
2. **Adds** an `## On Activation` section to the target SKILL.md (after frontmatter, before first content section)
3. **Adds** a `## Session History` section to the target SKILL.md (before the last section, or at the end)
4. **Reports** what was added and commits the changes

## Usage

```
/trailhead {skill-name}
```

Or naturally: "Add session tracking to the google-drive skill"

## Installation Steps

When invoked with a target skill name:

### Step 1: Validate the target skill

```bash
ls .claude/skills/{skill-name}/SKILL.md
```

If the skill doesn't exist, tell the user and stop.

### Step 2: Check if already installed

```bash
grep -l "trailhead" .claude/skills/{skill-name}/SKILL.md
```

If found, tell the user Trailhead is already installed and stop.

### Step 2.5: Skill type gate

Read the target skill's SKILL.md and assess whether it's a project-oriented skill or a utility skill. Session tracking is designed for project skills where work compounds across sessions.

**Install without warning for:** Skills that manage projects, external artifacts, or evolving domains.

**Warn before installing on:** Utility/reference skills, one-shot operation skills, or tool wrapper skills with no persistent state. Tell the user: "This looks like a utility skill -- Trailhead works best on project-oriented skills where work builds across sessions. Install anyway?"

If the user confirms, proceed. If not, stop.

### Step 3: Create the session index file

Write `.claude/skills/{skill-name}/{skill-name}-sessions.json`:

```json
[]
```

### Step 4: Add On Activation section to SKILL.md

Insert after the frontmatter closing `---` and before the first `#` heading:

```markdown
## On Activation

Follow the Trailhead protocol in `/.claude/includes/trailhead.md`.
Session index: `/.claude/skills/{skill-name}/{skill-name}-sessions.json`
```

If the skill already has an `## On Activation` section, append the two lines to it instead of creating a duplicate section.

### Step 5: Add Session History section to SKILL.md

Insert before the last section of the file (or at the end if placement is unclear):

```markdown
## Session History

Past sessions with summaries and IDs for easy context recovery:
**Session index:** `/.claude/skills/{skill-name}/{skill-name}-sessions.json`
```

### Step 6: Commit

```bash
git add .claude/skills/{skill-name}/{skill-name}-sessions.json .claude/skills/{skill-name}/SKILL.md
git commit -m "Add Trailhead session tracking to {skill-name} skill"
```

### Step 7: Confirm

Tell the user what was installed and that the skill will now track sessions automatically.

## Protocol Reference

The shared protocol that defines how session tracking works at runtime:
`/.claude/includes/trailhead.md`

## Notes

- This installer does not modify the skill's triggers or description
- The On Activation section is intentionally minimal -- it just points to the shared protocol
- If the user wants to backfill past sessions, they can search past session transcripts and manually add entries to the index
