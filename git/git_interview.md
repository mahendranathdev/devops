

## 1. `git fetch` vs `git pull` vs `git clone`

This is one of the most common interview questions.

| Command     | What it does                                | Changes working files? | Typical use                  |
| ----------- | ------------------------------------------- | ---------------------: | ---------------------------- |
| `git clone` | Creates a local copy of a remote repository |  Yes, initial checkout | First time getting a repo    |
| `git fetch` | Downloads remote refs/objects               |                     No | Inspect remote changes first |
| `git pull`  | Fetches and integrates changes              |            Usually yes | Update current branch        |
| `git push`  | Sends local commits to remote               |        No local change | Publish your commits         |

Git's documentation describes `git fetch` as downloading objects and refs and updating remote-tracking branches. `git pull` first fetches and then integrates the changes, traditionally by merge or optionally by rebase. ([Git SCM][2])

### Example

Suppose GitHub has:

```text
A---B---C---D
        ^
      origin/main
```

Your server has:

```text
A---B---C
        ^
       main
```

Run:

```bash
git fetch origin
```

Now locally:

```text
A---B---C---D
        ^   ^
       main origin/main
```

Your working files did not automatically move to `D`.

Now:

```bash
git merge origin/main
```

or:

```bash
git pull
```

and your branch can move to `D`.

### Interview answer

> `git fetch` downloads remote changes and updates remote-tracking references without integrating them into my current branch. `git pull` performs a fetch and then integrates the fetched changes, usually with merge or rebase depending on configuration/options. I use fetch when I want to inspect changes before integrating them. ([Git SCM][2])

---

# 2. `git merge` vs `git rebase`

Another very common DevOps interview question.

| Merge                              | Rebase                                 |
| ---------------------------------- | -------------------------------------- |
| Combines histories                 | Replays commits on a new base          |
| Can create merge commit            | Usually creates a linear history       |
| Does not rewrite existing commits  | Rewrites commit IDs of rebased commits |
| Generally safer for shared history | Use carefully on shared branches       |

Example:

```text
main:    A---B---C
              \
feature:       D---E
```

Merge:

```text
A---B---C-------M
     \         /
      D---E----
```

Rebase:

```text
A---B---C---D'---E'
```

### Commands

Merge:

```bash
git switch main
git merge feature
```

Rebase:

```bash
git switch feature
git rebase main
```

Official Git documentation also notes that a branch can be configured so `git pull` uses rebase rather than merge. ([Git SCM][3])

### Interview answer

> I prefer rebase for cleaning up my private feature branch before a PR when project policy allows it. I avoid rebasing commits that other developers are already using because rebase rewrites history.

---

# 3. `git reset` vs `git revert` vs `git restore`

This is a **high-value interview topic**.

| Command   | Main purpose                                   | Rewrites history? |
| --------- | ---------------------------------------------- | ----------------: |
| `restore` | Restore file content / unstage                 |                No |
| `reset`   | Move HEAD / unstage / rewrite local history    |               Can |
| `revert`  | Create a new commit that undoes another commit |                No |

### `git restore`

Unstage:

```bash
git restore --staged file.txt
```

Discard working-tree modification:

```bash
git restore file.txt
```

### `git reset`

Unstage everything:

```bash
git reset
```

Move branch back one commit while keeping changes staged:

```bash
git reset --soft HEAD~1
```

Move branch back and unstage changes:

```bash
git reset HEAD~1
```

Dangerous:

```bash
git reset --hard HEAD~1
```

### `git revert`

Undo a pushed commit safely:

```bash
git revert abc1234
```

This creates a new commit that reverses the earlier change.

### Interview question

**Q: I accidentally pushed a bad commit to `main`. What do you use?**

Better answer:

```bash
git revert <commit>
```

rather than rewriting the shared branch with:

```bash
git reset --hard
git push --force
```

---

# 4. `git pull` vs `git pull --rebase`

### Normal

```bash
git pull
```

