Yes. If you're using Git for **DevOps work**, it is worth learning more than just `add`, `commit`, and `push`.

Below is a more complete Git guide organized by **what you want to do**, with examples using your `devops` repository.

# Git Complete Practical Guide

## 1. Understand the 4 important areas

Git has four concepts you should understand first:

```text
Your Files
   │
   │ git add
   ▼
Staging Area
   │
   │ git commit
   ▼
Local Repository
   │
   │ git push
   ▼
GitHub
```

And the opposite direction:

```text
GitHub
   │
   │ git pull
   ▼
Local Repository / Working Files
```

### Working directory

Your actual files:

```bash
ls
```

Example:

```text
ansible/
docker/
linux/
python/
shell/
```

### Staging area

Files selected for the next commit:

```bash
git add .
```

### Local repository

Commits stored locally:

```bash
git commit -m "Add Linux documentation"
```

### Remote repository

Your GitHub repository:

```bash
git push
```

---

# 2. `git status`

Probably the most important command while learning Git.

```bash
git status
```

Example:

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

Create a file:

```bash
touch linux/linux-commands.md
```

Then:

```bash
git status
```

You may see:

```text
Untracked files:
  linux/linux-commands.md
```

Git is telling you:

> I found a new file, but you haven't asked me to include it in the next commit.

---

# 3. `git add`

Stage files.

### One file

```bash
git add linux/linux-commands.md
```

### Multiple files

```bash
git add linux/linux-commands.md docker/docker.md
```

### Entire directory

```bash
git add linux/
```

### Everything

```bash
git add .
```

### Another option

```bash
git add -A
```

For normal work, I recommend:

```bash
git add .
```

---

# 4. `git commit`

Save staged changes into Git history.

```bash
git commit -m "Add Linux documentation"
```

The message should describe **what changed**.

Good:

```bash
git commit -m "Add Linux networking commands"
```

Good:

```bash
git commit -m "Add Docker installation guide"
```

Bad:

```bash
git commit -m "update"
```

Bad:

```bash
git commit -m "changes"
```

A useful format is:

```text
Add ...
Update ...
Fix ...
Remove ...
Refactor ...
```

Examples:

```bash
git commit -m "Add Kubernetes troubleshooting guide"
git commit -m "Update Ansible installation steps"
git commit -m "Fix Prometheus configuration"
git commit -m "Remove outdated Docker documentation"
```

---

# 5. `git push`

Send commits to GitHub.

```bash
git push origin main
```

If upstream is already configured:

```bash
git push
```

Your repository already says:

```text
Your branch is up to date with 'origin/main'
```

So you can normally use:

```bash
git push
```

---

# 6. `git pull`

Get the latest changes from GitHub.

```bash
git pull
```

or:

```bash
git pull origin main
```

Typical workflow:

```bash
git pull
```

Make changes:

```bash
vi linux/linux-commands.md
```

Then:

```bash
git add .
git commit -m "Update Linux documentation"
git push
```

---

# 7. `git fetch`

`fetch` downloads information from GitHub without changing your working files.

```bash
git fetch origin
```

Then check:

```bash
git status
```

You can use this when you want to inspect remote changes before integrating them.

Difference:

```text
git fetch
    ↓
Download remote information
    ↓
Do not automatically integrate it
```

Whereas:

```text
git pull
    ↓
Fetch
    ↓
Integrate changes
```

---

# 8. `git remote`

See your GitHub connection.

```bash
git remote -v
```

You have:

```text
origin  https://github.com/mahendranathdev/devops.git (fetch)
origin  https://github.com/mahendranathdev/devops.git (push)
```

### Show only remote names

```bash
git remote
```

### Detailed information

```bash
git remote show origin
```

---

# 9. `git branch`

List branches.

```bash
git branch
```

Example:

```text
* main
```

The `*` means current branch.

---

# 10. Create a branch

Suppose you want to work on Docker documentation.

```bash
git branch docker-docs
```

Check:

```bash
git branch
```

You might get:

```text
  docker-docs
* main
```

---

# 11. Switch branch

```bash
git switch docker-docs
```

Now:

```bash
git branch
```

shows:

```text
* docker-docs
  main
```

---

# 12. Create and switch at the same time

This is very useful:

```bash
git switch -c docker-docs
```

It does both:

```text
create branch
+
switch to branch
```

---

# 13. Work with branches

Example:

```bash
git switch -c kubernetes-docs
```

Create your documentation:

```bash
vi kubernetes/kubernetes.md
```

Then:

```bash
git add .
git commit -m "Add Kubernetes documentation"
```

Push the new branch:

```bash
git push -u origin kubernetes-docs
```

Now GitHub will contain:

```text
main
kubernetes-docs
```

---

# 14. Merge a branch

Suppose your work is finished.

Switch to main:

```bash
git switch main
```

Update main:

```bash
git pull
```

Merge your branch:

