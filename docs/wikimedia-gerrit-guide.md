# Wikimedia Gerrit — Complete Reference for MediaWiki Contribution

> Self-contained reference for every Gerrit-specific task needed to contribute to MediaWiki (core, extensions, skins, Wikimedia operational repos). Scope is **only Gerrit**: the code-review tool, its workflows, its web UI, its CLI, and its policies. Out of scope (handle separately): MediaWiki coding conventions, local dev environment (MediaWiki-Docker), PHPUnit, task discovery.
>
> An assistant with this file should be able to perform any Gerrit action end-to-end without further context. All claims derive from the MediaWiki wiki pages cited inline; every primary source is linked so anything ambiguous can be re-checked.

---

## 0. Mental Model — What Gerrit Actually Is

Source: https://www.mediawiki.org/wiki/Gerrit/How_Gerrit_works

- Gerrit is a **Git server + code-review web UI** at **https://gerrit.wikimedia.org/r/**. Wikimedia hosts MediaWiki core, every extension, every skin, `operations/puppet`, Pywikibot, design/Codex, and many more repos here.
- You interact with it using a **plain Git client** — there is no Gerrit client to install. The web UI is for reviewing, commenting, voting, and merging.
- The **`refs/for/<branch>`** ref is Gerrit's magic. Pushing a commit to `refs/for/master` does NOT update `master`. It creates a **Change** — a review request visible in the web UI. Each such push either creates a new change or adds a new **patch set** to an existing change.
- A **Change** = Change-Id + metadata (owner, project, target branch) + one or more patch sets + comments + votes. Only the *latest* patch set is what gets merged. All older patch sets are archived for diff viewing.
- The **Change-Id** (a line like `Change-Id: Ibd3be19ed1a23c8638144b4a1d32f544ca1b5f97` starting with capital `I`) is how Gerrit links repeated pushes to the same Change. It is auto-inserted by a git hook.
- Wikimedia considered migrating to GitLab but **as of June 2024 this is not happening**. Gerrit is the canonical system.
- The GitHub org **https://github.com/wikimedia** is a **read-only mirror**. Pull requests there are not accepted.

**Vote model (the "+2"):** anyone can give nonbinding `+1`. Binding `+2` ("submit/merge") is restricted to Gerrit groups (see §11). On most Wikimedia repos, Jenkins automatically merges a change once it has `Code-Review +2` and passing CI.

---

## 1. Canonical URLs

| Purpose | URL |
|---|---|
| Gerrit home / your dashboard | https://gerrit.wikimedia.org/r/ |
| Browse / list repositories | https://gerrit.wikimedia.org/g/ |
| Repo admin list | https://gerrit.wikimedia.org/r/admin/repos/ |
| Your account settings | https://gerrit.wikimedia.org/r/settings/ |
| Notifications settings | https://gerrit.wikimedia.org/r/settings/#Notifications |
| REST API base | https://gerrit.wikimedia.org/r/ (endpoints under this) |
| Upstream Gerrit user docs | https://gerrit.wikimedia.org/r/Documentation/intro-user.html |
| Upstream search reference | https://gerrit.wikimedia.org/r/Documentation/user-search.html |
| Wikimedia developer account (LDAP) | https://idm.wikimedia.org/ |
| Main Gerrit wiki page | https://www.mediawiki.org/wiki/Gerrit |
| Full tutorial | https://www.mediawiki.org/wiki/Gerrit/Tutorial |
| Short tutorial (tl;dr) | https://www.mediawiki.org/wiki/Gerrit/Tutorial/tl;dr |
| Web-UI-only tutorial | https://www.mediawiki.org/wiki/Gerrit/Web_tutorial |
| How Gerrit works | https://www.mediawiki.org/wiki/Gerrit/How_Gerrit_works |
| Commit message guidelines | https://www.mediawiki.org/wiki/Gerrit/Commit_message_guidelines |
| git-review tool docs | https://www.mediawiki.org/wiki/Gerrit/git-review |
| Alternatives to git-review | https://www.mediawiki.org/wiki/Gerrit/Alternatives_to_git-review |
| Advanced usage | https://www.mediawiki.org/wiki/Gerrit/Advanced_usage |
| Navigation (search queries, CLI, REST) | https://www.mediawiki.org/wiki/Gerrit/Navigation |
| Troubleshooting | https://www.mediawiki.org/wiki/Gerrit/Troubleshooting |
| Getting your code reviewed | https://www.mediawiki.org/wiki/Gerrit/Code_review/Getting_reviews |
| Performing code review | https://www.mediawiki.org/wiki/Gerrit/Code_review |
| Privilege policy (+2 merge rights) | https://www.mediawiki.org/wiki/Gerrit/Privilege_policy |
| Creating new repositories | https://www.mediawiki.org/wiki/Git/Creating_new_repositories |
| Maintainers directory | https://www.mediawiki.org/wiki/Developers/Maintainers |
| Reviewer Bot registration | https://www.mediawiki.org/wiki/Git/Reviewers |
| Git & Gerrit FAQ | https://www.mediawiki.org/wiki/Git_and_Gerrit_FAQ |
| Patch Uploader (web tool, no Git) | https://gerrit-patch-uploader.toolforge.org/ |
| Test instance (practice safely) | https://gerrit-test.wmcloud.org/ |
| Phabricator (bug tracker) | https://phabricator.wikimedia.org/ |
| Jenkins / Zuul CI dashboard | https://integration.wikimedia.org/zuul/ |
| Code Review Office Hours | https://www.mediawiki.org/wiki/Code_Review/Office_Hours |
| Patch board (stuck patches) | https://www.mediawiki.org/wiki/Code_review/patch_board |
| TortoiseGit (Windows GUI) tutorial | https://www.mediawiki.org/wiki/Gerrit/TortoiseGit_tutorial |

**Protocol endpoints:**