Typically:

```text
fetch + merge
```

### Rebase

```bash
git pull --rebase
```

Typically:

```text
fetch + rebase
```

Example:

```text
Remote:
A---B---C

Local:
A---B---D---E
```

After pull with merge:

```text
A---B---C------M
     \        /
      D---E--
```

After pull with rebase:

```text
A---B---C---D'---E'
```

A current interview guide explicitly lists fetch/pull differences and branch rebase scenarios as practical DevOps interview topics. ([persomentor.com][4])

---

# 5. `git checkout` vs `git switch` vs `git restore`

Modern Git separates these responsibilities.

### Switch branches

```bash
git switch main
```

Create and switch:

```bash
git switch -c feature-login
```

### Restore files

```bash
git restore file.txt
```

Older Git commonly used:

```bash
git checkout main
git checkout -- file.txt
```

For modern interviews, know both, but understand the newer commands.

---

# 6. `git stash` vs `git commit`

| stash                                | commit                   |
| ------------------------------------ | ------------------------ |
| Temporary storage                    | Permanent history entry  |
| Usually for unfinished work          | Completed logical change |
| Not normally part of project history | Becomes project history  |

Example:

You changed files but need to quickly switch branches:

```bash
git stash
git switch main
```

Later:

```bash
git switch feature
git stash pop
```

Don't use stash as your long-term backup strategy.

---

# 7. `git fetch` vs `git fetch --prune`

Normal:

```bash
git fetch origin
```

Prune obsolete remote-tracking branches:

```bash
git fetch --prune
```

Suppose someone deleted:

```text
origin/old-feature
```

Your local remote-tracking reference can remain stale.

Use:

```bash
git fetch --prune
```

to clean obsolete remote-tracking references.

---

# 8. `git branch` vs `git switch`

```bash
git branch feature
```

Creates a branch.

```bash
git switch feature
```

Switches to it.

```bash
git switch -c feature
```

Creates and switches.

Very common interview trick:

**Q: Does `git branch feature` switch to that branch?**

**Answer:** No.

---

# 9. `git log` vs `git reflog`

### `git log`

Shows commit history reachable from refs.

```bash
git log --oneline --graph --decorate --all
```

### `git reflog`

Shows movements of local references such as `HEAD`.

```bash
git reflog
```

This is extremely useful when you accidentally do:

```bash
git reset --hard
```

or accidentally delete a branch.

---

# 10. Hard interview question: recover a deleted commit

Suppose:

```bash
git reset --hard HEAD~3
```

and you realize it was wrong.

Check:

```bash
git reflog
```

Find the previous HEAD:

```text
abc123 HEAD@{0}: reset: moving to HEAD~3
def456 HEAD@{1}: commit: Add Kubernetes guide
```

Recover:

```bash
git reset --hard def456
```

This is the kind of practical recovery topic that appears in current Git interview material. ([LastRound AI][1])

---

# 11. `git cherry-pick`

Very useful in DevOps.

Suppose a bug fix exists in another branch:

```text
main:       A---B
                  \
hotfix:            C
```

You want only commit `C` on `main`.

```bash
git switch main
git cherry-pick <commit-id>
```

Result:

```text
A---B---C'
```

You did not merge the whole branch.

### Interview question

**Q: When would you use cherry-pick?**

Answer:

> When I need a specific commit from another branch without bringing the entire branch history into my current branch.

---

# 12. `git tag` vs branch

| Tag                                    | Branch                         |
| -------------------------------------- | ------------------------------ |
| Usually marks a specific point/version | Moves as new commits are added |
| Common for releases                    | Common for development         |
| `v1.0.0`                               | `main`, `develop`, `feature-x` |

Example:

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

# 13. `git clone` vs `git init`

### Clone

Existing remote repository:

```bash
git clone https://github.com/user/project.git
```

### Init

Create Git repository from an existing local directory:

```bash
mkdir project
cd project
git init
```

Then connect remote:

```bash
git remote add origin <url>
```

---

# 14. `git remote` vs `git branch -r`

### Remote

```bash
git remote -v
```

Shows remote repositories.

### Remote branches

```bash
git branch -r
```

Shows remote-tracking branches.

Example:

```text
origin/main
origin/develop
origin/feature-login
```

---

# 15. Easy interview questions

### Q1. What is Git?

Git is a distributed version-control system used to track source-code changes and collaborate on projects.

### Q2. What is GitHub?

GitHub is a hosting and collaboration platform for Git repositories.

### Q3. What is a repository?

A repository contains project files plus Git's version history.

### Q4. What is a commit?

A commit is a recorded snapshot of staged changes.

### Q5. What does `git add` do?

It stages changes for the next commit.

### Q6. What does `git status` show?

It shows the current branch and working-tree/staging state.

### Q7. What does `git clone` do?

It creates a local copy of a remote repository.

### Q8. What does `git push` do?

It sends local commits to a remote repository.

### Q9. What does `git pull` do?

It fetches remote changes and integrates them into the current branch.

### Q10. What is `origin`?

Usually the default name assigned to the remote repository created by clone.

---

# 16. Medium interview questions

### Q11. Difference between fetch and pull?

Answer:

```text
fetch = download remote updates
pull  = fetch + integrate
```

### Q12. Why use a branch?

To isolate work from another line of development.

### Q13. What is a merge conflict?

It occurs when Git cannot automatically reconcile competing changes.

### Q14. How do you resolve a conflict?

Typical process:

```bash
git status
```

Open conflicting files and resolve markers:

```text
<<<<<<< HEAD
your code
=======
their code
>>>>>>> branch
```

Then:

```bash
git add <file>
git commit
```

For a rebase conflict:

```bash
git add <file>
git rebase --continue
```

### Q15. How do you cancel a merge?

```bash
git merge --abort
```

### Q16. How do you cancel a rebase?

```bash
git rebase --abort
```

### Q17. How do you see unstaged changes?

```bash
git diff
```

### Q18. How do you see staged changes?

```bash
git diff --staged
```

### Q19. How do you rename a tracked file?

```bash
git mv old.txt new.txt
```

### Q20. How do you undo the last commit but keep changes?

```bash
git reset --soft HEAD~1
```

---

# 17. Hard interview questions

## Q21. Your local branch and remote branch have diverged. What do you do?

First inspect:

```bash
git status
git fetch origin
git log --oneline --graph --decorate --all
```

Then choose the project-approved strategy.

Merge:

```bash
git merge origin/main
```

or rebase your local work:

```bash
git rebase origin/main
```

Do not blindly force-push.

---

## Q22. A developer asks you to force-push to `main`. What do you do?

Good answer:

> I first verify branch protection and team policy. I avoid force-pushing shared production branches. If history rewriting is genuinely required, I prefer `--force-with-lease` over `--force` and make sure the team understands the impact.

Command:

```bash
git push --force-with-lease
```

---

# 18. Very hard: `--force` vs `--force-with-lease`

### Dangerous

```bash
git push --force
```

It can overwrite remote history.

### Safer

```bash
git push --force-with-lease
```

This adds a check that helps prevent overwriting remote work you haven't seen.

Interview question:

**Why is `--force-with-lease` preferred?**

Answer:

> Because it verifies that the remote branch is still at the expected state before replacing its history.

---

# 19. Very hard: production hotfix

Interview scenario:

> Production is broken. A fix exists in a feature branch, but that branch contains five unrelated commits. How do you bring only the fix?

Answer:

```bash
git switch main
git pull
git cherry-pick <hotfix-commit>
git push
```

This is a practical DevOps scenario rather than a definition question.

---

# 20. Very hard: accidental secret committed

Scenario:

```bash
git add .
git commit -m "Add configuration"
git push
```

You discover an API key was committed.

### First action

Revoke/rotate the exposed secret.

Then remove it from the repository/history using the project's approved history-rewrite/removal process.

