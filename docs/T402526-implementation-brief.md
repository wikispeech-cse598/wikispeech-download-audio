# T402526 — Implementation Brief for Claude Code

> **For the Claude Code agent reading this:** This brief is a complete handoff from a planning conversation. Read it in full before doing anything. Then read the linked code files in the repo. Then, before writing implementation code, present a design recap to the human overseer (Pallak31) for approval. Iterate until approved. Only then write code. Do not skip the design-approval step.

---

## 0. Top-level orientation

This patch implements **T402526: UI for downloading an article**.

**Scope overview:**

Nitya's T402522 / T407468 / T402528 patches built the download-audio backend: API endpoints, job execution, Echo notifications. Those work but require `curl` and manual job runs — unusable for wiki readers.

This patch adds a **Special Page** named `Special:DownloadPageAudio` that fronts the backend with a human form. The flow is:

1. User navigates to the special page.
2. User enters an article title, picks a language, picks a voice (all in a single HTMLForm).
3. On submit the page shows:
   - The user's own existing download requests for that article, in a table.
   - A row marker/badge for the one existing row (if any) that is an "exact match" of the current form params (same page_id, language, voice, revision_id, where revision_id matches the current page revision).
   - A button to request a new download (which enqueues a job, refreshes the table).

This is the full spec from Sebastian's task description — not a minimal first cut.

**Decisions already made (do not relitigate):**

1. **Special Page, not a button on article pages.** Task description is explicit: "This should be a special page."
2. **HTMLForm.** Task explicitly mentions HTMLForm with title-type field.
3. **Single-form layout.** User submits article + language + voice once; the page then shows the list AND the ability to create a new one, on the same view. Not a two-step wizard.
4. **Exact-match badge only.** A row is "exact match" iff all four of page_id, language, voice, revision_id match (and revision_id matches the current page revision). Rows with older revisions or different params still appear in the table but without a match badge.
5. **Store method, not new API.** Add `DownloadRequestStore::findByPageIdAndUserId(int $pageId, ?int $userId): DownloadRequest[]` and call it from the Special Page's PHP. No new HTTP API surface in this patch.
6. **Show only the requesting user's own downloads.** Privacy-preserving. Anonymous users see an empty list.
7. **Anonymous users allowed to submit.** Mirrors T402522's `wikispeech-download` right which grants to all users. Anonymous users won't receive Echo notifications but can still create requests.
8. **No cross-wiki / consumer-URL support.** Deferred per T402522/T407468 pattern.
9. **Plain form submit, no AJAX.** Page reloads on submit. HTMLForm default.

**Size estimate:** 600-1000 lines plus tests. Medium-large patch — meaningfully larger than T407468 because of the table rendering and the new store method with test coverage.

**Out of scope:**

- Cross-wiki / consumer-wiki parameter handling.
- Listing other users' downloads.
- Real-time polling / AJAX refresh.
- Revision staleness warnings ("your download is for revision N, article is now M"). We show the revision ID and a "(current)" / "(outdated)" annotation but don't push the user to refresh.
- A separate "all my downloads across articles" page.
- Cancel-request functionality.

---

## 1. Files Claude Code must read before writing anything

Use the `view` tool on every one of these.

### Style and process references
1. `docs/wikispeech-code-style.md` — authoritative style guide. **Read fully.**
2. `docs/T402522-implementation-brief.md` — the API endpoint this page calls.
3. `docs/T407468-implementation-brief.md` — the job.
4. `docs/T402528-implementation-brief.md` — the notification.

### Existing code you'll interact with
5. `includes/Api/ApiWikispeechDownloadRequest.php` — the API module this page calls indirectly.
6. `includes/Api/ApiWikispeechDownloadStatus.php` — status API; fields surface in the table.
7. `includes/Download/DownloadRequest.php` — value object. You'll read every field.
8. `includes/Download/DownloadRequestStore.php` — you'll add `findByPageIdAndUserId()` here.
9. `includes/Download/DownloadJob.php` — see what gets fired on submit.
10. `includes/ServiceWiring.php` — how services are wired.
11. `extension.json` — the `SpecialPages` section. You'll add one.

### MediaWiki core patterns to mirror
12. `SpecialPage` base class — `/var/www/html/w/includes/specialpage/SpecialPage.php`. Understand `execute()`, `getOutput()`, `getDescription()`, `getGroupName()`.
13. Any existing Special Page using `HTMLForm` with `setSubmitCallback` as a good model. Look for one in `/var/www/html/w/includes/specials/`.
14. `HTMLForm` class: understand `newFromDescriptor`, field types (especially `title`), `setSubmitCallback`, `show()`.

