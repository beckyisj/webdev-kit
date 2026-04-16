---
name: ship
description: End-to-end ship workflow — commit, push, deploy to Vercel, and save project memory. Runs after a build session to handle all the boring stuff.
argument-hint: [optional commit message]
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
---

# Ship It

Post-build skill that handles commit → push → deploy → memory in one shot. No more asking for each step.

## When to use

After finishing a build session — new feature, bug fix, new tool, client deliverable. Replaces the manual "ok commit this, push it, deploy it, remember it" sequence.

## Process

### Step 1: Assess what changed

Run `git status` and `git diff --stat` in the current project directory. Identify:
- Which files changed
- Whether this is a new project or update to existing
- Whether there are sensitive files (.env, credentials) — exclude those

If there are no changes, say so and stop.

### Step 2: Commit

- Stage relevant files (NOT node_modules, .env, .DS_Store, credentials)
- If the user provided a commit message in the argument, use it
- If not, write a concise commit message based on the diff (1-2 sentences, focus on "why")
- Append `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
- Commit. If pre-commit hook fails, fix and retry.

### Step 3: Push

- Check if branch tracks a remote. If not, push with `-u origin <branch>`
- Push to remote

### Step 4: Deploy to Vercel

Check if this is a Vercel project:
```bash
ls vercel.json .vercel/project.json 2>/dev/null
```

If yes:
- Run `vercel --prod` for production deploy (default)
- Wait for deploy to complete
- Report the URL

If no Vercel config exists, skip this step and mention it.

### Step 5: Save project memory

Check if a memory file already exists for this project:
```bash
ls ~/.claude/projects/-Users-beckyisjwara/memory/*<project-name>*.md 2>/dev/null
```

- If exists: update it with what changed in this session
- If new project: create a memory file with:
  - Project name and directory
  - What it does (1-2 sentences)
  - Tech stack
  - Deploy URL (if Vercel)
  - Date shipped
- Update MEMORY.md index if a new memory file was created

### Step 6: Check if it's a youtubeproducer.app tool

If the project deploys to a `*.youtubeproducer.app` subdomain:
- Verify the subdomain is listed in the MEMORY.md "YouTube Producer Tools" section
- If not, add it

### Step 7: Summary

Report:
```
Shipped:
- Commit: <hash> — <message>
- Push: <branch> → origin
- Deploy: <url> (or "skipped — no Vercel config")
- Memory: <created/updated> <filename>
```

## Important

- Never commit .env files, API keys, or node_modules
- If `git status` shows nothing to commit, just say "nothing to ship" and stop
- Don't amend previous commits — always create new ones
- If deploy fails, still complete the other steps and report the error
- Keep commit messages concise — the diff tells the story