Important interview point:

> Removing the secret from the latest file is not enough if the secret remains in Git history.

---

# 21. Very hard: wrong branch

Scenario:

You accidentally committed directly to `main`, but the commit should be on a feature branch.

One approach:

```bash
git switch -c feature-login
```

Then move `main` back appropriately, depending on whether the commit has already been pushed and whether rewriting shared history is allowed.

For an unpushed commit, another common pattern is:

```bash
git branch feature-login
git reset --hard HEAD~1
git switch feature-login
```

For a pushed shared branch, a different strategy may be safer. The key interview skill is recognizing that **local unpushed history and shared pushed history should be handled differently**.

---

# 22. Very hard: Git repository is huge

Interview question:

> The repository has become very large and cloning takes 20 minutes. What would you investigate?

Possible areas:

```text
large binaries
Git history
generated files
artifacts
logs
build output
incorrectly tracked dependencies
```

Useful commands:

```bash
git count-objects -vH
git rev-list --objects --all
```

For modern repositories, history-cleanup tools and Git LFS may be considered depending on the type of data.

---

# 23. Very hard: find the commit that introduced a bug

Use:

```bash
git bisect start
git bisect bad
git bisect good <known-good-commit>
```

Git checks candidate commits.

You test:

```bash
git bisect good
```

or:

```bash
git bisect bad
```

Eventually Git identifies the likely first bad commit.

Finish:

```bash
git bisect reset
```

This is an excellent senior-level interview topic.

---

# 24. Practical DevOps interview scenario

### Question

You log into your server:

```bash
cd /root/floci/devops
```

Someone else has pushed changes to GitHub.

You have local modifications.

What do you do?

### Bad answer

```bash
git pull
```

immediately.

### Better approach

```bash
git status
git diff
git stash
git fetch origin
git log --oneline --decorate --graph --all
```

Then decide whether to:

```bash
git pull --rebase
```

or use another integration strategy.

Then restore your work:

```bash
git stash pop
```

Resolve conflicts if necessary.

This demonstrates actual Git reasoning rather than memorized commands.

---

# 25. Another real interview scenario

### Question

Your CI/CD pipeline says:

```text
your branch is behind origin/main
```

What do you check?

```bash
git fetch origin
git status
git log HEAD..origin/main --oneline
```

This tells you what commits exist remotely but not locally.

Then, depending on your workflow:

```bash
git rebase origin/main
```

or:

```bash
git merge origin/main
```

---

# 26. Another common scenario

### Question

You deleted a local branch:

```bash
git branch -D feature-login
```

Can you recover it?

Often yes, provided the commits haven't been garbage-collected.

Check:

```bash
git reflog
```

Find the commit:

```bash
git switch -c feature-login <commit-id>
```

---

# 27. Interview comparison cheat sheet

| Interview comparison      | Key answer                                          |
| ------------------------- | --------------------------------------------------- |
| fetch vs pull             | fetch downloads; pull integrates                    |
| pull vs pull --rebase     | merge-style integration vs rebase-style integration |
| merge vs rebase           | preserve topology vs rewrite/replay commits         |
| reset vs revert           | rewrite/move local history vs new undo commit       |
| restore vs reset          | file/staging restoration vs ref/history movement    |
| stash vs commit           | temporary work vs project history                   |
| branch vs tag             | moving development pointer vs fixed release marker  |
| clone vs init             | copy existing repo vs create new local repo         |
| cherry-pick vs merge      | one/few commits vs entire branch integration        |
| force vs force-with-lease | unconditional rewrite vs guarded rewrite            |
| log vs reflog             | reachable history vs local ref movement             |
| fetch vs fetch --prune    | fetch updates vs also remove stale remote refs      |

---

# 28. My recommended Git interview command list

For a DevOps interview, become comfortable with these:

```bash
git clone
git init
git status
git add
git commit
git push
git pull
git fetch
git fetch --prune
git branch
git switch
git merge
git rebase
git cherry-pick
git stash
git restore
git reset
git revert
git reflog
git log
git diff
git show
git blame
git bisect
git tag
git remote
git clean
git rm
git mv
```

The official Git reference covers these command families and the current documentation is the best primary reference for exact behavior. ([Git SCM][5])

# 29. Current sites for Git/DevOps interview practice

I found these useful current resources:

**Official Git documentation**
Best for exact command behavior and interview follow-up questions. ([Git SCM][5])
[Git Reference](https://git-scm.com/docs?utm_source=chatgpt.com)

**GitHub Skills**
Interactive Git/GitHub courses, including an Introduction to Git course. ([GitHub Skills][6])
[GitHub Skills](https://skills.github.com/?utm_source=chatgpt.com)

**GitHub DevOps interview questions repository**
This current repository advertises 115 DevOps/SRE interview questions covering Git, CI/CD, Kubernetes, Docker, Linux, networking, security and AWS, including hands-on scenarios. ([GitHub][7])
[DevOps Interview Questions on GitHub](https://github.com/devops-interviews/devops-interview-questions?utm_source=chatgpt.com)

**LastRoundAI Git interview questions**
Current 2026 material with Git object model, merge/rebase, reset/revert, stash, remotes, conflicts, reflog and bisect. ([LastRound AI][1])

**Prepare.sh DevOps interview questions**
Includes scenario-based Git tasks such as rebasing feature branches and changing HTTPS remotes to SSH. ([prepare.sh][8])

**PersoMentor Git + CI/CD questions**
Useful for entry-level DevOps preparation, particularly fetch/pull/clone and Git/CI/CD fundamentals. ([persomentor.com][4])

---

## 30. How I would prepare you for a DevOps interview

Since you're building a real `devops` GitHub repository, don't just memorize definitions. Practice each topic in your server:

```text
LEVEL 1
│
├── clone
├── status
├── add
├── commit
├── push
├── pull
└── branch

LEVEL 2
│
├── fetch
├── merge
├── rebase
├── stash
├── restore
├── reset
├── revert
└── conflicts

LEVEL 3
│
├── cherry-pick
├── reflog
├── bisect
├── force-with-lease
├── remote tracking
├── history recovery
└── production hotfix scenarios
```

The **most important interview comparisons to master first** are:

```text
fetch       vs pull
merge       vs rebase
reset       vs revert
restore     vs reset
stash       vs commit
cherry-pick vs merge
force       vs force-with-lease
log         vs reflog
```

These are much more valuable in a DevOps interview than memorizing isolated Git commands. Current 2026 interview material also increasingly emphasizes hands-on scenarios rather than only "what is Git?" definitions. ([LastRound AI][1])

[1]: https://lastroundai.com/interview-questions/git?utm_source=chatgpt.com "Git Interview Questions (2026) | LastRoundAI"
[2]: https://git-scm.com/docs/git-fetch.html?utm_source=chatgpt.com "Git - git-fetch Documentation"
[3]: https://git-scm.com/docs/git-branch.html?utm_source=chatgpt.com "Git - git-branch Documentation"
[4]: https://persomentor.com/blog/devops-interview-git-cicd?utm_source=chatgpt.com "Git and CI/CD Interview Questions for DevOps Engineers (0–2 Years) 2026 | PersoMentor"
[5]: https://git-scm.com/docs?utm_source=chatgpt.com "Git - Reference"
[6]: https://skills.github.com/?fs=e&mibextid=wACSiI&s=cl&utm_source=chatgpt.com "GitHub Learn - Skills"
[7]: https://github.com/devops-interviews/devops-interview-questions?utm_source=chatgpt.com "GitHub - devops-interviews/devops-interview-questions: DevOps Interview Questions · GitHub"
[8]: https://prepare.sh/interview-questions/devops?utm_source=chatgpt.com "100+ DevOps Interview Questions with Answers (2026) | Prepare.sh"
