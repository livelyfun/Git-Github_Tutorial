# 🚀 The Ultimate Git & GitHub Tutorial

Welcome to my comprehensive, step-by-step guide to mastering Git and GitHub. Whether you are typing your very first `git init` or trying to untangle a messy rebase, this repository has you covered.

## 🛠️ Prerequisites & Setup

Before diving into the tutorials, you need a GitHub account and Git installed on your local machine.

### 1. Create a GitHub Account
1. Go to [GitHub.com](https://github.com/).
2. Click **Sign up** in the top right corner.
3. Follow the on-screen prompts to enter your email, create a password, and choose a username.
4. Verify your account through the email GitHub sends you.

### 2. Download & Install Git
Find your operating system below and follow the instructions to install Git.

<details>
<summary><strong>🐧 Linux Distributions</strong></summary>

* **Arch Linux:** `sudo pacman -S git` *(Note: Fresh Arch Linux installations come completely barebones without default applications, so this manual step is required).*
* **Ubuntu / Debian / Pop!_OS / Mint:** `sudo apt install git`
* **Fedora / RHEL / CentOS:** `sudo dnf install git`
* **openSUSE:** `sudo zypper install git`
* **Alpine Linux:** `apk add git`
* **Gentoo:** `emerge --ask dev-vcs/git`
* **Void Linux:** `sudo xbps-install -Su git`
* **NixOS:** `nix-env -iA nixpkgs.git`
</details>

<details>
<summary><strong>🪟 Windows</strong></summary>

* **Standalone Installer:** Download from [git-scm.com/download/win](https://git-scm.com/download/win)
* **Winget (Command Line):** `winget install --id Git.Git -e --source winget`
* **Chocolatey:** `choco install git`
* **Scoop:** `scoop install git`
</details>

<details>
<summary><strong>🍏 macOS</strong></summary>

* **Homebrew:** `brew install git`
* **MacPorts:** `sudo port install git`
* **Xcode Command Line Tools:** `xcode-select --install` *(Prompts Git installation if not already present)*
</details>

<details>
<summary><strong>😈 BSD Systems</strong></summary>

* **FreeBSD:** `pkg install git`
* **OpenBSD:** `pkg_add git`
* **NetBSD:** `pkgin install git`
</details>

---

## 📂 What's Inside?

I have broken down the learning process into three logical phases, plus a quick-reference cheat sheet:

* **[Phase 1: Beginners](git-tutorial-beginner.md)** - Core architecture, basic commands (`add`, `commit`, `push`), and setting up your first repo.
* **[Phase 2: Intermediate](git-tutorial-intermediate.md)** - Branching, merging, handling merge conflicts, and stashing.
* **[Phase 3: Advanced Mastery](git-tutorial-advanced.md)** - Rebasing, cherry-picking, recovering lost commits with `reflog`, and hunting bugs with `bisect`.
* **[Ultimate Cheat Sheet](git-github-commands-cheatsheet.md)** - A quick, no-nonsense reference guide for all essential commands.

---

## 💻 How to Use This Guide

You can read the Markdown files directly here on GitHub. If you want to practice locally, you can clone this repository to your machine using your terminal:

```bash
git clone https://github.com/your-username/your-repo-name.git
```

## 🤝 Contributing
Found a typo, have a cool advanced Git trick to add, or want to translate a tutorial? Feel free to fork this repository, make your changes, and submit a Pull Request!