### Wikispeech helpers
15. `includes/VoiceHandler.php` — understand how voices are listed per language. You'll use this to populate the voice dropdown.

### Test patterns
16. `tests/phpunit/Download/ApiWikispeechDownloadRequestTest.php` — similar test structure.
17. `tests/phpunit/integration/Download/DownloadRequestStoreTest.php` — you'll add tests here.

After reading: hold an internal model. Present the design recap (§6 below).

---

## 2. The task — verbatim from Phabricator

**T402526: UI for downloading an article**
https://phabricator.wikimedia.org/T402526

> This should be a special page where you specify the parameter for the audio (see T402522: Make a request to download an article). Article can use a field with type "title" if HTMLForm is used.
> If the selected article already is available for download you should get a link to it. Possibly multiple versions if there are, either because of different parameters or if there's been changes to the article. You should be able to tell if any of these downloads match what you want.
> If there's no version available, or you want to make a different one, there's a button to request download.

---

## 3. Architecture

### 3.1 The pieces

| # | File | What |
|---|---|---|
| 1 | `includes/Specials/SpecialDownloadPageAudio.php` | The Special Page class. Extends `SpecialPage`. Renders HTMLForm, handles submission, renders result table. |
| 2 | `includes/Download/DownloadRequestStore.php` (modified) | Add `findByPageIdAndUserId()` method. |
| 3 | `extension.json` (modified) | Register Special Page in `SpecialPages` section. |
| 4 | `i18n/en.json` + `qqq.json` | Message keys. |
| 5 | `tests/phpunit/Specials/SpecialDownloadPageAudioTest.php` | New test file. |
| 6 | `tests/phpunit/integration/Download/DownloadRequestStoreTest.php` (modified) | Tests for new store method. |
| 7 | `CHANGES.md` | Entry for T402526. |

### 3.2 The Special Page flow

```
User visits Special:DownloadPageAudio (GET, no params)
  → Show HTMLForm: [title field] [language select] [voice select] [Submit]

User submits (POST with form data)
  → Validate inputs (HTMLForm does most of this)
  → Resolve title → page_id, current_revision_id
  → Query store: findByPageIdAndUserId(page_id, current_user_id)
  → Determine which existing row (if any) is exact match
  → Render:
      - The form (with submitted values pre-filled)
      - The table of existing requests (with exact-match badge on one row if applicable)
      - A "Request new download" button

User clicks "Request new download"
  → POST to self with action=create flag
  → Dispatch to MediaWiki's internal API (ApiWikispeechDownloadRequest) with the form params
  → Redirect back to self (same page with GET params) so user sees updated table without stale POST state
```

### 3.3 The "exact match" rule

A row in the existing-downloads table is "exact match" iff:
- `request.page_id == submitted_page_id`
- `request.language == submitted_language`
- `request.voice == submitted_voice`
- `request.revision_id == current_revision_id_of_page`

Note: we compare the stored revision to the *current* revision, not to the originally-submitted one. If params match but the article has been edited since the request was made, it's not an exact match — because the user presumably wants audio of the current article, not of an outdated version.

All other rows display in the table without a badge. No partial-match indicators.

### 3.4 Base class: SpecialPage (not FormSpecialPage)

MediaWiki offers both:

- **`FormSpecialPage`**: built for "show form, submit, redirect to result" pages. The post-submit view is separate.
- **`SpecialPage`**: general. Full control over what renders.

Since we need form AND result table on the same page, use **`SpecialPage`** and build the HTMLForm inside `execute()` manually.

Skeleton:

```php
public function execute( $subPage ) {
    $this->setHeaders();
    $this->checkPermissions();

    // If user clicked "Request new download", handle that first (POST with action=create).
    $request = $this->getRequest();
    if ( $request->wasPosted() && $request->getVal( 'action' ) === 'create' ) {
        $this->handleCreate();
        return; // handleCreate() does redirect
    }

    // Always show the form.
    $form = $this->buildForm();

    // If form was submitted (either via GET-with-params from a redirect, or POST without action=create),
    // show the result table under the form.
    $submittedPage = $request->getVal( 'wpPage' );
    if ( $submittedPage ) {
        $this->renderResult( $submittedPage );
    }

    $form->show();
}
```

### 3.5 `findByPageIdAndUserId` store method