```bash
git merge kubernetes-docs
```

Then:

```bash
git push
```

Flow:

```text
kubernetes-docs
       │
       │ git merge
       ▼
      main
       │
       │ git push
       ▼
     GitHub
```

---

# 15. Delete a local branch

After merging:

```bash
git branch -d kubernetes-docs
```

Force deletion if necessary:

```bash
git branch -D kubernetes-docs
```

Be careful with `-D`.

---

# 16. Delete a remote branch

```bash
git push origin --delete kubernetes-docs
```

This deletes the branch from GitHub.

---

# 17. See commit history

Basic:

```bash
git log
```

Short:

```bash
git log --oneline
```

Example:

```text
a83c2f1 Add Kubernetes documentation
7a92b31 Add Docker documentation
3b7c821 Add Linux documentation
12a1d55 Initial commit
```

I recommend:

```bash
git log --oneline --graph --decorate --all
```

This gives you a better picture of branches.

---

# 18. `HEAD`

`HEAD` means roughly:

> Where your current branch is currently pointing.

Check:

```bash
git log --oneline -5
```

You can also use:

```bash
git show HEAD
```

Previous commit:

```bash
git show HEAD~1
```

Two commits before:

```bash
git show HEAD~2
```

---

# 19. `git diff`

Suppose you modify:

```bash
vi linux/linux-commands.md
```

Before staging:

```bash
git diff
```

This shows changes between your working file and the staged version.

---

# 20. `git diff --staged`

After:

```bash
git add linux/linux-commands.md
```

run:

```bash
git diff --staged
```

This shows exactly what you're about to commit.

A good habit:

```bash
git add .
git diff --staged
git commit -m "Add Linux documentation"
```

---

# 21. Unstage a file

Suppose:

```bash
git add .
```

but you accidentally staged something.

You can unstage:

```bash
git restore --staged filename
```

Example:

```bash
git restore --staged docker/test.txt
```

The file remains on your server, but it won't be included in the next commit.

---

# 22. Discard modifications

If you changed a file but don't want the changes:

```bash
git restore linux/linux-commands.md
```

⚠️ Be careful. This can discard your uncommitted changes.

---

# 23. Rename files

Use:

```bash
git mv old.md new.md
```

Example:

```bash
git mv linux/linux.md linux/linux-commands.md
```

Then:

```bash
git commit -m "Rename Linux documentation"
```

---

# 24. Delete files

```bash
git rm old.md
```

Then:

```bash
git commit -m "Remove old documentation"
```

---

# 25. `.gitignore`

This is extremely important for DevOps.

Create:

```bash
vi .gitignore
```

Example:

```text
*.log
*.tmp
.env
*.secret
*.key
*.pem
node_modules/
__pycache__/
.terraform/
*.tfstate
*.tfstate.backup
```

Then:

```bash
git add .gitignore
git commit -m "Add Git ignore rules"
git push
```

This prevents files such as:

```text
passwords
API keys
private keys
Terraform state
temporary files
logs
```

from accidentally being committed.

**Important:** `.gitignore` does not remove a file that Git is already tracking.

---

# 26. `git stash`

Very useful when you have unfinished work.

Suppose:

```bash
vi linux/linux.md
```

You have changes but suddenly need to switch branches.

Save them:

```bash
git stash
```

Now:

```bash
git status
```

should be clean.

Later:

```bash
git stash pop
```

Your changes return.

### See stashes

```bash
git stash list
```

Example:

```text
stash@{0}: WIP on main
stash@{1}: WIP on docker-docs
```

---

# 27. `git show`

Show a commit:

```bash
git show a83c2f1
```

Show latest:

```bash
git show HEAD
```

---

# 28. `git revert`

Suppose you pushed a bad commit:

```text
A → B → C
```

You want to undo C.

Use:

```bash
git revert C
```

Git creates a new commit:

```text
A → B → C → undo-C
```

This is generally safer for shared branches because it doesn't rewrite existing history.

---

# 29. `git reset`

Reset can be more dangerous because it can move your branch backward.

### Unstage everything

```bash
git reset
```

### Move HEAD back one commit but keep changes

```bash
git reset --soft HEAD~1
```

### Mixed reset

```bash
git reset HEAD~1
```

### Hard reset

```bash
git reset --hard HEAD~1
```

⚠️ Be very careful with:

```bash
git reset --hard
```

It can permanently remove uncommitted work.

---

# 30. `git clean`

Remove untracked files.

First see what would be removed:

```bash
git clean -n
```

Then actually remove:

```bash
git clean -f
```

For untracked directories too:

```bash
git clean -fd
```

⚠️ Be careful. These remove files that Git isn't tracking.

---

# 31. Tags

Tags are useful for versions/releases.

Create:

```bash
git tag v1.0.0
```

List:

```bash
git tag
```

Push:

```bash
git push origin v1.0.0
```

Or all tags:

```bash
git push origin --tags
```

Example:

```text
v1.0.0
v1.1.0
v2.0.0
```

---

# 32. Find a commit

```bash
git log --oneline
```

Then:

```bash
git show a83c2f1
```

You can also search commit messages:

```bash
git log --grep="Docker"
```

---

# 33. Search who changed a line

Very useful:

```bash
git blame linux/linux-commands.md
```

Example:

```text
a83c2f1 Mahendra 2026-08-31 line...
```

This helps identify which commit introduced a particular line.

---

# 34. Check branches on GitHub

```bash
git branch -a
```

Example:

```text
* main
  remotes/origin/main
  remotes/origin/docker-docs
```

---

# 35. Compare branches

For example:

```bash
git diff main..docker-docs
```

This shows differences between branches.

You can also see commits:

```bash
git log main..docker-docs --oneline
```

---

# 36. Clone a specific branch

```bash
git clone -b docker-docs https://github.com/mahendranathdev/devops.git
```

---

# 37. Clone with a different directory name

```bash
git clone https://github.com/mahendranathdev/devops.git devops-training
```

Instead of creating:

```text
devops/
```

it creates:

```text
devops-training/
```

---

# 38. Change GitHub repository URL

If the repository URL changes:

```bash
git remote set-url origin https://github.com/mahendranathdev/newrepo.git
```

Check:

```bash
git remote -v
```

---

# 39. Git configuration

Check your username:

```bash
git config --global user.name
```

Check email:

```bash
git config --global user.email
```

Set them:

```bash
git config --global user.name "Mahendra Nath"
git config --global user.email "your-email@example.com"
```

View configuration:

```bash
git config --list
```

---

# 40. Create a new repository

For a new project:

```bash
mkdir myproject
cd myproject
git init
```

Create file:

```bash
echo "# My Project" > README.md
```

Then:

```bash
git add .
git commit -m "Initial commit"
```

Connect GitHub:

```bash
git remote add origin https://github.com/USERNAME/myproject.git
```

Then:

```bash
git branch -M main
git push -u origin main
```

---

# 41. Your current `devops` repository

For your current server, this is the normal workflow:

```bash
cd /root/floci/devops
```

First:

```bash
git status
```

Get latest GitHub changes:

```bash
git pull
```

Create/edit documentation:

```bash
vi linux/linux-commands.md
```

Check:

```bash
git status
```

Review:

```bash
git diff
```

Stage:

```bash
git add .
```

Review staged changes:

```bash
git diff --staged
```

Commit:

```bash
git commit -m "Add Linux commands documentation"
```

Push:

```bash
git push
```

Verify:

```bash
git status
```

---

# 42. Recommended branch workflow for your DevOps documentation

Instead of making every change directly on `main`, you can use branches.

For example:

```bash
git switch -c linux-documentation
```

Work:

```bash
vi linux/linux-commands.md
```

Stage:

```bash
git add .
```

Commit:

```bash
git commit -m "Add Linux commands documentation"
```

Push:

```bash
git push -u origin linux-documentation
```

Then merge through GitHub Pull Request or locally.

This gives you:

```text
main
 │
 ├── linux-documentation
 │
 ├── docker-documentation
 │
 ├── kubernetes-documentation
 │
 ├── ansible-documentation
 │
 └── networking-documentation
```

That's a much better approach once your repository becomes large.

---

# 43. Git commands by category

## Basic

```bash
git init
git clone
git status
git add
git commit
git push
git pull
```

## Remote

```bash
git remote
git remote -v
git remote show origin
git remote set-url
git fetch
```

## Branches

```bash
git branch
git switch
git switch -c
git merge
git branch -d
git branch -D
```

## History

```bash
git log
git log --oneline
git show
git blame
git diff
```

## Undo

```bash
git restore
git restore --staged
git revert
git reset
git stash
```

## Files

```bash
git mv
git rm
git clean
```

## Releases

```bash
git tag
git push --tags
```

## Configuration

```bash
git config
```

---

# 44. The commands I recommend you memorize

Don't try to memorize 50 commands at once.

Start with these **15**:

```bash
git status
git add .
git commit -m "message"
git push
git pull
git clone
git branch
git switch
git switch -c branch-name
git merge
git log --oneline
git diff
git restore
git stash
git remote -v
```

Then learn:

```bash
git fetch
git revert
git reset
git tag
git blame
git clean
```

## One real-world example

Suppose tomorrow you add Docker documentation:

```bash
cd /root/floci/devops

git pull

git switch -c docker-documentation

vi docker/docker.md

git status

git diff

git add docker/docker.md

git diff --staged

git commit -m "Add Docker documentation"

git push -u origin docker-documentation
```

Then after the work is approved:

```bash
git switch main
git pull
git merge docker-documentation
git push
```

That is a **real DevOps-style Git workflow** and is much safer than always working directly on `main`.
