# 🚀 Git & GitHub Ultimate Command Cheat Sheet

A quick reference guide for all the essential Git and GitHub commands, from basic setup to advanced time-travel.

## ⚙️ 1. Setup & Configuration
* **`git config --global user.name "Name"`**: Sets your username for commits.
* **`git config --global user.email "email@example.com"`**: Sets your email for commits.
* **`git init`**: Initializes a brand new, empty local Git repository in your current folder.

## 🔄 2. Basic Workflow (Local)
* **`git status`**: Shows the state of your working directory (what is modified, staged, or untracked).
* **`git add <file>`**: Stages a specific file, prepping it for the next commit.
* **`git add .`**: Stages ALL modified and new files in the current directory.
* **`git commit -m "Message"`**: Takes a snapshot of the staged files and saves them permanently to local history.
* **`git log`**: Displays the commit history.
* **`git log --oneline`**: Shows a compact, single-line version of the commit history.

## ☁️ 3. GitHub & Remote Repositories
* **`git clone <url>`**: Downloads a complete copy of a remote GitHub repository to your local machine.
* **`git remote add origin <url>`**: Links your local repository to a remote GitHub repository.
* **`git push -u origin <branch>`**: Uploads your local branch commits to the remote repository.
* **`git pull origin <branch>`**: Fetches the newest updates from GitHub and immediately merges them into your local files.
* **`git fetch`**: Downloads history and changes from the remote repository but does NOT merge them yet.

## 🌿 4. Branching & Merging
* **`git branch`**: Lists all your local branches.
* **`git branch <branch-name>`**: Creates a new branch.
* **`git switch <branch-name>`**: Switches your working directory to the specified branch. (Modern alternative to `git checkout`).
* **`git switch -c <branch-name>`**: Creates a new branch and switches to it immediately in one step.
* **`git merge <branch-name>`**: Combines the specified branch's history into your current active branch.

## 🎒 5. Stashing (Temporary saves)
* **`git stash`**: Temporarily hides your modified, uncommitted files so you can work on something else.
* **`git stash pop`**: Restores the most recently stashed files back into your working directory.
* **`git stash list`**: Shows a list of all your currently stashed changes.

## ⏪ 6. Undoing Mistakes
* **`git restore --staged <file>`**: Unstages a file but keeps the edits in your working directory.
* **`git revert <commit-hash>`**: The safe undo. Creates a *new* commit that perfectly reverses a previous commit.
* **`git reset --soft HEAD~1`**: Undoes the last commit but keeps your files changed and staged.
* **`git reset --hard HEAD~1`**: ⚠️ **DANGER:** Permanently deletes the last commit and destroys all uncommitted changes.
* **`git reflog`**: The ultimate safety net. Shows a log of every single action you've taken, letting you recover from bad resets.

## 🏗️ 7. Advanced Tools
* **`git rebase <branch>`**: ⚠️ Moves your current branch to start at the tip of another branch, creating a clean, straight history.
* **`git rebase -i HEAD~3`**: Opens an interactive session to squash (combine), edit, or delete your last 3 commits.
* **`git cherry-pick <commit-hash>`**: Copies the changes from one specific commit and pastes them into your current branch.
* **`git bisect start`**: Starts a binary search process to help you find the exact commit that introduced a bug.