```php
/**
 * Find all requests for a page by a specific user.
 *
 * Returns requests of all statuses ordered by requested_at DESC.
 * Anonymous users get an empty list.
 *
 * @since 0.1.15
 * @param int $pageId
 * @param int|null $userId
 * @return DownloadRequest[]
 */
public function findByPageIdAndUserId( int $pageId, ?int $userId ): array {
    if ( $userId === null ) {
        return [];
    }
    $dbr = $this->connectionProvider->getReplicaDatabase();
    $result = $dbr->newSelectQueryBuilder()
        ->select( '*' )
        ->from( self::TABLE )
        ->where( [
            'wsd_page_id' => $pageId,
            'wsd_user_id' => $userId,
        ] )
        ->orderBy( 'wsd_requested_at', 'DESC' )
        ->caller( __METHOD__ )
        ->fetchResultSet();
    $out = [];
    foreach ( $result as $row ) {
        $out[] = $this->rowToRequest( $row );
    }
    return $out;
}
```

### 3.6 Table rendering

Seven columns:

| Language | Voice | Revision | Status | File | Requested | Match? |

Render with `Html::openElement` / `Html::closeElement` / `Html::element` for safety. Example skeleton:

```php
private function renderResultTable( array $requests, int $currentRevisionId, array $submittedParams ): string {
    if ( empty( $requests ) ) {
        return $this->msg( 'downloadpageaudio-no-requests' )->parse();
    }
    $html = Html::openElement( 'table', [ 'class' => 'wikitable' ] );
    // thead with column labels
    // tbody with one row per request
    //   match check inline: is this row an exact match of $submittedParams?
    $html .= Html::closeElement( 'table' );
    return $html;
}
```

The Revision column shows `42 (current)` or `40 (outdated)` depending on whether `request.revision_id == currentRevisionId`.

### 3.7 Submit → create flow

On the "Request new download" button click (which posts `action=create`), dispatch to `ApiWikispeechDownloadRequest` internally using MediaWiki's FauxRequest/ApiMain pattern:

```php
private function handleCreate(): void {
    $user = $this->getUser();
    $params = [
        'action' => 'wikispeech-download-request',
        'format' => 'json',
        'page' => $this->getRequest()->getVal( 'wpPage' ),
        'language' => $this->getRequest()->getVal( 'wpLanguage' ),
        'voice' => $this->getRequest()->getVal( 'wpVoice' ),
        'token' => $user->getEditToken(),
    ];
    $fauxRequest = new FauxRequest( $params, /* wasPosted */ true );
    $apiContext = new DerivativeContext( $this->getContext() );
    $apiContext->setRequest( $fauxRequest );
    $apiMain = new ApiMain( $apiContext, /* enableWrite */ true );
    $apiMain->execute();
    // Regardless of create vs dedup-hit, redirect to same page with GET params
    // so the table shows the current state without POST-replay issues.
    $url = $this->getPageTitle()->getFullURL( [
        'wpPage' => $params['page'],
        'wpLanguage' => $params['language'],
        'wpVoice' => $params['voice'],
    ] );
    $this->getOutput()->redirect( $url );
}
```

This avoids duplicating Nitya's validation/dedup/enqueue logic.

**Alternate consideration:** the agent may find that `FauxRequest`/`ApiMain` dispatch is awkward for write-action APIs. If so, an acceptable fallback is to extract Nitya's logic in `ApiWikispeechDownloadRequest::execute()` into a shared helper method (e.g., on `DownloadRequestStore` or a new `DownloadRequestCreator` service) and call that from both API and Special Page. This keeps one source of truth. Flag during design recap if you go this route.

### 3.8 Pre-fill form from GET params after redirect

After handleCreate() redirects with `?wpPage=...&wpLanguage=...&wpVoice=...`, the form's buildForm() reads these to pre-populate defaults. HTMLForm's field definitions support `default`:

```php
$request = $this->getRequest();
$formDescriptor = [
    'Page' => [
        'type' => 'title',
        'label-message' => 'downloadpageaudio-form-page-label',
        'required' => true,
        'default' => $request->getVal( 'wpPage', '' ),
    ],
    // ... language and voice similarly
];
```

### 3.9 i18n keys needed (~25)

