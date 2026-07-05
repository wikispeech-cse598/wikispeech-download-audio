# Dev Environment Notes (Windows / PowerShell)

> Practical notes for the machine this project is actually developed on, so a
> new session doesn't rediscover the same friction. The canonical setup story
> (Docker + WSL) is in `docs/mediawiki-docker-windows-wsl-setup.md`; this file
> records how things really run day-to-day, including the native-PHP path that
> ended up being used instead of Docker for lint/test/phan.

Last updated: **2026-07-05**

---

## Machine layout

- **OS / shell:** Windows, PowerShell (not bash). No `grep`/`sed`/`awk` — use
  `Select-String` or `git grep`. `2>&1` on native commands makes PowerShell
  print stderr as red "NativeCommandError" lines; these are usually **not**
  errors (e.g. `git checkout` prints "Switched to branch" on stderr). Judge by
  content, not the red.
- **MediaWiki checkout:** `c:/Users/Manoj/Desktop/mediawiki`
- **Extension:** `c:/Users/Manoj/Desktop/mediawiki/extensions/Wikispeech`
- **PHP:** 8.5.4, on `PATH` as `php` (lives at `C:\tools\php85\php.exe`).
- **Composer:** 2.9.5, on `PATH` as `composer`.
- **Docker:** **not available** in the PowerShell session (`docker` is not a
  recognised command). All lint/test/phan is run with native PHP against the
  tools in `vendor/bin`. Don't reach for `docker compose exec` — it won't work.

## One-time setup that's already been done

- `composer install` has been run inside `extensions/Wikispeech`, so
  `vendor/bin/phan`, `vendor/bin/phpcs`, `vendor/bin/phpcbf` all exist there.
- **`core.autocrlf` set to `false`** at the repo level for the Wikispeech
  extension. This project is LF-only; Windows' default `autocrlf=true` was
  converting files to CRLF on every checkout, which made `phpcs` fail locally
  (`Generic.Files.LineEndings.InvalidEOLChar`) and forced a `phpcbf`-then-amend
  loop on every branch switch. With it `false`, checkouts preserve LF and the
  churn stops. If you see phantom "modified" files after a branch switch with an
  empty `git diff`, it's a stale stat cache from the autocrlf change — a plain
  `git checkout -- <file>` clears it.

## Commands that work

Run from the extension directory unless noted.

**phan** (note the polyfill flag — there is no `php-ast` extension installed):
```
php vendor/bin/phan -d . --allow-polyfill-parser --no-progress-bar
```
Exit 0 and no output = clean.

**phpcs** (pass the specific files you touched):
```
php vendor/bin/phpcs -sp <file> <file> ...
```

**phpcbf** (auto-fix, mainly CRLF/whitespace):
```
php vendor/bin/phpcbf <file> <file> ...
```

**phpunit** — run from the **mediawiki root**, not the extension dir:
```
cd c:/Users/Manoj/Desktop/mediawiki
php vendor/bin/phpunit extensions/Wikispeech/tests/phpunit/Download/
```
Integration tests that hit the DB need the table migrated first:
```
php maintenance/run.php update --quick
```
(The `wikispeech_download_request` table lives in the SQLite dev DB at
`cache//my_wiki.sqlite`. If you see `no such table: unittest_wikispeech_download_request`,
run the update.)

**Amend without an editor prompt** (PowerShell):
```
$env:GIT_EDITOR="true"; git commit --amend
```
`git rebase --continue --no-edit` is **not** a valid flag — set `GIT_EDITOR`
instead.

## Gerrit push (from the extension dir)

```
git push origin HEAD:refs/for/master
```
It may prompt for login the first time in a session; the prompt sometimes
doesn't render visibly in PowerShell — answering it (or pressing Enter) still
works. Success prints `remote: SUCCESS` and the change URL. `Verified+2 …
removed` on push is normal (the old CI vote is cleared for the new patchset).

## Local state to preserve (do NOT lose)

On the dev machine, the following exist outside the pushed commits:

- **A `git stash`** parked on `master` (`stash@{0}: WIP on master: …
  Localisation updates`). It holds an uncommitted `extension.json` change from
  before the rebases started. Recover it with `git stash pop` once back on a
  clean `master` — don't drop it.
- **Three untracked local-demo files** (the T402528 Echo / local-wiring work,
  see `docs/local-demo-wiring.md`):
  - `includes/Api/ApiWikispeechDownloadArticle.php`
  - `includes/Utterance/GenerateArticleAudioJob.php`
  - `tests/phpunit/ApiWikispeechDownloadArticleTest.php`
  These are intentionally **not** part of any Gerrit change and are correctly
  left unstaged. (One phan error found this session —
  `PhanParamTooMany` on `makePageFile()` in `GenerateArticleAudioJob.php` — was
  fixed locally in these files; it does not affect any pushed patch.)

## Local branches in the Wikispeech checkout

- `T402522-rebase` → 1270914 PS2 (parent)
- `T407468-ps2` → 1271020 PS2 (child)
- `T403152-ps2` → 1270609 PS2 (announcement, independent) — may still exist
- (to be created) `T402526-ps2` → 1271979 rebase, still pending
