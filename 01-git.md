# Getting Started with Git

## What is Git?
Git is a distributed version control system that tracks changes to files over time, enabling collaboration and history management.

---

## Initial Setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## Core Workflow

**1. Initialize or clone a repo**
```bash
git init                        # start a new repo
git clone <url>                 # copy an existing repo
```

**2. Stage changes**
```bash
git status                      # see what's changed
git add <file>                  # stage a specific file
git add .                       # stage all changes
```

**3. Commit**
```bash
git commit -m "your message"    # save a snapshot
```

**4. Sync with remote**
```bash
git pull                        # fetch + merge from remote
git push                        # send commits to remote
```

---

## Branching

```bash
git branch feature-x            # create a branch
git checkout feature-x          # switch to it
git checkout -b feature-x       # create + switch in one step
git merge feature-x             # merge into current branch
```

---

## Inspect History

```bash
git log --oneline               # compact commit history
git diff                        # see unstaged changes
git diff --staged               # see staged changes
```

---

## Key Concepts

| Term | Meaning |
|------|---------|
| **Working tree** | Your local files |
| **Staging area** | Changes queued for commit |
| **Commit** | A saved snapshot |
| **Remote** | A hosted copy (e.g., GitHub) |
| **Branch** | An independent line of work |

---

## Common First-Time Flow

```bash
git init
git add .
git commit -m "first commit"
git remote add origin <url>
git push -u origin main
```
