# Auto-Documentation System for Synapto

**Automatically updates Obsidian documentation when you code**

---

## 🎯 What It Does

This system automatically:
1. ✅ **Creates daily notes** with your development activity
2. ✅ **Logs git commits** with changed files
3. ✅ **Suggests documentation updates** based on what you changed
4. ✅ **Tracks progress** in Obsidian vault

---

## 📁 Components

### 1. Git Hooks (Automatic)

**Pre-commit hook** (`.git/hooks/pre-commit`):
- Runs BEFORE each commit
- Creates daily note for today if missing
- Lists all changed files
- Suggests relevant docs to update

**Post-commit hook** (`.git/hooks/post-commit`):
- Runs AFTER each commit
- Adds commit message to daily note
- Logs commit hash for reference

### 2. VS Code Tasks (Manual)

**Update Documentation** task:
- Scans recent changes
- Generates documentation suggestions
- Opens relevant Obsidian notes

**Create Daily Note** task:
- Creates daily note with template
- Lists open tasks
- Links to active work

### 3. Daily Note Template

**Location**: `Daily-Notes/Template.md`

Each daily note includes:
- Date and tags
- Changes made (auto-populated)
- Git commits (auto-logged)
- Tasks to update docs
- Notes section for manual additions

---

## 🚀 How to Use

### Automatic (Git Hooks)

**Just commit as normal:**
```bash
git add .
git commit -m "feat: Add new feature"
```

**What happens:**
1. Pre-commit creates/updates today's daily note
2. Lists changed files
3. Suggests docs to update (e.g., "Update Backend/AI-Services.md")
4. Post-commit adds your commit message
5. Daily note now has complete log

### Manual (VS Code Tasks)

**Run task:** `Ctrl+Shift+P` → "Tasks: Run Task" → "Update Obsidian Docs"

**What it does:**
1. Analyzes recent git commits
2. Identifies outdated documentation
3. Opens relevant notes in Obsidian
4. Shows suggested changes

---

## 📝 Daily Note Example

```markdown
---
tags: [daily-note, synapto]
date: 2026-01-05
---

# Daily Development - 2026-01-05

## Changes Made

### [13:30] Git Commit

**Changed files:**
```
backend/app/services/ai_service.py
backend/app/services/groq/client.py
```

- [ ] Update [[Backend/AI-Services]] if AI code changed

**Commit [a1b2c3d]:**
```
feat: Add streaming support to GroqClient
```

---

### [14:15] Git Commit

**Changed files:**
```
flutter_app/lib/features/inquiries/providers/inquiry_list_provider.dart
```

- [ ] Update [[Mobile-App/Flutter-App-Architecture]] if features changed

**Commit [e4f5g6h]:**
```
fix: Handle offline mode in inquiry list
```

---

## Manual Notes

Working on offline-first improvements. Need to document the mutation queue retry logic.

## Links
- [[Backend/AI-Services]]
- [[Mobile-App/Flutter-App-Architecture]]
```

---

## 🔧 Configuration

### Enable/Disable Auto-Documentation

**Disable temporarily:**
```bash
cd C:\Users\empir\synapto
chmod -x .git/hooks/pre-commit
chmod -x .git/hooks/post-commit
```

**Re-enable:**
```bash
chmod +x .git/hooks/pre-commit
chmod +x .git/hooks/post-commit
```

### Customize Suggestions

Edit `.git/hooks/pre-commit` to change which docs are suggested based on file patterns.

**Example - Add DevOps suggestions:**
```bash
DEVOPS_CHANGED=$(echo "$STAGED_FILES" | grep "^deployment/" || true)
if [ -n "$DEVOPS_CHANGED" ]; then
    echo "- [ ] Update [[DevOps/Deployment-Guide]] if infrastructure changed" >> "$DAILY_NOTE"
fi
```

---

## 🎨 VS Code Integration

### Task: Update Obsidian Docs

**Run:** `Ctrl+Shift+B` → "Update Obsidian Documentation"

**What it does:**
```bash
1. Get recent commits (last 7 days)
2. Analyze changed files
3. Identify affected documentation
4. Generate update suggestions
5. Open Obsidian vault
```

### Task: Create Daily Note

**Run:** `Ctrl+Shift+P` → "Tasks: Run Task" → "Create Daily Note"

**What it does:**
```bash
1. Create today's daily note from template
2. List uncommitted changes
3. Show open documentation tasks
4. Open in Obsidian
```

---

## 📊 Benefits

✅ **Never forget to document** - Automatic reminders
✅ **Complete development log** - Every commit tracked
✅ **Easy to review** - All changes in one place
✅ **Team collaboration** - Daily notes shareable
✅ **Historical record** - Full activity timeline

---

## 🔗 Related

- [[README]] - Documentation hub
- `Daily-Notes/Template.md` - Daily note template
- `.vscode/tasks.json` - VS Code automation tasks

#automation #documentation #obsidian #git-hooks
