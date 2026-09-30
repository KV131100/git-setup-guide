# Git Setup Guide

Notes from setting up Git on Ubuntu Linux and connecting it to GitHub over SSH, plus a quick primer on "agent mode" in AI coding tools.

## Contents

1. [What is agent mode?](#1-what-is-agent-mode)
2. [Connecting Git to GitHub](#2-connecting-git-to-github)
3. [Creating this repo](#3-creating-this-repo)
4. [Command cheat sheet](#4-command-cheat-sheet)
5. [Troubleshooting](#5-troubleshooting)

---

## 1. What is agent mode?

**Agent mode** is when an AI assistant doesn't just answer your question. It works on a task by itself: it makes a plan, uses tools (reads files, runs commands, edits code, searches the web), checks the results, and keeps going until the task is finished.

| | Chat mode | Agent mode |
|---|---|---|
| What it does | Answers or suggests | Does the task itself |
| Tools | None, or very few | Files, terminal, web, APIs |
| Steps | One reply | Many steps in a loop: plan → act → check → repeat |
| Your role | You apply the suggestions yourself | You review and approve what it does |

**Where you'll see the term:**

- **VS Code Copilot / Cursor / Windsurf:** "Agent" is a setting in the chat panel, next to "Ask" and "Edit." It edits several files and runs terminal commands for you.
- **ChatGPT:** agent mode uses a virtual browser and computer to complete tasks.
- **Claude Code:** always agentic. **Permission modes** control how much it does without asking you (press `Shift+Tab` to switch):
  - **Default:** asks before editing files or running commands
  - **Accept edits:** edits files without asking
  - **Plan mode:** only reads and plans; makes no changes until you approve
  - **Auto mode:** runs with fewer approval prompts

---

## 2. Connecting Git to GitHub

### Step 1: Check Git is installed

```bash
git --version
```

If it isn't found, install it:

```bash
sudo apt install git
```

### Step 2: Tell Git who you are

```bash
git config --global user.name "KV131100"
git config --global user.email "karthik.boredha1@gmail.com"
```

**Why:** every commit is stamped with this name and email. Use the email on your GitHub account so commits link to your profile.

Check it:

```bash
git config --global --list
```

### Step 3: Create an SSH key

```bash
ssh-keygen -t ed25519 -C "karthik.boredha1@gmail.com"
```

Press Enter to accept the default location (`~/.ssh/id_ed25519`). A passphrase is optional.

**Why:** GitHub no longer accepts passwords for Git. The key proves it's your machine: the **private** key (`id_ed25519`) stays on your computer, and you give GitHub the **public** key (`id_ed25519.pub`).

### Step 4: Add the public key to GitHub

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the output, then on GitHub go to **Settings → SSH and GPG keys → New SSH key** and paste it in.

### Step 5: Test the connection

```bash
ssh -T git@github.com
```

If it worked, you'll see:

```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

### Step 6: Use it

**Download an existing repo:**

```bash
git clone git@github.com:<username>/<repo>.git
```

**Or turn a local folder into a repo and push it** (create an empty repo on GitHub first):

```bash
cd my-project
git init
git add .
git commit -m "First commit"
git branch -M main
git remote add origin git@github.com:<username>/<repo>.git
git push -u origin main
```

**Why `remote add`:** it gives the GitHub URL a short name, `origin`, so from now on you can just type `git push` and `git pull`.

### Alternative: GitHub CLI

The GitHub CLI can handle authentication (steps 3–5) for you:

```bash
sudo apt install gh
gh auth login
```

---

## 3. Creating this repo

This repo was created locally with:

```bash
mkdir ~/git-setup-guide
cd ~/git-setup-guide
git init -b main
git add README.md
git commit -m "Add Git setup guide README"
```

To publish it on GitHub:

1. On GitHub, create a new **empty** repository named `git-setup-guide`. Don't add a README, `.gitignore`, or license, because this repo already has a README.
2. Connect it and push:

```bash
cd ~/git-setup-guide
git remote add origin git@github.com:KV131100/git-setup-guide.git
git push -u origin main
```

---

## 4. Command cheat sheet

| Command | What it does |
|---|---|
| `git --version` | Show the installed Git version |
| `git config --global user.name "..."` | Set your commit name |
| `git config --global user.email "..."` | Set your commit email |
| `git config --global --list` | Show your global settings |
| `ssh-keygen -t ed25519 -C "email"` | Create an SSH key |
| `cat ~/.ssh/id_ed25519.pub` | Print the public key to copy into GitHub |
| `ssh -T git@github.com` | Test the SSH connection to GitHub |
| `git clone <url>` | Download a repo |
| `git init` | Turn the current folder into a repo |
| `git status` | Show changed or untracked files |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Save staged changes as a commit |
| `git branch -M main` | Rename the current branch to `main` |
| `git remote add origin <url>` | Link the local repo to GitHub |
| `git remote -v` | Show linked remotes |
| `git push -u origin main` | Upload commits and set the default upstream |
| `git pull` | Download and merge new commits |
| `git log --oneline` | Show a short commit history |

---

## 5. Troubleshooting

**`Permission denied (publickey)`**
GitHub doesn't recognise your key. Check that:
- the key from `cat ~/.ssh/id_ed25519.pub` has been added to GitHub (Step 4)
- the SSH agent has the key loaded:
  ```bash
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
  ```

**`remote origin already exists`**
Point the existing remote at the new URL:
```bash
git remote set-url origin git@github.com:<username>/<repo>.git
```

**`Updates were rejected` / `fetch first`**
The GitHub repo has commits your local repo doesn't have, usually because a README was added on GitHub. Pull them in first:
```bash
git pull origin main --allow-unrelated-histories
git push -u origin main
```
