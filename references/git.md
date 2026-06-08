# Git Reference

## Table of Contents
1. Core Concepts
2. Branching Strategies
3. Essential Git Commands
4. Git Hooks
5. Conventional Commits
6. .gitignore Patterns
7. File Structure

---

## 1. Core Concepts

```
Working Directory → [git add] → Staging Area → [git commit] → Local Repo → [git push] → Remote Repo
                                                                                ↑
                                                                         [git pull]
```

**Key objects:**
- **Commit** — snapshot of all tracked files at a point in time
- **Branch** — a movable pointer to a commit
- **Remote** — a hosted copy of the repo (GitHub, GitLab, Bitbucket)
- **HEAD** — pointer to your current branch/commit

---

## 2. Branching Strategies

### Git Flow (structured releases)
```
main          ──────────────────────────────●──────────
                                           /
release/1.2  ─────────────────────────●──/
                                     /
develop       ──●──────────────●────/
                 \            /
feature/login    ──●──●──●──●
```

**Branches:**
- `main` — production-ready code only
- `develop` — integration branch
- `feature/*` — new features
- `release/*` — release prep
- `hotfix/*` — urgent production fixes

### GitHub Flow (simpler, CI/CD friendly)
```
main          ──●──────────────────────────────●──
                 \                            /
feature/my-feat   ──●──●──●──[PR + review]──●
```

**Rules:** branch from main → commit → open PR → review → merge → deploy

### Trunk-Based Development (fastest)
Everyone commits directly to `main` (or very short-lived branches < 1 day).
Feature flags used to hide incomplete features.

---

## 3. Essential Git Commands

```bash
# ── Setup ──────────────────────────────────────────────
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main

# ── Daily workflow ─────────────────────────────────────
git status                          # What changed?
git add .                           # Stage all changes
git add src/                        # Stage specific folder
git commit -m "feat: add login"     # Commit with message
git push origin feature/my-feat     # Push to remote

# ── Branching ─────────────────────────────────────────
git checkout -b feature/my-feat     # Create + switch branch
git checkout main                   # Switch to main
git branch -d feature/my-feat       # Delete branch (after merge)
git branch -a                       # List all branches

# ── Merging & Rebasing ────────────────────────────────
git merge feature/my-feat           # Merge into current branch
git rebase main                     # Replay commits on top of main
git rebase -i HEAD~3                # Interactive rebase (squash, reword)

# ── Undoing things ────────────────────────────────────
git restore file.txt                # Discard working dir changes
git restore --staged file.txt       # Unstage a file
git reset --soft HEAD~1             # Undo last commit, keep changes staged
git reset --hard HEAD~1             # ⚠️ Undo last commit AND discard changes
git revert abc1234                  # Create new commit that undoes abc1234 (safe)

# ── Stash ─────────────────────────────────────────────
git stash                           # Save dirty state temporarily
git stash pop                       # Re-apply stashed changes
git stash list                      # Show all stashes

# ── Inspection ────────────────────────────────────────
git log --oneline --graph           # Pretty commit graph
git diff main..feature/my-feat      # Diff between branches
git blame src/app.js                # Who changed each line
git bisect start                    # Binary search for a bug-introducing commit

# ── Remote ────────────────────────────────────────────
git remote -v                       # List remotes
git fetch origin                    # Download remote changes (no merge)
git pull --rebase origin main       # Pull + rebase (cleaner history)
git push --force-with-lease         # Safe force push (fails if remote changed)
```

⚠️ **Never use `git push --force` on shared branches.** Use `--force-with-lease` instead.

---

## 4. Git Hooks

Git hooks run scripts automatically on git events. Put them in `.git/hooks/` (local only)
or use `husky` to commit them to the repo.

```bash
# Install husky
npm install --save-dev husky
npx husky init
```

```bash
# .husky/pre-commit — runs before every commit
#!/bin/sh
npm run lint
npm run test -- --passWithNoTests
```

```bash
# .husky/commit-msg — enforces commit message format
#!/bin/sh
npx commitlint --edit $1
```

```js
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional']
}
```

---

## 5. Conventional Commits

Format: `type(scope): description`

| Type | When to use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code change, no feature/fix |
| `test` | Adding/fixing tests |
| `chore` | Build process, dependencies |
| `ci` | CI/CD changes |
| `perf` | Performance improvement |

```bash
# Examples
git commit -m "feat(auth): add Google OAuth login"
git commit -m "fix(api): handle null response from payment gateway"
git commit -m "ci: add docker build caching to GitHub Actions"
git commit -m "feat!: drop support for Node.js 16"   # ! = breaking change
```

Benefits: auto-generates changelogs, enables semantic versioning tools.

---

## 6. .gitignore Patterns

```gitignore
# Dependencies
node_modules/
vendor/
.venv/

# Build output
dist/
build/
*.egg-info/
__pycache__/
*.pyc

# Environment & secrets
.env
.env.local
.env.*.local
*.pem
*.key

# OS files
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp

# Logs
*.log
logs/

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars          # Contains secrets — use .tfvars.example instead

# Docker
.docker/
```

---

## 7. File Structure

```
my-project/
├── .git/                   # Git internals (don't touch)
├── .husky/                 # Git hook scripts (committed to repo)
│   ├── pre-commit          # Lint + test before commit
│   └── commit-msg          # Enforce commit message format
├── .github/
│   ├── CODEOWNERS          # Who reviews what
│   ├── pull_request_template.md
│   └── ISSUE_TEMPLATE/
├── .gitignore
├── .gitattributes          # Line endings, diff drivers
└── commitlint.config.js    # Commit message rules
```