- `specialpages-group-wikispeech`
- `downloadpageaudio`
- `downloadpageaudio-summary`
- `downloadpageaudio-form-page-label`
- `downloadpageaudio-form-language-label`
- `downloadpageaudio-form-voice-label`
- `downloadpageaudio-form-submit`
- `downloadpageaudio-no-requests`
- `downloadpageaudio-existing-requests-header`
- `downloadpageaudio-table-language`
- `downloadpageaudio-table-voice`
- `downloadpageaudio-table-revision`
- `downloadpageaudio-table-status`
- `downloadpageaudio-table-file`
- `downloadpageaudio-table-requested`
- `downloadpageaudio-table-match`
- `downloadpageaudio-match-exact`
- `downloadpageaudio-revision-current`
- `downloadpageaudio-revision-outdated`
- `downloadpageaudio-download-link`
- `downloadpageaudio-status-pending`
- `downloadpageaudio-status-in-progress`
- `downloadpageaudio-status-done`
- `downloadpageaudio-status-failed`
- `downloadpageaudio-request-new-button`
- `downloadpageaudio-error-invalid-page`
- `downloadpageaudio-error-invalid-language`
- `downloadpageaudio-error-invalid-voice`

All keys need entries in both en.json and qqq.json. qqq.json notes should be specific enough that a translator unfamiliar with Wikispeech can write correct translations.

---

## 4. Tests

### 4.1 Additions to `DownloadRequestStoreTest`

1. `testFindByPageIdAndUserId_userHasRequests_returnsAllForThatUser` — create 3 for (page=1,user=5), 1 for (page=1,user=6), 1 for (page=2,user=5). Query (1,5). Assert returns 3 in desc requested_at order.
2. `testFindByPageIdAndUserId_userHasNoRequests_returnsEmpty` — query a combo with no matches. Assert empty.
3. `testFindByPageIdAndUserId_anonymousUser_returnsEmpty` — pass null userId. Assert empty regardless of rows.

### 4.2 `SpecialDownloadPageAudioTest`

1. `testExecute_unsubmitted_showsForm` — GET with no params. Assert HTML contains form markup.
2. `testExecute_submittedValidParams_showsFormAndTable` — GET with wpPage/wpLanguage/wpVoice. Mock store. Assert HTML contains form (pre-filled) and table.
3. `testExecute_submittedWithExactMatch_showsMatchBadge` — mock the store to return a row that exactly matches params (including revision). Assert the "Exact match" badge appears on that row.
4. `testExecute_submittedInvalidPage_showsError` — submit with non-existent page. Assert error message.
5. `testExecute_actionCreate_dispatchesAndRedirects` — POST with action=create. Mock the API dispatch. Assert a redirect response with correct URL.

Use `SpecialPageTestBase` helpers from MW core if available.

### 4.3 No visual tests
Visual polish verified during manual E2E.

---

## 5. Manual end-to-end testing

