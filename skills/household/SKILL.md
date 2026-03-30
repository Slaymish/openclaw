---
name: household
description: "Shared household assistant for flatmates. Manages shopping lists, reminders, and notes stored in ~/.openclaw/household/data.json."
metadata: { "openclaw": { "emoji": "🏠" } }
---

# Household Assistant

Shared assistant for all flatmates. Any flatmate can read or update household data.

## Data Storage

All household data lives in `~/.openclaw/household/data.json`. Always read before writing to get current state.

Structure:

```json
{
  "shopping_list": [{ "item": "milk", "added_by": "Hamish", "added_at": "2026-03-30" }],
  "reminders": [
    { "text": "Pay rent", "due": "2026-04-01T08:00:00", "added_by": "Hamish", "cron_id": null }
  ],
  "notes": [{ "title": "Wifi password", "body": "...", "added_by": "Hamish" }],
  "flatmates": ["Hamish"]
}
```

## Commands

### Shopping List

- "add [item] to the shopping list" → append to `shopping_list`
- "what's on the shopping list" / "shopping list" → list all items
- "remove [item] from the shopping list" / "we got [item]" → remove matching item
- "clear the shopping list" → empty the array (confirm first)

### Reminders

- "remind everyone [time/date] to [thing]" → add to `reminders`, set up cron job via `openclaw cron`
- "what reminders do we have" → list upcoming reminders
- "cancel the reminder about [thing]" → remove matching reminder and cancel cron job

### Notes

- "save a note: [title] - [body]" → add to `notes`
- "what notes do we have" → list note titles
- "show the [title] note" → display full body

### Flatmates

- "add [name] as a flatmate" → append to `flatmates`
- "who are the flatmates" → list flatmates

## File Operations

Read current state:

```bash
cat ~/.openclaw/household/data.json 2>/dev/null || echo '{"shopping_list":[],"reminders":[],"notes":[],"flatmates":[]}'
```

Write updated state (always merge, never overwrite the whole file blindly):

```bash
# Read, modify in memory, write back atomically
cat ~/.openclaw/household/data.json | jq '.shopping_list += [{"item":"milk","added_by":"Hamish","added_at":"2026-03-30"}]' > /tmp/household-tmp.json && mv /tmp/household-tmp.json ~/.openclaw/household/data.json
```

Initialize if missing:

```bash
mkdir -p ~/.openclaw/household && [ -f ~/.openclaw/household/data.json ] || echo '{"shopping_list":[],"reminders":[],"notes":[],"flatmates":[]}' > ~/.openclaw/household/data.json
```

## Reminders via Cron

Set a one-off reminder using openclaw's cron system. Example — remind flatmates about rent on April 1st at 8am:

```
openclaw cron add --once "2026-04-01T08:00:00" "Reminder: Pay rent"
```

Store the cron ID in the reminder entry so it can be cancelled later.

## Response Format

**Shopping list:**

```
🛒 Shopping list (5 items):
• Milk (added by Hamish)
• Eggs
• Bread
• Butter
• Coffee
```

**Reminder set:**

```
⏰ Reminder set for April 1 at 8am:
"Pay rent"
I'll notify all flatmates then.
```

**Note saved:**

```
📝 Note saved: "Wifi password"
```

## Rules

- Always read the current file before modifying it — never assume its state.
- Use `jq` for atomic JSON modifications to avoid corruption.
- When a flatmate says "we" or "us", it applies to all flatmates.
- If the file doesn't exist, initialize it first.
- Don't delete notes or reminders without confirming.
- Keep responses short and practical — this is a quick-access tool.
