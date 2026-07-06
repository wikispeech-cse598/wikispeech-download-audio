# Patch Status & Session Log

> **Single source of truth for where every patch stands.** If you are a new
> session picking this project up cold, read this file first, then
> `docs/dev-environment-notes.md`, then the per-task implementation briefs.
> Update the "Snapshot" table and "Work done" log whenever state changes.

Last updated: **2026-07-06**

---

## Snapshot

The feature is five patches under umbrella task
[T397023](https://phabricator.wikimedia.org/T397023). Gerrit is the canonical
review system (see `docs/wikimedia-gerrit-guide.md`); the GitHub repo this file
lives in is only companion documentation + demo.

| Task | What | Gerrit change | Latest PS | CI (Verified) | Review state | Next action |
|---|---|---|---|---|---|---|
| [T403152](https://phabricator.wikimedia.org/T403152) | Download announcement | [1270609](https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/1270609) | **PS3** (rebased locally on `T403152-ps3`; push if not yet done) | was +2 on PS1; needs trusted `recheck` | Viktoria's 4 comments addressed in PS2; **check reply drafts were SENT** | Push PS3, send drafts, get CI kicked |
| [T402522](https://phabricator.wikimedia.org/T402522) | Download request API | [1270914](https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/1270914) | **PS2** | pending PS2 run | No comments yet | Wait for CI, then review |
| [T407468](https://phabricator.wikimedia.org/T407468) | Download job | [1271020](https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/1271020) | **PS2** | was -1 on PS1 (phan); fixed in PS2 | No comments yet | Wait for CI, then review |
| [T402526](https://phabricator.wikimedia.org/T402526) | Special page UI | [1271979](https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/1271979) | PS1 | — | **Needs rebase** onto new parent chain | Rebase onto 1271020 PS2 (see "Pending") |
| [T402528](https://phabricator.wikimedia.org/T402528) | Echo notification | *(not on Gerrit)* | local only | — | Committed locally, pending Sebastian's confirmation of notification design | Hold |

**Relation chain on Gerrit** (parent → child, each merges bottom-up):

```
1270914  Add API for requesting article audio downloads   (T402522)  ← parent
   └─ 1271020  Implement DownloadJob                       (T407468)
        └─ 1271979  Add Special:DownloadPageAudio          (T402526)  ← top, needs rebase
```

The announcement patch **1270609 (T403152) is independent** — not part of the
chain. It stands alone on master.

---

## Per-patch detail

### T403152 — Download announcement (1270609) — PS2 pushed

Class `DownloadAnnouncement` in `includes/Download/`. Synthesises the spoken
license/attribution + TTS-notice intro. Design is in
`docs/T403152-implementation-brief.md`.

**PS1 review** (Viktoria Hillerud, WMSE, no negative vote — "well implemented
and well documented"). Four inline comments, all addressed in PS2. Full
walkthrough in `docs/T403152-patchset2-review-response.md`. Summary of what
changed PS1 → PS2:

1. Exception message now includes the language:
   `"Invalid default voice configuration for language \"$language\"."`
   (was the bare copy of `ApiWikispeechListen`'s phrasing). Test's
   `expectExceptionMessage` updated to match.
2. The response-passthrough test mocks a realistic Speechoid response
   including a `tokens` array (`orth`/`endtime`, copied from
   `SpeechoidConnectorTest` fixtures), matching the documented return shape.
3. The null-voice test now asserts the resolved default voice actually reaches
   `synthesizeText()` via `->with( 'en', 'en-US', $this->anything() )`.
4. The license tests no longer grep for the English phrase `'released under'`
   (brittle against i18n rewording). The positive test sets `RightsText` to a
   sentinel (`'Sentinel License 9.9'`) and asserts it appears in the
   synthesised text; the empty-rights test asserts the text is non-empty and
   contains no unreplaced `$3` placeholder.

Gerrit replies posted on all four threads (marked Resolved) + a thank-you on
the overall comment.

### T402522 — Download request API (1270914) — PS2 pushed

Two API modules (`wikispeech-download-request`, `wikispeech-download-status`),
the `wikispeech_download_request` table, `DownloadRequest` +
`DownloadRequestStore`, a stub `DownloadJob`, the `wikispeech-download` user
right, and a cleanup maintenance script.

**PS1 → PS2**: rebased onto current master (was three months stale). Plus three
phan fixes that live in *this* change's files (so they belong here, not in the
child):
- `ApiWikispeechDownloadRequest.php`: non-capturing `catch
  ( SpeechoidConnectorException )` (was `catch ( … $e )` with `$e` unused →
  `PhanUnusedVariableCaughtException`).
- `DownloadRequestStore.php`: extract `$completedAt = $request->getCompletedAt()`
  into a local before the insert, so the nullable narrows correctly for
  `$dbw->timestamp()` (`PhanTypeMismatchArgumentNullable`). Written as a plain
  two-line form, not an inline-assignment ternary (reads like the codebase).
- Stub `DownloadJob.php`: added `use MediaWiki\Title\Title;` — the stub's
  `@param Title` docblock had no import, which phan flagged as
  `PhanUndeclaredTypeParameter` when the parent is analysed standing alone.
  **This was a latent failure in PS1 that would have -1'd this change on its
  own once CI ran against it.**

### T407468 — Download job (1271020) — PS2 pushed

Replaces the stub `DownloadJob::run()` with a real call to `PageFileGenerator`;
transitions the request pending → in_progress → done|failed; service-wires
`PageFileGenerator`; injects `TitleFactory` + `MainConfig`.

**PS1 → PS2**: rebased onto 1270914's PS2 (resolving the merge conflict that
was blocking CI entirely — Zuul couldn't even build). The DownloadJob commit
now contains **exactly its own five files**: `CHANGES.md`, `extension.json`,
`includes/Download/DownloadJob.php`, `includes/ServiceWiring.php`,
`tests/phpunit/Download/DownloadJobTest.php`. The phan fixes that had been
mis-amended into this commit during rebasing were moved back to the parent
(1270914), where the affected files are actually created. Announcement
integration (wiring T403152 in) remains deliberately out of scope — noted as a
follow-up in the commit message.

### T402526 — Special page UI (1271979) — needs rebase (PENDING)

Sits on top of 1271020 in the relation chain. Because 1270914 and 1271020 were
both amended and force-pushed as PS2, this change is now based on stale parent
commits and will show as outdated / possibly conflicting. **Not yet rebased.**
See "Pending work" below. Author: Pallak (@pallakk).

### T402528 — Echo notification (local only)

Not pushed to Gerrit. Committed locally pending Sebastian's confirmation of the
notification mechanism. No action this session. The local demo wiring for it
lives in the untracked files noted in `docs/dev-environment-notes.md`.

---

## Work done this session (chronological)

1. **T403152 PS2** — made the four review fixes above, ran phpunit (7/7) + phpcs
   (clean after `phpcbf` fixed CRLF), amended keeping Change-Id
   `I888de74a…`, pushed to `refs/for/master` → landed as PS2 of 1270609.
   Posted Gerrit replies.
2. **Rebased the chain** — 1270914 then 1271020 onto current master (`883e1e2`).
   Conflicts (all resolved keep-both): `CHANGES.md` in both (kept upstream
   0.1.15 section + our SNAPSHOT entries), `DownloadJob.php` `use` imports.
3. **Reproduced the phan -1 locally** (Jenkins log had expired) —
   `php vendor/bin/phan -d . --allow-polyfill-parser`. Found and fixed the four
   phan errors (see per-patch detail; two in parent-owned files, one in an
   untracked local file, one the missing `use Title` in the stub).
4. **Corrected commit ownership** — moved the two parent-owned phan fixes out of
   the child commit and into 1270914, so the parent is phan-clean standing
   alone. Re-seated the child with `git rebase --onto`.
5. **Killed the CRLF churn** — set `core.autocrlf false` at repo level (this is
   an LF-only project; Windows autocrlf was re-introducing CRLF on every
   checkout and making phpcs fail locally). Ran `phpcbf` to normalise, folded
   line-ending fixes into the correct commits.
6. **Verified final state** — both branches: phan exit 0, phpcs clean, phpunit
   `OK (20 tests, 44 assertions)`. Parent verified phan-clean *standing alone*.
   Both Change-Ids intact (`I453c8f0a…`, `Ibd341d2c…`).
7. **Pushed** both PS2 (parent first, then child) and added a top-level patchset
   note on each explaining what changed.

---

## Session addendum (2026-07-06)

- **T403152 rebased to PS3** on local branch `T403152-ps3`: PS2 had gone stale
  against master (Merge Conflict banner; Zuul can't build an unmergeable
  change, so even `recheck` was a no-op). Rebase was conflict-light: only
  `CHANGES.md` needed manual keep-both resolution (entry moved into the
  0.1.16-SNAPSHOT section above the released 0.1.15 block); the other shared
  files auto-merged. No code changes; phan/phpcs/phpunit (7/7) all clean
  first run. Push as PS3 with note "rebased onto current master, no code
  changes".
- **Draft-comment trap discovered**: the five replies to Viktoria's review
  threads were sitting as unsent Gerrit DRAFTS ("5 drafts" in the Comments
  header) — invisible to reviewers. Fix: Reply button → Send. Check this is
  done; drafts do NOT publish when a patchset is pushed.
- **CI trust discovered**: Zuul does not auto-run CI for non-allowlisted
  contributors, and a `recheck` from the change owner is ignored — it must
  come from a trusted user (this is why Pppery's `recheck`s worked and ours
  don't). Ask Sebastian to kick rechecks on 1270609/1270914/1271020 and to
  help get added to the allowlist (`integration/config`).

## Pending work / next actions

1. **Rebase T402526 (1271979)** onto the new chain. From the Wikispeech
   checkout, roughly:
   ```
   git fetch origin refs/changes/79/1271979/1
   git checkout -b T402526-ps2 FETCH_HEAD
   git rebase --onto T407468-ps2 <old-parent-sha>   # old-parent = 1271979's PS1 parent
   ```
   Then the same diagnostics (phan `--allow-polyfill-parser`, phpcs on touched
   files, phpunit), amend keeping its Change-Id, push. Resolve any keep-both
   conflicts the same way.
2. **Watch CI** on 1270914 and 1271020 PS2. With the conflict gone Zuul can
   build; phan was reproduced against the real CI version, so expect
   Verified +2 on both. If either fails, grab the console link from the fresh
   jenkins-bot comment *before it expires* (old logs 404 after some months).
3. **Add reviewers to attention set** on any change missing Sebastian/Viktoria.
4. **Announcement → job integration** (wiring T403152's `DownloadAnnouncement`
   into the merge step) is still an unfiled follow-up — file the Phab task once
   1271020 and 1270609 are both reviewed. Already flagged in README follow-ups.
5. **T402528 (Echo)** stays on hold until Sebastian confirms the notification
   design.

---

## Gotchas for a new session

- **`recheck`** as a Gerrit comment re-runs CI on the current patchset; it does
  **not** re-merge. Deterministic failures (phan) will reproduce until code
  changes — don't burn rechecks on them.
- **Push order matters** for the chain: parent (1270914) before child (1271020)
  before grandchild (1271979), so Gerrit links relations correctly.
- **Never touch Change-Id lines** when amending. Current IDs: 1270609
  `I888de74a334b0b97104611c5fcf4d4d2b294ecfe`, 1270914
  `I453c8f0a31fd2627c8057bb2a404796089766385`, 1271020
  `Ibd341d2c444156b0e8cf5bb5662c529dd0cf2a15`.
- **phan needs `--allow-polyfill-parser`** on the dev machine (no `php-ast`
  extension installed). See `docs/dev-environment-notes.md`.
- The dev machine has **local uncommitted state**: a `git stash` parked on
  master and three untracked local-demo files. Don't lose or commit them —
  details in the dev-environment notes.
