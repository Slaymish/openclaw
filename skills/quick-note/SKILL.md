# Quick-Note Skill

## Purpose

Quick reminders, notes, and pattern detection for Hamish.

## Commands

### Quick Reminders

Use this for short-term reminders that don't need a full cron job:

- `/note remember <thing>` - Quick mental note
- `/note remind <date/time> <thing>` - Set one-off reminder (converts to cron job)

### Pattern Detection

- `/note patterns` - Scan last 7 days of agentic-week logs, surface patterns and insights

### Project Momentum

- `/note stuck` - List in-flight projects and check if anything is stuck
- `/note review <project>` - Quick review of recent work on a specific project

## Pattern Detection Logic

Look for:

- Recurring blockers (same issue mentioned 3+ days)
- Incomplete work (started but not revisited in 3+ days)
- Emerging themes (similar topics mentioned repeatedly)
- Opportunities (good ideas not captured elsewhere)

## Output Format

**For patterns:**

```
Found 3 patterns in last 7 days:

🔄 Recurring theme: [theme]
   Mentioned on: [dates]
   What it means: [interpretation]
   Action: [what to do]

🚧 Potential blocker: [blocker]
   First seen: [date]
   Context: [what's stuck]

💡 Opportunity: [idea]
   First captured: [date]
   Notes: [why it's interesting]
```

**For stuck projects:**

```
In-flight projects:

1. [Project] (started [date])
   Last work: [date] ([X] days ago)
   Status: [what's happening]
   Next: [suggested action]
```

## Files

- `memory/quick-notes.json` - Stores reminders and notes
- Uses `memory/agentic-week/` for pattern detection

## Notes

- Keep it simple and practical
- Don't over-analyze - surface only what's actionable
- Reminders default to 8am unless specified