- SSH: `ssh://<username>@gerrit.wikimedia.org:29418/<repo>.git` (port **29418**)
- HTTPS: `https://gerrit.wikimedia.org/r/<repo>.git`

---

## 2. One-Time Account & Machine Setup

Source: https://www.mediawiki.org/wiki/Gerrit/Tutorial (Prepare to work with Gerrit section)

### 2.1. Install Git

- **Debian/Ubuntu:** `sudo apt-get install git`
- **Fedora/RHEL:** `sudo dnf install git`
- **macOS:** pre-installed on 10.9+; else `brew install git`
- **Windows:** Git for Windows from https://git-scm.com/ (provides Git Bash). For a GUI, see TortoiseGit tutorial above.

Identity — must match the email on your Wikimedia developer account:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 2.2. Create a Wikimedia developer account

1. Sign up at https://idm.wikimedia.org/ (see *Help:Create a Wikimedia developer account*).
2. Confirm email.
3. Your **shell username** (aka LDAP username) is the one used for SSH and Gerrit.

### 2.3. Generate an SSH key

Wikimedia Security Team (Aug 2021) recommends **ed25519**:

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

Default path `~/.ssh/id_ed25519` is fine; a dedicated name like `~/.ssh/id_wikimedia_gerrit` is also common. Choose a strong passphrase.

### 2.4. Upload the public key to Gerrit

Adding it only to `idm.wikimedia.org` is **not enough** — you must also add it in Gerrit itself.

1. Log in at https://gerrit.wikimedia.org/r/ (credentials = your developer account).
2. Avatar (top right) → **Settings** → **SSH Keys** (left menu).
3. Paste the contents of `~/.ssh/id_ed25519.pub` (or whichever pub file).
4. Click **ADD NEW SSH KEY**.

### 2.5. Test the SSH connection

```bash
ssh -p 29418 <username>@gerrit.wikimedia.org
```

Expected output ends with:

```
**** Welcome to Gerrit Code Review ****

Hi <username>, you have successfully connected over SSH.
Unfortunately, interactive shells are disabled.
To clone a hosted Git repository, use:
git clone ssh://gerrit.wikimedia.org:29418/REPOSITORY_NAME.git

Connection to gerrit.wikimedia.org closed.
```

Verify the host key fingerprint on first connect matches the one published on the Gerrit wiki. If it fails: `ssh -v -p 29418 <username>@gerrit.wikimedia.org` (or `-vv` / `-vvv`) and see §10.

### 2.6. Optional SSH alias

Add to `~/.ssh/config`:

```
Host gerrit
    Hostname gerrit.wikimedia.org
    Port 29418
    User <username>
    IdentityFile ~/.ssh/id_ed25519
```

Then `ssh gerrit` or clone `ssh://gerrit/<repo>.git` works.

### 2.7. Install `git-review`

`git-review` is a thin Python wrapper that: pushes to the right `refs/for/<branch>`, installs the `commit-msg` hook that adds `Change-Id`, and makes downloading/amending existing changes trivial. **Strongly recommended** — manual Gerrit usage is a pain.

