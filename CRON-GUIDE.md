# 🔧 Cron Job Guide — Java Research Automation

## How the Cron Job Works

### Flow

```
1. Cron triggers → OpenClaw agent wakes up
2. Agent reads README.md
3. Finds first task with [ ] (unchecked)
4. Researches the topic deeply
5. Creates research file: research/XX-topic-name.md
6. Pushes file to GitHub
7. Updates README.md: [ ] → [x], adds link to research file
8. Done — waits for next trigger
```

### Cron Job Configuration

```json
{
  "name": "java-research-task",
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * *",
    "tz": "Asia/Bangkok"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "You are a Java research assistant. Your job:\n\n1. Read the file `/home/ngocnq/.openclaw/workspace/devmode-senior/README.md`\n2. Find the FIRST task line that starts with `- [ ]` (unchecked task)\n3. Extract the task number and topic name\n4. Research this topic thoroughly — cover:\n   - Core concepts and theory\n   - How it works internally (with diagrams described in text)\n   - Real Java code examples\n   - Common pitfalls and best practices\n   - How it applies to banking/financial systems\n   - Recommended resources (books, articles, videos)\n5. Write the research as a detailed markdown file at `/home/ngocnq/.openclaw/workspace/devmode-senior/research/{task-number}-{topic-slug}.md`\n6. Run these shell commands to commit and push:\n   ```\n   cd /home/ngocnq/.openclaw/workspace/devmode-senior\n   git add -A\n   git commit -m 'research: complete task {number} - {topic}'\n   git push origin main\n   ```\n7. Update README.md:\n   - Change `- [ ] **Task XX**` to `- [x] **Task XX**`\n   - In the Research Files table, add a row: `| XX | Topic Name | research/XX-topic-name.md | ✅ |`\n   - Update the Progress section (completed count, remaining count, current phase)\n8. Commit and push the README update:\n   ```\n   git add README.md\n   git commit -m 'docs: mark task {number} as completed'\n   git push origin main\n   ```\n\nIMPORTANT:\n- Only do ONE task per run\n- If ALL tasks in a phase are done, update Current Phase to the next phase\n- Research must be in-depth (minimum 500 words), not surface-level\n- Include Java code examples that actually compile\n- Always push to GitHub after completing",
    "timeoutSeconds": 300
  },
  "delivery": {
    "mode": "announce"
  },
  "enabled": true
}
```

### Trigger Manually

```bash
# Run immediately (skip schedule)
openclaw cron run <job-id>

# Check run history
openclaw cron runs <job-id>

# List all jobs
openclaw cron list
```

### Adjusting Schedule

```bash
# Change to every 6 hours
openclaw cron update <job-id> --schedule '{"kind":"cron","expr":"0 */6 * * *","tz":"Asia/Bangkok"}'

# Change to twice daily (9am and 9pm)
openclaw cron update <job-id> --schedule '{"kind":"cron","expr":"0 9,21 * * *","tz":"Asia/Bangkok"}'

# Disable temporarily
openclaw cron update <job-id> --enabled false

# Re-enable
openclaw cron update <job-id> --enabled true
```

### Research File Format

Each research file should follow this structure:

```markdown
# Task XX: Topic Name

## 📖 Overview
(Brief summary of what this topic is about)

## 🔍 Deep Dive
(Detailed explanation with internal mechanisms)

## 💻 Code Examples
(Working Java code with comments)

## 🏦 Banking/Financial Application
(How this applies to banking systems specifically)

## ⚠️ Common Pitfalls
(Mistakes to avoid)

## ✅ Best Practices
(Production-ready recommendations)

## 📚 Resources
(Books, articles, videos for further learning)

---
*Researched on: YYYY-MM-DD*
```
