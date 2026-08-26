# ⚡ Git & GitHub Ultimate Command Cheat Sheet

<div align="center">

[⬅️ Home](README.md) &nbsp;•&nbsp; [🌱 Phase 1: Beginners](git-tutorial-beginner.md) &nbsp;•&nbsp; [🌿 Phase 2: Intermediate](git-tutorial-intermediate.md) &nbsp;•&nbsp; [🌳 Phase 3: Advanced](git-tutorial-advanced.md)

</div>

---

A high-density reference sheet of essential Git and GitHub commands, organized by category, followed by the **🚨 Git Emergency Room** triage guide for instant recovery from mistakes.

---

## 📑 Quick Navigation
* [1. Configuration & Setup](#-1-configuration--setup)
* [2. Daily Local Workflow](#-2-daily-local-workflow)
* [3. Branching & Switching](#-3-branching--switching)
* [4. Remote Repositories & GitHub](#-4-remote-repositories--github)
* [5. Inspecting, Diffs & History](#-5-inspecting-diffs--history)
* [6. Stashing (Temporary Shelving)](#-6-stashing-temporary-shelving)
* [7. Undoing & Rewriting History](#-7-undoing--rewriting-history)
* [8. Advanced Debugging & Recovery](#-8-advanced-debugging--recovery)
* [🚨 The Git Emergency Room (Panic Fixes)](#-the-git-emergency-room-what-to-do-when-things-go-wrong)

---

## ⚙️ 1. Configuration & Setup

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git config --global user.name "<name>"` | Sets your global author name | `git config --global user.name "Alice Dev"` |
| `git config --global user.email "<email>"` | Sets your global author email | `git config --global user.email "alice@dev.io"` |
| `git config --global init.defaultBranch main` | Sets `main` as the default initial branch | Recommended for all new repos |
| `git config --global core.editor "<editor>"` | Sets default editor for commit messages | `git config --global core.editor "code --wait"` |
| `git config --list` | Displays all active Git configurations | Checks system, global, and local settings |
| `git init` | Initializes a new local Git repository | Creates the hidden `.git` folder |
| `git clone <url>` | Clones a remote repository to local machine | `git clone git@github.com:user/repo.git` |

---

## 🔄 2. Daily Local Workflow

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git status` | Shows state of working directory and staging area | Always run this first |
| `git add <file>` | Stages a specific file for the next commit | `git add index.html` |
| `git add .` | Stages ALL modified, deleted, and new files | Prepares entire project for commit |
| `git add -p` | Interactive staging (stages changes hunk by hunk) | Great for reviewing code piece by piece |
| `git commit -m "<msg>"` | Records staged snapshot to repository history | `git commit -m "feat: add user login"` |
| `git commit -am "<msg>"` | Stages tracked modified files AND commits in one step | Skips `git add` for existing tracked files |
| `git commit --amend` | Amends the most recent commit | Use to fix typo in message or add forgotten files |

---

## 🌿 3. Branching & Switching

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git branch` | Lists all local branches (* = current active branch) | `git branch` |
| `git branch -a` | Lists all local and remote-tracking branches | Shows `remotes/origin/...` |
| `git switch <branch>` | Switches to an existing branch | Modern replacement for `git checkout` |
| `git switch -c <branch>` | Creates and switches to a new branch immediately | `git switch -c feat/darkmode` |
| `git branch -m <new-name>` | Renames the current active branch | `git branch -m main` |
| `git merge <branch>` | Merges specified branch into your active branch | `git merge feat/darkmode` |
| `git branch -d <branch>` | Safely deletes a merged local branch | `git branch -d feat/darkmode` |
| `git branch -D <branch>` | Force deletes an unmerged local branch | ⚠️ Deletes unmerged commits |

---

## ☁️ 4. Remote Repositories & GitHub

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git remote add origin <url>` | Links local repository to a remote repository URL | `git remote add origin git@github.com:user/repo.git` |
| `git remote -v` | Lists all configured remote URLs | Shows fetch and push endpoints |
| `git push -u origin <branch>` | Pushes branch to remote and sets upstream tracking | `git push -u origin main` |
| `git push` | Pushes commits to the tracked upstream branch | Used after `-u` has been configured |
| `git pull` | Fetches remote changes AND merges them into active branch | Shortcut for `fetch` + `merge` |
| `git fetch` | Downloads objects and refs from remote without merging | Inspect remote work safely |
| `git push origin --delete <branch>` | Deletes a branch on GitHub remote | `git push origin --delete feat/legacy` |
| `git fetch --prune` | Removes stale remote tracking branches | Cleans up branches deleted on GitHub |

---

## 🔍 5. Inspecting, Diffs & History

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git diff` | Shows unstaged changes in working directory | Compares working tree vs staging area |
| `git diff --staged` | Shows changes that are staged for the next commit | Compares staging area vs last commit |
| `git diff <b1>..<b2>` | Shows differences between two branches | `git diff main..feature` |
| `git log` | Displays full commit history | Standard multi-line log |
| `git log --oneline` | Displays compact one-line-per-commit history | Quick history scan |
| `git log --oneline --graph --all` | Visual ASCII branch and merge tree | High-level repository topology |
| `git show <commit>` | Shows details, metadata, and diff of a specific commit | `git show 4a8b1c2` |
| `git blame <file>` | Shows line-by-line author and commit history of a file | Great for finding who wrote a specific line |

---

## 🎒 6. Stashing (Temporary Shelving)

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git stash` / `git stash push -m "<msg>"` | Saves dirty working changes to temporary stash | `git stash push -m "WIP: auth"` |
| `git stash list` | Lists all saved stashes | Shows `stash@{0}`, `stash@{1}`, etc. |
| `git stash pop` | Applies the latest stash AND removes it from list | Restores changes back to working tree |
| `git stash apply` | Applies the latest stash BUT keeps it in the stash list | Useful to apply stash on multiple branches |
| `git stash -u` | Stashes tracked AND untracked (new) files | Leaves directory completely clean |
| `git stash drop stash@{0}` | Deletes a specific stash from the list | Discards stashed changes |
| `git stash clear` | Permanently deletes all stashes | Empties the entire stash stack |

---

## ⏪ 7. Undoing & Rewriting History

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git restore <file>` | Discards uncommitted changes in working directory | ⚠️ Reverts file to last commit |
| `git restore --staged <file>` | Unstages a file while preserving your local edits | Moves file out of staging area |
| `git revert <commit>` | Creates a NEW commit that reverses specified commit | Safe for public branches |
| `git reset --soft HEAD~1` | Undoes last commit, keeps changes **staged** | Perfect for amending or reorganizing commits |
| `git reset --mixed HEAD~1` | Undoes last commit, keeps changes **unstaged** | Default reset behavior |
| `git reset --hard HEAD~1` | Undoes last commit and **DESTROYS** all local edits | ⚠️ Irreversible data loss risk |
| `git rebase <base-branch>` | Replays current branch commits on top of base branch | Creates a clean linear history |
| `git rebase -i HEAD~<N>` | Interactive rebase for the last N commits | Squash, reword, edit, or drop commits |
| `git cherry-pick <commit>` | Applies specific commit from another branch to active branch | `git cherry-pick 7a8b9c` |

---

## 🛠️ 8. Advanced Debugging & Recovery

| Command | Description | Example / Note |
| :--- | :--- | :--- |
| `git reflog` | Logs every single movement of `HEAD` (commits, checkouts, resets) | The ultimate emergency safety net |
| `git bisect start` | Initiates binary search mode to locate bug introduction | `git bisect start` |
| `git bisect bad` | Marks current commit as buggy / broken | `git bisect bad` |
| `git bisect good <commit>` | Marks a past commit as working / clean | `git bisect good v1.0.0` |
| `git bisect run <cmd>` | Automatically tests commits using an automated script | `git bisect run npm test` |
| `git bisect reset` | Exits bisect mode and returns to starting branch | Cleans up bisect state |

---

## 🚨 The Git Emergency Room (What to Do When Things Go Wrong)

| The Scenario / Mistake | The Immediate Solution |
| :--- | :--- |
| **"I accidentally committed directly to `main` instead of a feature branch!"** | 1. Create feature branch where you are: `git branch feat/my-work`<br/>2. Wind `main` back 1 commit: `git reset --hard HEAD~1`<br/>3. Switch to your feature branch: `git switch feat/my-work` |
| **"I typed the wrong commit message on my last commit."** | `git commit --amend -m "feat: corrected commit message"` |
| **"I forgot to add a file to the commit I just made."** | `git add forgotten-file.js`<br/>`git commit --amend --no-edit` |
| **"I accidentally deleted a branch with unmerged work!"** | 1. Run `git reflog` and locate the hash of the last commit on that branch.<br/>2. Run `git branch <branch-name> <commit-hash>` |
| **"I made messy uncommitted edits and want to discard EVERYTHING back to clean state."** | `git restore .`<br/>`git clean -fd` *(Removes untracked files and directories)* |
| **"I accidentally committed an API key or password (not yet pushed)!"** | 1. Undo commit: `git reset --soft HEAD~1`<br/>2. Remove secret file from staging: `git restore --staged .env`<br/>3. Add `.env` to `.gitignore`<br/>4. Re-commit: `git commit -m "feat: my commit without secret"` |
| **"Git says `error: failed to push some refs` because remote has new changes."** | Pull and rebase cleanly before pushing:<br/>`git pull --rebase origin main`<br/>`git push origin main` |
| **"I am stuck in Vim / an editor after typing `git commit`!"** | Press `Esc`, type `:wq`, and press `Enter` (or `ZZ`) to save and exit. To cancel without saving: `Esc`, `:q!`, `Enter`. |
| **"A merge or rebase went horribly wrong and I want to abort!"** | For a merge: `git merge --abort`<br/>For a rebase: `git rebase --abort` |

---

<div align="center">

[⬅️ Back to README](README.md) &nbsp;•&nbsp; [🌱 Phase 1](git-tutorial-beginner.md) &nbsp;•&nbsp; [🌿 Phase 2](git-tutorial-intermediate.md) &nbsp;•&nbsp; [🌳 Phase 3](git-tutorial-advanced.md)

</div>