- **Debian/Ubuntu:** `sudo apt-get install git-review` (apt version is usually slightly older than pip's; either is fine)
- **Fedora/RHEL:** `sudo dnf install git-review` (enable EPEL if needed)
- **Arch:** AUR package `git-review` (`git clone https://aur.archlinux.org/git-review.git && cd git-review && makepkg -si`)
- **NixOS:** `git-review` in nixpkgs
- **Any OS with pip:** `pip install git-review` (or `pipx install git-review`)
- **Windows:** install Python from python.org, then run Git Bash as Administrator and `pip install git-review`

Verify: `git review --version`.

### 2.8. Configure `git-review` default remote

Wikimedia convention: the Gerrit remote should be called `origin` (not the default `gerrit`). Create `~/.config/git-review/git-review.conf`:

```ini
[gerrit]
defaultremote=origin
```

Alternatively (modern style, per-repo): put `defaultremote=origin` in each repo's `.gitreview` file, since the global config path is now deprecation-warned in newer `git-review`.

### 2.9. (Optional) Force all Gerrit URLs to use SSH

```bash
git config --global url."ssh://<username>@gerrit.wikimedia.org:29418/".insteadOf "https://gerrit.wikimedia.org/r/"
```

This rewrites any HTTPS Gerrit URL to SSH on the fly, so copy-pasted HTTPS clone commands work with your key.

---

## 3. Cloning

Find clone URLs on any repo's page, e.g. https://gerrit.wikimedia.org/g/mediawiki/core.

| Repo | URL |
|---|---|
| MediaWiki core | `mediawiki/core.git` |
| Extension (example) | `mediawiki/extensions/VisualEditor` |
| Skin (example) | `mediawiki/skins/Vector` |
| Pywikibot | `pywikibot/core` |
| Sandbox (for tutorial practice) | `sandbox` |
| Operations/puppet | `operations/puppet` (branch `production`, not `master`) |
| Codex design system | `design/codex` |

**If you plan to push patches, use SSH:**

```bash
git clone ssh://<username>@gerrit.wikimedia.org:29418/mediawiki/core.git mediawiki
```

Or clone HTTPS then switch the remote:

```bash
git clone https://gerrit.wikimedia.org/r/mediawiki/core.git mediawiki
cd mediawiki
git remote set-url origin ssh://<username>@gerrit.wikimedia.org:29418/mediawiki/core
```

**After cloning, always run:**

```bash
cd mediawiki
git review -s
```

This installs the `commit-msg` hook (which adds `Change-Id:` to commits) and sets up the remote. Without it, `git push` will be rejected with `missing Change-Id in commit message footer`.

**Check the default branch name.** Most repos still use `master`, but some newer/renamed ones use `main`, and `operations/puppet` uses `production`. Verify via `.gitreview`'s `defaultbranch=` or `git branch -r`.

If the repo doesn't use `master`, tell `git-review` about it:

```bash
git config gitreview.branch main     # or whatever the branch is
```

---

## 4. The Standard Contribution Workflow

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Tutorial — shorter: https://www.mediawiki.org/wiki/Gerrit/Tutorial/tl;dr

### 4.1. Create a branch from up-to-date main

```bash
git checkout master         # or: main, production — whichever this repo uses
git pull
git checkout -b meaningful-branch-name
```

### 4.2. Edit, stage, test, commit

Make your changes. Run the project's tests locally. Then:

```bash
git add <files>        # or: git commit -a   to stage all tracked changes
git commit             # editor opens for commit message
```

**One logical change = one commit.** If you made multiple WIP commits locally, squash them before pushing (see §6.2). Do `git commit` without `--amend` **only once per branch**; subsequent edits in response to review use `--amend` (§4.7).

### 4.3. Write a compliant commit message

Full rules in §5. Minimal template:

```
component: Short imperative subject, no period, ≤~70 chars

Longer explanation of *why* this change exists. Wrap around 72 chars
(max 100). Do not put URLs or task IDs in the subject.

Bug: T12345
```

The `Change-Id:` footer is inserted automatically by the hook the first time you commit. Don't type it yourself.

### 4.4. Rebase onto latest upstream

```bash
git pull --rebase origin master
```

This temporarily sets aside your changes, applies new upstream commits, then re-applies yours on top. Avoids merge conflicts later and ensures you've tested against current code.

### 4.5. Push to Gerrit

```bash
git review
```

Pushes to `refs/for/<defaultbranch>` from `.gitreview`. Success looks like:

```
remote: New Changes:
remote:   https://gerrit.wikimedia.org/r/c/<project>/+/<number> <subject>
remote:
To ssh://gerrit.wikimedia.org:29418/<project>.git
 * [new branch]      HEAD -> refs/for/master
```

Open the URL — this is your **Change page** (dashboard for this review).

**If `git-review` is unavailable,** push manually:

```bash
git push origin HEAD:refs/for/master
```

…and install the hook yourself if missing:

```bash
scp -p -P 29418 <username>@gerrit.wikimedia.org:hooks/commit-msg .git/hooks/
```

### 4.6. Add reviewers

On the Change page:

1. Under **Reviewers**, click **Add reviewer** → add 1–2 maintainers. Find them via https://www.mediawiki.org/wiki/Developers/Maintainers, the extension's page on mediawiki.org, or `git log` of the files you touched.
2. Click **Add to attention set** for each so the change appears in their "Your turn" queue.

Do not wait passively — adding reviewers is your responsibility. Experienced contributors may also add reviewers to patches they notice lingering.

### 4.7. Respond to review — ALWAYS amend, never add new commits

To update the same Change:

```bash
# ...edit files to address review feedback...
git add <files>
git commit --amend          # KEEP the Change-Id line intact
git review                  # uploads Patch Set 2 (or 3, 4, …) of the same Change
```

Why amend? Because Gerrit keys changes by `Change-Id`. An amended commit pushed with the same Change-Id = new patch set of the same review. A brand-new commit = a brand-new review.

If you accidentally create a new commit: use `git rebase -i origin/master` to squash them, keeping the correct Change-Id (the one of the change you wanted to update).

### 4.8. Rebase during review (if master moved or Gerrit reports a conflict)

Easiest: on the Change page, click **Rebase** → keep default "Rebase on top of the master branch" → **Rebase**. For merge conflicts, check "Allow rebase with conflicts" to amend with conflict markers you'll resolve.

CLI version:

```bash
git review -d <change-number>
git rebase origin/master
# resolve conflicts: edit, git add, git rebase --continue
git review
```

If you prefer to keep rebases as separate patches for reviewer clarity, you can do so — but most people just let Gerrit's Rebase button do it.

### 4.9. Merging

- **You do not merge your own change** unless you hold +2 on the repo. A maintainer with +2 clicks **Submit** (or Jenkins auto-merges on +2).
- Before merge, Jenkins (CI) must be green — `Verified +1` — or the maintainer must manually override.
- After merge, `Bug: Txxxxx` trailers trigger bot comments on the linked Phabricator task.

---

## 5. Commit Message Guidelines (complete rules)

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Commit_message_guidelines

### 5.1. Structure

```
<subject: imperative, ≤~70 chars, no trailing period, no URL/task>
                                                                    ← blank line
<body: explain what + why (not how). Wrap at 72–100 chars. Multiple
paragraphs OK. Don't break URLs to fit the wrap — put long URLs on
their own line.>
                                                                    ← blank line
Bug: T12345
Change-Id: I0123456789abcdef0123456789abcdef01234567
```

### 5.2. Subject line rules

- **Imperative mood.** "Fix X", not "Fixed X" / "Fixes X" / "Fixing X".
- **No trailing period.**
- **No URLs, task IDs, or commit hashes.** The subject appears in plain-text contexts (email, IRC, `git log --oneline`, GitHub mirror commit list) where those aren't clickable and can't always be copied. Put such references in the body/footer.
- Optional component prefix: `resourceloader: Cache module dependencies`.
- Prefix with `[WIP]` for work-in-progress.

### 5.3. Body rules

- Wrap at **≤100 chars**; 72 is the common industry convention.
- Don't break URLs — long URLs go on their own line.
- Explain **motivation + context**, not the diff. Don't just restate what changed; explain *why*.
- Don't use a URL as the sole explanation — summarize the key point in the body, *then* link the URL.

### 5.4. Footer trailers

Trailers live at the bottom, one per line, no blank line between `Bug:` and `Change-Id:` if amending.

- **`Bug: T12345`** — links the change to a Phabricator task. A bot comments on the task on push/merge/abandon. For multiple bugs, use one line each:
  ```
  Bug: T12345
  Bug: T23456
  ```
  Use **`Txxxxx`** form only, not the full URL. (Gerrit+Phab integration doesn't parse URL form.)
- **`Change-Id: I<40 hex>`** — **always the last line**, starts with capital `I`. Auto-inserted by the `commit-msg` hook. Preserve it verbatim across amendments.
- **`Co-Authored-By: Name <email>`** — acknowledges pair-programming / second author.
- **`Depends-On: I<change-id>`** — if another unmerged Change must merge first (used by mediawiki-config and operations/puppet automation).

### 5.5. Referring to other commits

Inside the body, refer to another commit by:

- **Short Gerrit Change-Id** (e.g. `I83f83377f2`) for changes not yet merged — the SHA-1 changes on rebase, so using the Git hash would become a dead link.
- **Short Git SHA-1** (e.g. `51e3fb9a71`) for already-merged commits.

**Never** use a full URL or the Gerrit change number. Change numbers can collide with commit hashes (`665661` the change vs `665661` the commit prefix).

### 5.6. Use a template if the repo has one

```bash
git config commit.template .gitmessage     # if .gitmessage exists in repo root
```

### 5.7. Good-example template (copy & modify)

```
skin: Avoid double-escaping edit link text

The EditLink::getLabel() method previously HTML-escaped text that had
already been escaped by the caller, producing &amp;amp; in user-facing
output. This removes the inner escape and adds a test case for the
regression.

Bug: T384712
Change-Id: Iabc123…
```

---

## 6. Common Everyday Workflow Tasks

### 6.1. Download someone else's patch (to review or amend locally)

```bash
git review -d 12345              # 12345 = Gerrit change number from URL
# You're now on branch review/<author>/<topic>
```

- `git review -x 12345` — cherry-picks into current branch instead.
- **Amending someone else's patch** is restricted. You must be in the **Trusted-Contributors** group (or own the repo). Membership is viral — any current member can add you.

### 6.2. Squash multiple local commits into one

```bash
git rebase -i origin/master
# change "pick" to "squash" (or "s") for all but the first commit
# in the combined message, keep exactly ONE Change-Id:
#   - if updating an existing Gerrit change: the Change-Id of THAT change
#   - if creating a new change: any of them is fine
```

### 6.3. Mark a patch as Work-In-Progress

```bash
git review --work-in-progress     # or --wip
```

Also prefix subject with `[WIP]`. WIP changes are visible but do not notify reviewers and can be filtered with `-is:wip` in searches. Reviewers' dashboards usually hide WIP patches.

### 6.4. Push to a non-default branch

```bash
git review REL1_40                              # via git-review
# or manually:
git push origin HEAD:refs/for/REL1_40
```

### 6.5. Set a topic (groups related changes together)

```bash
git review -t my-topic-name                     # single-string, no spaces
# or manually:
git push origin HEAD:refs/for/master%topic=my-topic-name
# (equivalent new-style: HEAD:refs/heads/master -o topic=my-topic-name)
```

Topic can also be set in the web UI's **Topic** field on the Change page.

### 6.6. Set hashtags (free-form tags, multiple per change)

In the web UI, on the Change page, there's a hashtag area. Search with `hashtag:my-tag`.

```bash
git push origin HEAD:refs/for/master%t=stable-bugfix
```

### 6.7. Abandon a change

Change page → **Abandon**. Removes it from review queues, keeps it archived. You can **Restore** later.

### 6.8. Cherry-pick a merged change to another branch (e.g. a release branch)

Web UI: on the merged Change page → **⋮ (More)** → **Cherry pick** → pick target branch → creates a new Change on that branch.

CLI: look up the merged commit SHA, then:

```bash
git checkout -b backport-X origin/REL1_40
git cherry-pick <sha>
git review REL1_40
```

If conflicts: resolve, `git add`, `git cherry-pick --continue`. **Leave the original `Change-Id` in the message intact** — add a new `Conflicts:` section *above* it if needed. Backports usually go to a specific topic like `refs/for/REL1_40%topic=my-topic`.

### 6.9. Push dependent patches (a chain)

Just stack commits locally and `git review` them together; Gerrit auto-detects the parent relationship from the Git history. Each becomes its own Change, with a parent/child link visible in the UI's "Relation chain" section.

To rebase a child onto an amended parent:

```bash
PARENT=424242
CHILD=424243
git fetch "https://gerrit.wikimedia.org/r/<project>" refs/changes/<NN>/${PARENT}/<PS> \
  && git checkout FETCH_HEAD
git branch merge_${PARENT}
git review -d ${CHILD}
git rebase -i merge_${PARENT}
git review
```

### 6.10. Fetch a specific patch set manually

Any patch set's fetch/checkout command is in the Change page under **Download → Checkout** (dropdown gives Checkout / Cherry-Pick / Format-Patch / Pull). Underlying ref format:

```
refs/changes/<last-two-digits-of-change-num>/<change-num>/<patch-set-num>
```

Example for change 12345, patch set 3:
```bash
git fetch origin refs/changes/45/12345/3 && git checkout FETCH_HEAD
```

### 6.11. Edit a small change entirely in the web UI

For tiny fixes (typos, i18n strings) you don't need to clone anything. Source: https://www.mediawiki.org/wiki/Gerrit/Web_tutorial

1. Go to https://gerrit.wikimedia.org/r/admin/repos/ (log in first).
2. Pick the repo → **General** tab → **Create Change**.
3. Fill in:
   - **Branch**: `master` (or the target branch).
   - **Topic**: e.g. `copy-edit` — must be one string, no spaces.
   - **Description** (commit summary): follow §5 rules.
4. Click **Create**.
5. On the new Change page → **Edit** (top right) → **Add/Open/Upload** under *Actions* next to the file → type path (e.g. `i18n/en.json`) → **Confirm**.
6. Edit the file inline → **Save**.
7. Click **Publish Edit** — now it's a normal Change open for review.

### 6.12. Submit without cloning: Patch Uploader

For even simpler submission: https://gerrit-patch-uploader.toolforge.org/ — upload a `.patch` / `.diff` and it pushes to Gerrit on your behalf.

---

## 7. Navigating Gerrit — Searching, Queries, Dashboards

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Navigation
Upstream operator reference: https://gerrit.wikimedia.org/r/Documentation/user-search.html

### 7.1. Common search operators

Type these in the top search box:

| Operator | Meaning |
|---|---|
| `status:open` / `status:merged` / `status:abandoned` | change state |
| `is:open` / `is:closed` / `is:wip` / `is:merged` / `is:abandoned` | same as above |
| `owner:<user>` / `owner:self` | author of the change |
| `reviewer:<user>` / `reviewer:self` | a specified reviewer |
| `ownerin:<group>` / `reviewerin:<group>` | by group membership |
| `project:<n>` / `project:^<regex>` | specific repo (regex with `^`) |
| `projects:<prefix>` | all repos with a prefix |
| `branch:<n>` | target branch |
| `topic:<n>` | designated topic |
| `hashtag:<tag>` | free-form hashtag |
| `label:Code-Review=+2` / `label:Verified>=1` | vote filters |
| `label:Code-Review<0` | any negative review |
| `has:attention` | in YOUR attention set — your primary queue |
| `has:unresolved` | has unresolved comment threads |
| `message:"..."` | full-text search of commit messages |
| `commentby:<user>` | someone commented |
| `-is:wip` | exclude WIP changes |
| `age:2w` / `-age:2w` | older / newer than N (d/w/mo) |

Compose with `AND`, `OR`, `NOT`, parens. Escape quotes in values with backslash-in-quotes: `message:"This \"is\" fixing a bug"`.

### 7.2. Useful ready-made queries

- **Your attention set** — `has:attention` (bookmark this; it's your single most useful URL)
- **Unreviewed open changes by new contributors, CI green:**
  `ownerin:newcomers status:open label:Verified>=1 -label:Code-Review<0`
- **Newcomers' changes with -1 needing guidance:**
  `ownerin:newcomers status:open label:Verified>=1 label:Code-Review<0`
- **Your own open patches:** `owner:self is:open`
- **Changes awaiting your review:** `reviewer:self is:open -is:wip`

### 7.3. Run queries from CLI (SSH)

```bash
ssh -p 29418 <username>@gerrit.wikimedia.org gerrit query \
  'status:open project:^mediawiki/.* AND NOT label:Code-Review<=-1'
```

Output includes `rowCount` at the end — handy for counts. Full command reference: https://gerrit.wikimedia.org/r/Documentation/cmd-query.html

### 7.4. REST API

Endpoints live under https://gerrit.wikimedia.org/r/ — e.g. `https://gerrit.wikimedia.org/r/changes/?q=status:open+owner:self`. Responses are prefixed with `)]}'` (XSSI guard) — strip the first line before parsing JSON. Upstream docs: https://gerrit.wikimedia.org/r/Documentation/rest-api.html

### 7.5. Dashboards & notifications

- **Settings → Preferences** lets you add custom queries to the dashboard menu.
- **Settings → Watched Projects** (or https://gerrit.wikimedia.org/r/settings/#Notifications) — subscribe to email notifications for new patch sets on repos/branches/patterns you maintain, with filters like "commit message contains" or "file matches".
- **Reviewer Bot** (https://www.mediawiki.org/wiki/Git/Reviewers) — register via wiki template to be auto-added as reviewer on changes matching project/file regex. Useful when you review only part of a repo (e.g. only CSS).

### 7.6. Helpful bookmarklets

Hide Jenkins bot comments (for cleaner human-review viewing):

```javascript
javascript:Array.from(document.querySelectorAll('[class*=messageBox]')).filter(box=>box.querySelector('[class*=name]').textContent==='jenkins-bot').forEach(box=>box.style.display='none')
```

The Change page's **"Only Comments"** toggle also hides bot noise.

---

## 8. Getting Your Code Reviewed (contributor perspective)

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Code_review/Getting_reviews

### 8.1. Pre-push checklist

- Changes follow the project's coding conventions (separate wiki pages — out of scope here).
- Local tests pass.
- For new features (not bugfixes), align with maintainers first on Phabricator/IRC **before** writing the code. Saves wasted work.
- A Phabricator task exists describing the motivation. Patches without `Bug:` trailers are harder to find reviewers for and harder to track.

### 8.2. Maximize review speed

- **Add 1–2 specific reviewers yourself** immediately after pushing. Do not wait for someone to stumble on it.
- **Small patches get merged faster.** If the change is large, split into a dependent chain.
- **Respond within 24–48 h** when you get feedback. Stale reviews bitrot — reviewers have to re-load context.
- **Amend trivial requests without argument.** Style quibbles aren't worth a debate; just fix and move on. Review is consensus, not voting.
- **Address negatives, don't outvote them.** Multiple `+1`s do not neutralize a `-1`. You're expected to resolve it.
- **Sometimes addressing a -1 just means explaining** — if the reviewer misunderstood, clarify in the commit message or code comments (not just a reply), so future maintainers with the same concern are answered.
- If CI fails: Change page → click the failed job link → in the Jenkins console, hover the red dot → "Console output". Fix → `git commit --amend` → `git review`.
- Enable email notifications in Gerrit settings so you don't miss reviews.

### 8.3. What vote numbers mean to you

| Vote | Meaning for the contributor |
|---|---|
| `Code-Review +2` | Approved — a maintainer will merge (or Jenkins auto-merges). |
| `Code-Review +1` | "Looks good to me, but I'm not a maintainer / I want a second opinion." Does not enable merge. |
| `Code-Review -1` | "Needs improvement." Read the comment; amend. |
| `Code-Review -2` | "Do not merge." Address the reason or abandon. |
| `Verified +1` / `-1` | Automatic from Jenkins CI (build/tests passed/failed). |
| No vote, just comments | Some reviewers give suggestions without a vote to be gentle on new contributors. Take them seriously. |

### 8.4. If your patch is stuck for weeks

- List it at https://www.mediawiki.org/wiki/Code_review/patch_board (one patch per person at a time; must be ready, CI-green, all -1s addressed).
- Join **Code Review Office Hours** on `#wikimedia-codereview` IRC: https://www.mediawiki.org/wiki/Code_Review/Office_Hours
- Ask on `wikitech-l` mailing list or the relevant team's IRC channel.

### 8.5. Etiquette

Be patient, grow your reputation with small good patches first, and take feedback graciously. Experienced patch writers receive faster *and* more positive reviews.

---

## 9. Performing Code Review (reviewer perspective)

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Code_review

### 9.1. Finding what to review

- **Primary queue: `has:attention`** — Gerrit auto-adds you when a new reviewer request comes in or when a comment is addressed to you.
- **Watched Projects** — subscribe in Settings → Notifications, filter by project/branch/file.
- **Reviewer Bot** — register in https://www.mediawiki.org/wiki/Git/Reviewers to be auto-added to matching changes.
- **Newcomer queues** — prioritize patches from contributors with ≤5 changesets. Delayed feedback is particularly discouraging to them.
- **Chronological / per-author / per-repo** — different strategies for different moods: a new-file review is best done by reading the whole file; a prolific author's backlog is best done chronologically.

### 9.2. What to check

Drawn from https://www.mediawiki.org/wiki/Gerrit/Code_review

- **Does it do what the commit message says?** Mismatch between description and diff is a red flag.
- **Does it address the linked Phabricator task?**
- **Style / conventions** — indentation, spacing, function naming (a `getX` function should `get` something; a `setX` should `set`), English identifiers, no obscure abbreviations.
- **Doc comments** on new/changed functions.
- **Simplicity** — overly clever code (nested ternaries, etc.) should be replaced by the longer, clearer form.
- **Security** — especially user-input handling, SQL, HTML escaping.
- **Tests** — new logic should have them; regressions should add a test that fails before and passes after.
- **Backwards compatibility** — for MediaWiki core, check the Stable interface policy and any `@stable` / `@unstable` annotations.
- **DB impact** — schema changes or heavy queries should be tagged `schema-change` on Phab and reviewed by a DBA.
- **UX impact** — consult `#wikimedia-design` or the design list if visual/UX changes exist.

### 9.3. How to vote

- **+2** — "I read it, I tested it (mentally or literally), I take responsibility for this going live." Requires your group membership allows +2 on this repo (see §11).
- **+1** — "Looks right; want a second pair of eyes or a maintainer +2."
- **-1** — "Good direction, needs changes." **Always** include a comment stating *what* is wrong and *how* to fix it. A bare -1 is bad form.
- **-2** — "Must not merge as-is." Reserved for fundamental objections; use sparingly.
- **No vote + comments** — valid pattern for encouraging new contributors without a scary red mark.

To submit the review: **Reply** button → write summary comment + vote → **Send**. Inline comments from your draft pass-through are attached.

### 9.4. Overriding Jenkins (rarely needed)

If Jenkins voted `Verified -2` but the failure is a known transient issue (e.g. network hiccup, infra flap) and you're a maintainer:

1. Click the `[x]` on the jenkins-bot `Verified` row to remove its vote.
2. Under **Reply**, give `Verified +2` alongside your `Code-Review +2`.

Document in a comment why you're overriding. Don't do this for real test failures.

### 9.5. Reviewer etiquette

- **Be nice.** Contributed patches are gifts. Tone shapes whether volunteers come back.
- **Be quick.** A fast -1 with a clear path forward is better than weeks of silence.
- If you can't review it, **say so** and remove yourself from reviewers, or suggest someone else via "Add to attention set".
- **Give feedback yourself** — don't silently rely on another reviewer to handle it.

---

## 10. Troubleshooting

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Troubleshooting

### 10.1. `missing Change-Id in commit message footer`

The hook isn't installed. Fix:

```bash
git review -s
# or manually:
gitdir=$(git rev-parse --git-dir)
scp -p -P 29418 <username>@gerrit.wikimedia.org:hooks/commit-msg ${gitdir}/hooks/
```

To retrofit the current commit after installing:

```bash
git commit --amend --no-edit
```

Gerrit's error message usually includes a suggested `Change-Id:` line you can paste into the commit message manually if you prefer.

### 10.2. `Unauthorized` / `Authentication failed for 'https://gerrit.wikimedia.org/r/…'`

You're pushing over HTTPS but SSH is what's set up. Switch the remote:

```bash
git remote set-url origin ssh://<username>@gerrit.wikimedia.org:29418/<repo>
```

### 10.3. `Permission denied (publickey)`

Typical causes:
- SSH key not in Gerrit (add at Settings → SSH Keys).
- Wrong key offered by SSH client — test with `ssh -v -p 29418 <username>@gerrit.wikimedia.org` and check which IdentityFile it tries.
- Key file permissions too open — `chmod 600 ~/.ssh/id_*`.
- Outdated key algorithm (pre-ed25519); see phab T276486.

### 10.4. `git review` complains about multiple (other people's) commits

Your local branch has drifted from remote. Fix:

```bash
# Answer "no" to abort the push
git fetch --all                # or: git remote update  (both equivalent)
git review
```

If your master has drifted heavily, hard-reset to the remote:

```bash
git checkout master
git fetch origin
git reset --hard origin/master
```

### 10.5. `Working tree is dirty`

Stage/commit or stash:

```bash
git add -A && git commit --amend --no-edit    # fold into current commit
# or:
git stash
```

### 10.6. Jenkins reports merge conflict after approval

Gerrit does not auto-resolve. Rebase (web UI Rebase button is easiest; CLI fallback):

```bash
git checkout master && git pull
git review -d <change#>
git rebase origin/master
# resolve conflicts → git add → git rebase --continue
git review
```

When editing the commit message after manual conflict resolution, place any `Conflicts:` section **above** the `Change-Id:` line. `Change-Id` must remain the last line or the push is rejected.

### 10.7. Email address mismatch on push

Gerrit rejects pushes where `user.email` doesn't match a verified email on your Gerrit account.

- If the email Git reports is the one you want: add it at Settings → Email Addresses and click the confirmation link it mails you. Re-push.
- If Git is using something else (stale `root@localhost`, old address):
  ```bash
  git config --global user.email yournewemail@example.org
  git commit --amend --reset-author
  git review
  ```

### 10.8. SSH hangs or port unreachable

- `ssh -v -p 29418 <username>@gerrit.wikimedia.org` for verbose output.
- Check firewall/ISP isn't blocking port **29418**. If it is, you can clone over HTTPS (read-only for push unless you configure HTTP password).
- On Windows + PuTTY: Settings → Connection → SSH → Bugs → set *"Chokes on PuTTY's SSH-2 'winadj' requests"* to **On**.

### 10.9. `git review: command not found` after seemingly successful install

Don't just reinstall on top. Uninstall cleanly first:

```bash
pip uninstall git-review
pip install -U setuptools
pip install git-review
```

### 10.10. Working across multiple Gerrit accounts on one machine

Use an SSH config Host alias per account (§2.6) and per-repo `user.email` via `~/.gitconfig` `includeIf`:

```
[includeIf "gitdir:~/src/wikimedia/"]
    path = ~/.gitconfig-wikimedia
```

Where `~/.gitconfig-wikimedia` contains the `[user]` block for that identity.

---

## 11. +2 Rights, Groups, and Privilege Policy

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Privilege_policy

### 11.1. How +2 works

- Each repo has Gerrit groups that grant `+2` rights. Usually there's a group named after the repo.
- The **`mediawiki`** group grants `+2` on MediaWiki core + all deployed extensions + skins + several related repos.
- Most **WMF employees** are in the LDAP `wmf` group, which auto-maps into the Gerrit `mediawiki` group.
- On most repos, once a change has `Code-Review +2` and Jenkins passes, **Jenkins auto-merges**. The merge is essentially immediate.

### 11.2. What `+2` means (the social contract)

When you `+2`, you assert:

- "This follows MediaWiki conventions and good engineering practice."
- "The code does what the commit message says."
- "I take responsibility for this landing in production-adjacent branches."

For MediaWiki core + deployed extensions, a merged change reaches the **Beta Cluster** immediately and the next train deployment unless reverted. So a bad `+2` can cause real outages. +2 is a strong trust signal.

### 11.3. Rules on self-merging

- **Do not merge your own change** unless it falls into documented exceptions:
  - Pure reverts of recent commits.
  - Re-merging your own already-+2'd change when a transient Jenkins failure blocked the auto-merge.
  - Changes in deployment branches / puppet where the deployer doing the deploy is expected to self-merge.
  - New repo creation / permission bootstrapping.
- Merging your own code without someone else's review may cost you your `+2`.

### 11.4. Requesting `+2` (or other group membership)

Process:

1. File a task under the **Gerrit-Privilege-Requests** project on Phabricator: https://phabricator.wikimedia.org/tag/gerrit-privilege-requests/
2. For the `mediawiki` group specifically, use the **MediaWiki-Gerrit-Group-Requests** project.
3. Build consensus in the task — trusted developers weigh in.
4. A Gerrit admin executes once consensus exists.

Short-circuit paths:
- **WMF / WMDE new hires** may be added to LDAP groups (`wmf`, `wmde`) without a Phab task, per hiring process.
- **TechCom / CTO** may direct addition/removal.

### 11.5. Trusted-Contributors group (for amending others' patches)

Source: https://www.mediawiki.org/wiki/Gerrit/Tutorial

- Required to amend/push patch sets to a change you don't own (if you're not a +2-er on that repo).
- **Self-managed and viral** — any current member can add new members. Just ask someone you've worked with.

### 11.6. Revocation

- Emergency revocation (e.g. compromised account) — any Gerrit admin, at their discretion.
- Employee offboarding — the employee's manager, via WMF Talent & Culture.
- TechCom / CTO may direct.

---

## 12. Creating a New Repository (for maintainers/admins)

Authoritative: https://www.mediawiki.org/wiki/Git/Creating_new_repositories

Only **Project Creators** can do this.

```bash
ssh -p 29418 gerrit.wikimedia.org gerrit create-project \
  --require-change-id \
  --owner=MyGroup \
  --parent=mediawiki/extensions \
  --description='"My super cool extension"' \
  mediawiki/extensions/MyExtension
```

Rules:

- **Names are nearly impossible to change later.** Pick carefully.
- Allowed chars: `[a-z0-9-]` only.
- **Always set `--parent`** to inherit permissions from the logical parent (e.g. `mediawiki/extensions`). The default `All-Projects` is almost never right.
- **Always set `--owner`** to a group you belong to. If you don't, you can't edit its permissions later — you'd have to beg a Gerrit admin.
- New top-level prefixes (like `mediawiki/*` or `operations/*`) need prior discussion on `wikitech-l`. Don't create one unilaterally.

### After creation — add a `.gitreview` file

Required for extensions (or Translatewiki's l10n integration breaks):

```ini
[gerrit]
host=gerrit.wikimedia.org
port=29418
project=mediawiki/extensions/MyExtension.git
defaultbranch=master
defaultrebase=0
track=1
```

`host` and `port` never change. `project` must match what you created. `defaultrebase=0` disables `git-review`'s implicit pre-push rebase (most people find this less annoying).

After initial permissions are set, **remove** the `Project and Group Creators` group from Owner — they shouldn't retain rights once the real owners are in place.

### Deployed extensions — extra audit step

Before an extension is first deployed to Wikimedia cluster, audit permissions: downgrade any legacy "ordinary-developer repo-ownership" to +2 access only. Gerrit admins should not grant repo ownership to regular developers for deployed code.

---

## 13. Advanced Usage

Authoritative: https://www.mediawiki.org/wiki/Gerrit/Advanced_usage

Many of the old "hard way" tricks are obsolete now that `git-review` handles them. Read `man git review` first. Below are the ones still genuinely useful:

### 13.1. Backporting a merged change to a release branch

```bash
git fetch origin
git checkout -b backport-X origin/REL1_40
git cherry-pick <merged-sha>        # SHA found on the original Change page
# resolve conflicts if any → git add → git cherry-pick --continue
git review REL1_40                  # or: git push origin HEAD:refs/for/REL1_40%topic=backport-xyz
```

Keep the original `Change-Id` — Gerrit will correctly link the cherry-picked Change as a backport.

### 13.2. Fetching commit notes (old code-review history)

```bash
git fetch origin refs/notes/commits:refs/notes/commits      # old SVN review metadata
git fetch gerrit refs/notes/review:refs/notes/review        # Gerrit vote metadata
```

Repeat per repo.

### 13.3. A patch that depends on another unmerged patch

Just build on top locally and push both — Gerrit detects the parent. If you need to *change* an existing patch's dependency, cleanly rebase onto the new parent (see §6.9).

### 13.4. Custom dashboards

Build URLs of the form `https://gerrit.wikimedia.org/r/dashboard/?title=MyDash&Section=query&...`, save to bookmarks. Or use `Settings → Preferences → Menu Entries` to add custom queries to your sidebar.

---

## 14. Quick-Reference Cheat Sheet

```bash
### ONE-TIME SETUP
ssh-keygen -t ed25519 -C "you@example.com"
# → paste ~/.ssh/id_ed25519.pub in Gerrit Settings → SSH Keys
ssh -p 29418 <user>@gerrit.wikimedia.org        # verify
pip install git-review                          # or apt/dnf
mkdir -p ~/.config/git-review
printf '[gerrit]\ndefaultremote=origin\n' > ~/.config/git-review/git-review.conf

### CLONE + SETUP HOOKS
git clone ssh://<user>@gerrit.wikimedia.org:29418/mediawiki/core.git
cd core
git review -s

### NEW CHANGE
git checkout master && git pull
git checkout -b fix-thing
# ...edit files...
git commit -a                                   # follow §5 commit message rules
git pull --rebase origin master
git review                                      # push → opens Change in Gerrit

### AMEND AFTER REVIEW FEEDBACK
# ...edit files...
git commit -a --amend                           # KEEP the Change-Id line
git review

### DOWNLOAD SOMEONE'S CHANGE
git review -d 12345                             # 12345 = change number from URL

### WIP MODE
git review --wip                                # reviewers not notified

### DIFFERENT BRANCH
git review REL1_40

### SET TOPIC
git review -t my-topic

### ABANDON
# (web UI: More → Abandon)

### HANDY SEARCH URLS
# Your queue:                https://gerrit.wikimedia.org/r/q/has:attention
# Your open changes:         https://gerrit.wikimedia.org/r/q/owner:self+is:open
# Reviews needing you:       https://gerrit.wikimedia.org/r/q/reviewer:self+is:open+-is:wip
```

---

## 15. Pre-Push Sanity Checklist

1. ✅ `ssh -p 29418 <user>@gerrit.wikimedia.org` prints the Gerrit welcome.
2. ✅ `git review -s` was run in this repo (commit-msg hook installed, `origin` remote set up).
3. ✅ Commit message follows §5: imperative subject ≤~70 chars, blank line, body, `Bug: Txxxx` if applicable, `Change-Id:` as the very last line.
4. ✅ Local tests pass.
5. ✅ Rebased onto latest upstream (`git pull --rebase origin master`).
6. ✅ One logical change per patch (squashed any local WIP commits).
7. ✅ About to push via `git review` (or `git push origin HEAD:refs/for/<branch>`).
8. ✅ Plan to add 1–2 reviewers immediately after and click "Add to attention set".
9. ✅ On subsequent revisions, will use `git commit --amend` (NEVER a new commit) and preserve `Change-Id`.

---

## 16. Where to Get Human Help

- **IRC (libera.chat):** `#mediawiki-core`, `#wikimedia-dev`, `#wikimedia-codereview`, `#wikimedia` (general)
- **Mailing list:** `wikitech-l` — https://lists.wikimedia.org/mailman/listinfo/wikitech-l
- **Phabricator:** tag `#gerrit` for Gerrit itself; `#gerrit-privilege-requests` for access; the project tag for the code.
- **Code Review Office Hours** — twice weekly on `#wikimedia-codereview`: https://www.mediawiki.org/wiki/Code_Review/Office_Hours
- **FAQ:** https://www.mediawiki.org/wiki/Git_and_Gerrit_FAQ

---

## 17. Out-of-Scope Reminder

This file intentionally does **not** cover:

- MediaWiki coding conventions (PHP / JS / CSS): https://www.mediawiki.org/wiki/Manual:Coding_conventions
- Local dev environment: https://www.mediawiki.org/wiki/MediaWiki-Docker
- How to pick a task: https://www.mediawiki.org/wiki/New_Developers and https://www.mediawiki.org/wiki/How_to_become_a_MediaWiki_hacker
- Pre-commit checklist: https://www.mediawiki.org/wiki/Manual:Pre-commit_checklist
- Security checklist: https://www.mediawiki.org/wiki/Security_checklist_for_developers
- Stable interface policy: https://www.mediawiki.org/wiki/Stable_interface_policy

Those belong in separate companion files. This document is complete for **Gerrit mechanics and policy only**.
