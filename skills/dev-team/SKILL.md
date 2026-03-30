---
name: dev-team
description: "Multi-agent dev team. Spawns specialized Claude Code agents (dev, QA, design-review, copy) on a GitHub repo, coordinates their work, and creates a PR."
metadata: { "openclaw": { "emoji": "🧑‍💻", "requires": { "bins": ["claude", "gh", "git"] } } }
---

# Dev Team Skill

Orchestrate a team of specialist Claude Code agents on any GitHub repo. Each agent runs in its own git worktree with a focused role.

## When to Use

User says things like:

- "dev team: add dark mode to https://github.com/user/repo"
- "run the team on [repo] — [task description]"
- "get the team to review and fix [issue] in [repo]"
- "spawn agents on [repo]"

## Agents & Their Roles

| Agent             | Focus                                                                | System prompt key |
| ----------------- | -------------------------------------------------------------------- | ----------------- |
| **dev**           | Implement the feature/fix. Write clean, minimal code.                | `dev`             |
| **qa**            | Write or update tests. Find edge cases. Run the test suite.          | `qa`              |
| **design-review** | Review UI/UX consistency, accessibility, component patterns.         | `design`          |
| **copy**          | Review and improve user-facing text: labels, errors, docs, comments. | `copy`            |

Not every task needs all four. Use judgement:

- Pure backend task → skip design-review
- CLI tool → skip copy unless there are user messages
- Greenfield feature → use all four
- Bug fix → dev + qa at minimum

## Standard Workflow

### Step 1: Set up workspace

```bash
REPO_URL="https://github.com/user/repo"
TASK="add dark mode toggle to settings page"
WORKSPACE=$(mktemp -d)
REPO_NAME=$(basename $REPO_URL .git)

git clone $REPO_URL $WORKSPACE/$REPO_NAME
cd $WORKSPACE/$REPO_NAME
```

### Step 2: Create worktrees for each agent

```bash
BASE_BRANCH=main  # or detect: git symbolic-ref refs/remotes/origin/HEAD | sed 's|.*/||'
BRANCH_PREFIX="dev-team/$(date +%Y%m%d)"

git worktree add /tmp/${REPO_NAME}-dev   -b ${BRANCH_PREFIX}-dev   $BASE_BRANCH
git worktree add /tmp/${REPO_NAME}-qa    -b ${BRANCH_PREFIX}-qa    $BASE_BRANCH
git worktree add /tmp/${REPO_NAME}-design -b ${BRANCH_PREFIX}-design $BASE_BRANCH
git worktree add /tmp/${REPO_NAME}-copy  -b ${BRANCH_PREFIX}-copy  $BASE_BRANCH
```

### Step 3: Spawn agents in parallel (background + PTY)

```bash
# Dev agent
bash pty:true workdir:/tmp/${REPO_NAME}-dev background:true command:"claude 'You are the dev agent on a team. Task: ${TASK}. Implement the feature. Write clean, minimal code. Commit your changes when done. Then run: openclaw gateway wake --text \"Dev agent done\" --mode now'"

# QA agent (after cloning, reads the repo, writes/runs tests)
bash pty:true workdir:/tmp/${REPO_NAME}-qa background:true command:"claude 'You are the QA agent. Task: ${TASK}. Write tests covering the feature and edge cases. Run the test suite. Fix any failing tests you find. Commit when done. Then run: openclaw gateway wake --text \"QA agent done\" --mode now'"

# Design review agent
bash pty:true workdir:/tmp/${REPO_NAME}-design background:true command:"claude 'You are the design-review agent. Task: ${TASK}. Review the UI components affected, check accessibility (aria labels, keyboard nav, color contrast). Leave suggestions as inline comments in the code or as a DESIGN_REVIEW.md file. Commit when done. Then run: openclaw gateway wake --text \"Design agent done\" --mode now'"

# Copy agent
bash pty:true workdir:/tmp/${REPO_NAME}-copy background:true command:"claude 'You are the copy agent. Task: ${TASK}. Review all user-facing text: button labels, error messages, tooltips, placeholder text, docs. Fix typos, improve clarity, ensure consistency. Commit when done. Then run: openclaw gateway wake --text \"Copy agent done\" --mode now'"
```

### Step 4: Monitor progress

```bash
process action:list
process action:log sessionId:XXX  # check individual agents
```

Report to the user when each agent finishes. Don't kill slow agents.

### Step 5: Synthesize and create PR

Once all agents are done:

1. Check each worktree for commits:

```bash
git -C /tmp/${REPO_NAME}-dev log ${BASE_BRANCH}..HEAD --oneline
git -C /tmp/${REPO_NAME}-qa log ${BASE_BRANCH}..HEAD --oneline
```

2. Push each branch:

```bash
cd /tmp/${REPO_NAME}-dev && git push -u origin ${BRANCH_PREFIX}-dev
cd /tmp/${REPO_NAME}-qa  && git push -u origin ${BRANCH_PREFIX}-qa
# etc.
```

3. Create a summary PR for the main dev branch:

```bash
gh pr create \
  --repo $REPO_URL \
  --head ${BRANCH_PREFIX}-dev \
  --title "feat: ${TASK}" \
  --body "$(cat <<EOF
## Dev Team Run

**Task:** ${TASK}

### Agents

- 🧑‍💻 **Dev** → \`${BRANCH_PREFIX}-dev\`
- 🧪 **QA** → \`${BRANCH_PREFIX}-qa\`
- 🎨 **Design Review** → \`${BRANCH_PREFIX}-design\`
- ✍️ **Copy** → \`${BRANCH_PREFIX}-copy\`

Review each branch, merge the ones that look good.
EOF
)"
```

### Step 6: Clean up worktrees

```bash
git -C $WORKSPACE/$REPO_NAME worktree remove /tmp/${REPO_NAME}-dev --force
git -C $WORKSPACE/$REPO_NAME worktree remove /tmp/${REPO_NAME}-qa --force
git -C $WORKSPACE/$REPO_NAME worktree remove /tmp/${REPO_NAME}-design --force
git -C $WORKSPACE/$REPO_NAME worktree remove /tmp/${REPO_NAME}-copy --force
rm -rf $WORKSPACE
```

## Shortened Mode (single agent, quick task)

For small tasks or when the user says "quick fix" / "just the dev agent":

```bash
WORKSPACE=$(mktemp -d)
git clone $REPO_URL $WORKSPACE
bash pty:true workdir:$WORKSPACE background:true command:"claude --yolo '${TASK}. Commit and push. Then gh pr create.'"
```

## Rules

- Always use `pty:true` — Claude Code is interactive.
- Agents work in **separate worktrees**, never the same directory.
- Never start agents inside `~/Documents/openclaw/` (the live OpenClaw instance).
- If an agent hangs >10 minutes with no output, check with `process action:log`, then ask the user before killing.
- Report clearly when each agent starts and finishes.
- Use `openclaw gateway wake` at the end of each agent prompt for immediate notification.
- After all agents finish, always summarize: which branches have changes, what each agent did.

## Progress Updates

On start: "🚀 Dev team kicked off on [repo]. 4 agents running: dev, qa, design-review, copy."

Per completion: "✅ Dev agent done — pushed to [branch]."

On all done: "🏁 Dev team finished. PR created: [url]. Branches: dev ✅ / qa ✅ / design ✅ / copy ✅"