### 5.1 Setup
- Wikispeech + Echo already set up.
- TestUser already exists (`TestUser` / `testpassword`, created during Nitya's T402528 work).
- Main_Page and Earth exist locally.

### 5.2 Visit the page

Navigate to `http://localhost:8080/wiki/Special:DownloadPageAudio` logged in as TestUser.

Expected: form with Page, Language, Voice fields, Submit button. No table yet.

### 5.3 Submit form for Main_Page

Page = `Main Page`, Language = `en`, Voice = `cmu-slt-hsmm`. Submit.

Expected: form pre-filled, table shows 1 row (the existing download from Nitya's T402528 testing, request_id=4), with "Exact match" badge (assuming Main_Page hasn't been edited since).

Click Download link in the row. Audio plays.

### 5.4 Submit form for Earth

Change Page to `Earth`. Submit.

Expected: 1 row (request_id=5 from earlier testing). Exact match.

### 5.5 Request new download (if a second voice is configured)

Check `$wgWikispeechVoices` for additional English voices. If one exists:
- Go back to Main Page. Change voice. Submit.
- Expected: table still shows the cmu-slt-hsmm row but NO exact match (voice differs). "Request new download" button visible.
- Click "Request new download".
- Page reloads. Table now has 2 rows. The new pending row is the exact match.
- Run `runJobs --type wikispeechDownload`. Refresh page. New row flips to done.

If no other voice is available locally, skip this step and describe it in the demo.

### 5.6 Anonymous case
Log out. Submit form. Table is empty (anonymous users see nothing). "Request new download" button still appears. Clicking it creates a request (attributable to anonymous). Nothing appears in the anonymous user's table — per design.

### 5.7 Invalid page
Submit a non-existent page title. Expected: error message. No table.

---

## 6. Workflow — strict ordering

### Step A — Read
`view` every file in §1.

### Step B — Recap design to Pallak31
- File layout (§3.1)
- Submit flow (§3.2)
- Base class choice (§3.4)
- Create path: FauxRequest dispatch (§3.7) or extract shared helper — pick one, explain why
- Test plan (§4)
- Deviations

Wait for approval.

### Step C — Implement, file by file
Order:
1. `includes/Download/DownloadRequestStore.php` — add method.
2. `tests/phpunit/integration/Download/DownloadRequestStoreTest.php` — add tests. Run. Green.
3. `includes/Specials/SpecialDownloadPageAudio.php` — biggest file.
4. `extension.json` — register the Special Page.
5. `i18n/en.json` + `i18n/qqq.json` — message keys.
6. `tests/phpunit/Specials/SpecialDownloadPageAudioTest.php` — test the page.
7. `CHANGES.md`.

After each: show diff, wait.

### Step D — Self-check
§14 of style guide. 12 questions.

### Step E — Tests
```bash
cd ~/mediawiki
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/Download/ 2>&1 | tail -10
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/integration/Download/ 2>&1 | tail -10
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/Specials/ 2>&1 | tail -10
```
All green.

### Step F — Lint
```bash
chmod -x ~/mediawiki/extensions/Wikispeech/docs/*.md
cd ~/mediawiki
docker compose exec mediawiki bash -c "cd extensions/Wikispeech && composer test" 2>&1 | tail -20
```
All three stages clean.

### Step G — Manual E2E
§5 walkthrough. Show outputs.

### Step H — Final review
`git status`, `git diff --stat`, summary. Wait for approval.

### Step I — Commit
printf → `/tmp/T402526-commit-msg.txt`. Verify with `cat -n`. `git commit -F`.

### Step J — Push
`git review`. This is Pallak's first Gerrit push from this account. Verify SSH identity is pallak31 first.

### Step K — Phab comment
Post on T402526 with the Gerrit URL. Content in §8.

---

## 7. Commit message

```
Add Special:DownloadPageAudio for requesting article audio

Introduces a user-facing Special Page that wraps the Wikispeech
download APIs. Users enter an article title, language, and voice
via an HTMLForm; after submission, the page shows the user's
existing download requests for that article in a table and offers
a button to create a new request.

An "exact match" badge marks the row whose page_id, language,
voice, and revision_id precisely match the submitted parameters
and the article's current revision. Other rows display without
a badge — they remain usable for download but aren't highlighted.

Only the requesting user's own downloads are listed, to avoid
leaking other users' request history. Anonymous users can create
requests but don't see a list (their requests are unattributed).

Adds DownloadRequestStore::findByPageIdAndUserId() as the backing
query. The Special Page dispatches to the existing
ApiWikispeechDownloadRequest module internally for the create
path, so dedup and job-enqueue logic aren't duplicated.

Bug: T402526
```

---

## 8. Phab comment to post after pushing

```
Initial patch up: https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/[NUMBER]

Implements Special:DownloadPageAudio as a single-form page: user enters title, language, voice, and gets back (a) a table of their existing requests for that article and (b) a "request new" button. Exact-match badge fires when page_id, language, voice, and revision_id all match the current article revision.

Notes for review:

1. List scope: shows only the requesting user's own downloads (privacy). Anonymous users see an empty list but can still create requests.

2. Match rule: only exact matches get a badge. Rows with older revisions or different parameters show in the table without special treatment — usable, not highlighted.

3. Create path: internally dispatches to ApiWikispeechDownloadRequest rather than duplicating the validation/dedup/enqueue logic. Keeps one source of truth.

4. Out of scope (follow-ups if needed): revision staleness warnings, cancel-request, listing across articles, AJAX refresh.

Tested end-to-end locally: visited the page, submitted the form, saw existing downloads, clicked download link, audio plays. Request-new-download flow creates a new row, job runs, row flips to done.
```

---

## 9. Quality bar

Same standard as Nitya's patches: Sebastian's style, no unnecessary abstractions, test naming `test<Method>_<scenario>_<outcome>`, `@since 0.1.15`, i18n consistency.

Special Pages in MediaWiki have strong conventions — don't reinvent headers, permissions, group assignment. The base class handles those.

~25 i18n keys is a lot. Use the `downloadpageaudio-` prefix consistently. qqq notes must be specific.

Before `git review`: **verify SSH identity is Pallak31, not Nitya**. Test:
```bash
ssh -p 29418 pallak31@gerrit.wikimedia.org
```
Should greet "Hi Pallak31". If it says "Nitya" or errors, SSH config is wrong — fix before pushing.

Also verify git identity:
```bash
git config --get user.name   # should be Pallak31
git config --get user.email  # should be pallak's gerrit-registered email
```

---

## 10. Begin

Step A. View every file in §1. After reading, Step B.
