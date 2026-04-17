# T407468 — Implementation Brief for Claude Code

> **For the Claude Code agent reading this:** This brief is a complete handoff from a planning conversation. Read it in full before doing anything. Then read the linked code files in the repo. Then, before writing implementation code, present a design recap to the human overseer (Nityamittal) for approval. Iterate until approved. Only then write code. Do not skip the design-approval step.

---

## 0. Top-level orientation

This patch implements **T407468: Job for creating download files**.

**Scope overview:**

T402522 (already on Gerrit as change 1270914) introduced a `DownloadJob` class whose `run()` method is currently a stub — it immediately marks the request as `failed` with the message "Download generation not yet implemented (T407468)." This patch replaces that stub body with real audio generation.

The real work is already done in existing code. **You are not writing the audio-merging logic**. Sebastian Berlin wrote it in `includes/PageFileGenerator.php` (merged November 2025, see task T402518). Your job is to invoke that class from within `DownloadJob::run()` and translate its behavior (exceptions, produced file paths) into updates to the `DownloadRequest` state machine.

Net effect: when a client calls the `wikispeech-download-request` API and then runs the job queue, a real `.opus` audio file of the requested article is produced, stored on disk, and referenced from the `DownloadRequest.file_path` column. The client can then poll the status API and see `status: done` with a `file_url` they can use to retrieve the audio.

**Decisions already made (do not relitigate):**

1. **Thin wrapper approach.** The job delegates to `PageFileGenerator::makePageFile()`. We do not re-implement audio merging, we do not modify `PageFileGenerator`.
2. **Announcement integration deferred.** T403152's `DownloadAnnouncement` class is NOT wired into audio production in this patch. That belongs in a follow-up after Sebastian decides where it should live (inside `PageFileGenerator` or in a wrapper). Surface this as a question in the Phab comment.
3. **No producer-mode (consumer-url) support in this patch.** `makePageFile` accepts an optional `consumerUrl` argument; for the job, we pass `null`. If Sebastian asks for consumer-url support during review, add it as a patchset. For now, producer mode was explicitly deferred from T402522 too, so this patch maintains consistency.
4. **Service-wire `PageFileGenerator`.** Currently it's not in `ServiceWiring.php` — the existing maintenance script constructs it manually. We add it as `Wikispeech.PageFileGenerator` so the job can receive it via dependency injection (matching the codebase pattern).
5. **Status transitions:** On entry to `run()`, mark request as `in_progress`. On success (no exception), mark as `done` with `file_path` populated. On `RuntimeException` (the only exception `PageFileGenerator` throws), mark as `failed` with the exception message as `error_message`.

**Size estimate:** This will be a small patch. Roughly 300-400 lines of changed code plus tests. Significantly smaller than T402522.

**Out of scope:**

- Audio merging logic (already done in `PageFileGenerator`, not ours to reimplement).
- Announcement integration (deferred to follow-up).
- Producer mode / consumer-url support (deferred).
- Converting `file_path` to a public URL for the status API response (may be a follow-up; for now the status API returns the raw file path, which is the same behavior as the T402522 stub produced).

---

## 1. Files Claude Code must read before writing anything

Use the `view` tool on every one of these. Read fully, not skim.

### Style and process references
1. `docs/wikispeech-code-style.md` — authoritative style guide. Failure to match this guide causes review friction. **Read fully.**

### The reference implementation (most important)
2. `maintenance/generatePageFile.php` — existing maintenance script that calls `PageFileGenerator::makePageFile()`. **This is your template.** Your job's `run()` method will do approximately what this script's `execute()` method does. Read the constructor, the `execute()` method, and note how services are fetched via `WikispeechServices::getXxx()` static accessors.
3. `includes/PageFileGenerator.php` — the class you'll call. **Read all 192 lines.** Pay close attention to:
   - The constructor signature (6 parameters).
   - `makePageFile(string $page, string $language, ?string $consumerUrl = null)` — no return value, throws `RuntimeException` on every failure path.
   - The file path construction at the end: `"$dirPath/$title.opus"` where `$dirPath = $this->config->get('UploadDirectory') . '/page-audio'`.

### Your existing T402522 code (what you'll modify)
4. `includes/Download/DownloadJob.php` — your existing stub. This patch replaces the body of `run()`.
5. `includes/Download/DownloadRequest.php` — status constants, value object.
6. `includes/Download/DownloadRequestStore.php` — `updateStatus()` method signature.
7. `includes/Api/ApiWikispeechDownloadStatus.php` — status API response shape (you may add a `file_url` conversion here, or leave as-is for follow-up).
8. `tests/phpunit/Download/DownloadJobTest.php` — your existing 3 tests that assert the stub's behavior. These will need to be rewritten, because stub behavior is gone.

### Service wiring
9. `includes/ServiceWiring.php` — you'll add one new service entry for `Wikispeech.PageFileGenerator`.
10. `includes/WikispeechServices.php` — static accessor class used by the maintenance script pattern. Contains accessors like `getSegmentPageFactory()`, `getUtteranceGenerator()`, `getVoiceHandler()`. Read the full file; you may reference these accessors from service wiring.
11. `extension.json` — confirm whether `JobClasses.wikispeechDownload` already has the right `services` entry. We'll need to update it to include `Wikispeech.PageFileGenerator`.

### Test patterns to mirror
12. `tests/phpunit/integration/Download/DownloadRequestStoreTest.php` — your existing integration test pattern.
13. `tests/phpunit/ApiWikispeechListenTest.php` — for any mocking patterns you might need.

After reading: hold an internal model. Don't summarize unless asked. Present the design recap (§9 below).

---

## 2. The task — verbatim from Phabricator

**T407468: Job for creating download files**
https://phabricator.wikimedia.org/T407468

> This will merge utterance files into one big file for downloading. Each job should:
>
> 1. Check if an utterance exists. If not generate it.
> 2. Add the utterance to the end of "download file". Uses the service from ~~T402518: Combine utterances to audio files~~.
> 3. If the file is done, put it somewhere to be accessible.

**Key context:** T402518 is struck through because it was resolved. Its output was `PageFileGenerator` and the `generatePageFile.php` maintenance script. Your job is to invoke that existing class from the job queue.

---

## 3. Architecture

### 3.1 What changes

The T402522 patch introduced the `DownloadJob` class. That class currently has this body in `run()`:

```php
public function run() {
    $this->downloadRequestStore->updateStatus(
        $this->requestId,
        DownloadRequest::STATUS_FAILED,
        null,
        'Download generation not yet implemented (T407468).'
    );
    return true;
}
```

This patch replaces that body with actual work:

```php
public function run() {
    // 1. Mark as in-progress so the status API reflects that work started.
    $this->downloadRequestStore->updateStatus(
        $this->requestId,
        DownloadRequest::STATUS_IN_PROGRESS
    );

    // 2. Fetch the request to learn what to generate.
    $request = $this->downloadRequestStore->findById( $this->requestId );
    if ( !$request ) {
        // Request was deleted between enqueue and run. Nothing to do.
        $this->logger->warning(
            "DownloadJob: request {id} not found, skipping",
            [ 'id' => $this->requestId ]
        );
        return true;
    }

    // 3. Resolve title to a usable page name for PageFileGenerator.
    $title = $this->titleFactory->newFromID( $request->getPageId() );
    if ( !$title ) {
        $this->downloadRequestStore->updateStatus(
            $this->requestId,
            DownloadRequest::STATUS_FAILED,
            null,
            'Page no longer exists.'
        );
        return true;
    }

    // 4. Run the existing audio generation pipeline.
    try {
        $this->pageFileGenerator->makePageFile(
            $title->getPrefixedText(),
            $request->getLanguage()
        );
    } catch ( RuntimeException $e ) {
        $this->downloadRequestStore->updateStatus(
            $this->requestId,
            DownloadRequest::STATUS_FAILED,
            null,
            $e->getMessage()
        );
        return true;
    }

    // 5. Compute the produced file's path and mark as done.
    $filePath = $this->computeProducedFilePath( $title );
    $this->downloadRequestStore->updateStatus(
        $this->requestId,
        DownloadRequest::STATUS_DONE,
        $filePath
    );
    return true;
}
```

Key flow: `in_progress` → (call `PageFileGenerator`) → `done` on success, `failed` on exception.

### 3.2 The file-path computation problem

`PageFileGenerator::makePageFile()` does not return the produced file path — it produces the file as a side effect and returns void. We have to recompute the path ourselves based on its known construction: `"{UploadDirectory}/page-audio/{title}.opus"`.

This creates a tight coupling between our job and `PageFileGenerator`'s internal behavior. If Sebastian changes the output path later, our code breaks. **Flag this in the Phab comment** as a suggestion: `makePageFile` should return the produced file path.

For now, we replicate the path construction in a private helper:

```php
private function computeProducedFilePath( Title $title ): string {
    $uploadDir = $this->config->get( 'UploadDirectory' );
    return "$uploadDir/page-audio/{$title->getPrefixedText()}.opus";
}
```

Note: `$title->getPrefixedText()` produces "Main Page" (with spaces). `PageFileGenerator` uses the same construction via `"$dirPath/$title.opus"` — the `__toString()` on Title also produces prefixed text. So we match.

### 3.3 Constructor changes

The existing `DownloadJob` constructor takes `($title, $params, $downloadRequestStore)`. We need to add more services:

- `PageFileGenerator` — to invoke audio generation.
- `TitleFactory` — to resolve page_id → Title.
- `Config` — to read `UploadDirectory`.
- `LoggerInterface` — already present.

Updated signature:

```php
public function __construct(
    $title,
    $params,
    $downloadRequestStore,
    $pageFileGenerator,
    $titleFactory,
    $config
) {
    parent::__construct( 'wikispeechDownload', $title, $params );
    $this->logger = LoggerFactory::getInstance( 'Wikispeech' );
    $this->requestId = $params['request_id'];
    $this->downloadRequestStore = $downloadRequestStore;
    $this->pageFileGenerator = $pageFileGenerator;
    $this->titleFactory = $titleFactory;
    $this->config = $config;
}
```

Corresponding update in `extension.json`:

```json
"wikispeechDownload": {
    "class": "\\MediaWiki\\Wikispeech\\Download\\DownloadJob",
    "services": [
        "Wikispeech.DownloadRequestStore",
        "Wikispeech.PageFileGenerator",
        "TitleFactory",
        "MainConfig"
    ]
}
```

### 3.4 Service wiring for `PageFileGenerator`

Add to `includes/ServiceWiring.php`:

```php
'Wikispeech.PageFileGenerator' => static function (
    MediaWikiServices $services
): PageFileGenerator {
    return new PageFileGenerator(
        RequestContext::getMain(),
        WikispeechServices::getSegmentPageFactory(),
        WikispeechServices::getUtteranceGenerator(),
        WikispeechServices::getVoiceHandler(),
        $services->getTitleFactory(),
        $services->getMainConfig()
    );
},
```

This mirrors how the maintenance script constructs it. `RequestContext::getMain()` is used because jobs run in a CLI context without a real HTTP request; the main context is the best we have. If Sebastian wants a different context, we adjust.

---

## 4. Other changes

### 4.1 Update the existing stub docblock

The `DownloadJob` class currently has this docblock noting the stub status:

```
Currently a stub: marks the request as failed with a "not yet
implemented" message. T407468 will replace the body of run() with
actual audio generation — fetching segments, synthesising each,
prepending the DownloadAnnouncement, and storing the merged file.
```

Update it to reflect that the stub is gone:

```
Job that processes a queued download request by generating a
downloadable audio file for the requested page.

Delegates to PageFileGenerator for the actual segment synthesis,
merging, and encoding. Updates the DownloadRequest's status as it
moves through in_progress -> done or failed.

@since 0.1.15
```

(Note: no mention of T403152's announcement — that's deferred.)

### 4.2 Logging

Keep the existing `$this->logger = LoggerFactory::getInstance('Wikispeech')`. Add log entries at:
- Start of `run()` with request_id.
- On successful completion with file path and elapsed time.
- On failure with the exception message.

Matches `FlushUtterancesFromStoreByPageIdJob`'s logging pattern.

### 4.3 Status API `file_url` (optional, scope decision)

Currently the status API returns `file_url` as the raw `filePath` from `DownloadRequest`. That's `/var/www/html/w/images/page-audio/Main Page.opus` — a filesystem path, not a URL.

Two options:

**Option X:** Leave as-is. The raw path is stored in the DB. Future work can turn it into a URL. This patch just plumbs through what `PageFileGenerator` produces.

**Option Y:** Convert to a URL in the status API using `$wgUploadPath` (the URL prefix for the upload directory).

Pick Option X for this patch. The URL conversion adds scope. Flag it in the Phab comment: "The `file_url` field currently returns the raw server path. Turning this into a public URL probably belongs here or in a small follow-up — happy to address in review."

---

## 5. Tests

### 5.1 Update `DownloadJobTest`

Your existing test file `tests/phpunit/Download/DownloadJobTest.php` has 3 tests that assert the stub's behavior. All three need to be rewritten or replaced, because the stub is gone.

New test plan (5 tests):

1. `testRun_validRequest_marksAsDone` — construct a request, run the job with a mocked `PageFileGenerator` that succeeds (returns void, throws nothing). Assert the request's status is `done`, `file_path` is populated, `completed_at` is set.

2. `testRun_pageFileGeneratorThrows_marksAsFailed` — construct a request, mock `PageFileGenerator::makePageFile()` to throw `RuntimeException`. Assert the request's status is `failed`, `error_message` matches the exception message.

3. `testRun_marksAsInProgressBeforeCalling` — hard to test cleanly without introspection. If feasible, use a test double that captures the order of operations. If not, skip this test and trust the code.

4. `testRun_pageDeleted_marksAsFailed` — construct a request whose `page_id` doesn't resolve to a Title. Assert the request is marked as failed with "Page no longer exists."

5. `testRun_requestNotFound_returnsTrueWithoutUpdate` — construct a Job with a non-existent `request_id`. Assert `run()` returns true, no state changes.

For all tests: use `$this->createMock(PageFileGenerator::class)` to avoid needing opus-tools in the test environment. Tests should NOT require running Speechoid or opus-tools; they test the job's logic, not the underlying audio pipeline.

### 5.2 No new integration tests for `PageFileGenerator`

`PageFileGenerator` is Sebastian's code, already in the repo. If it had tests, they'd be his to write. The task description on T402518 says "I ended up not writing any tests for this" — he explicitly left it untested due to the difficulty of mocking shell commands. Our patch doesn't add tests for his class.

---

## 6. Manual end-to-end testing (per §11 of the T402522 pattern)

This is the payoff for the single-patch approach — end-to-end test of the full download feature.

### 6.1 Apply DB state

Your DB already has the `wikispeech_download_request` table from T402522. Confirm:

```bash
docker compose exec mediawiki php maintenance/run.php sql --query \
    "SELECT COUNT(*) FROM wikispeech_download_request;"
```

Should return 0 (cleaned up at the end of T402522 testing).

### 6.2 Create a request

From your host shell:

```bash
cd ~
rm -f cookies.txt
TOKEN=$(curl -s -c cookies.txt -b cookies.txt \
    "http://localhost:8080/w/api.php?action=query&meta=tokens&format=json" \
    | python3 -c "import sys, json; print(json.load(sys.stdin)['query']['tokens']['csrftoken'])")

curl -s -c cookies.txt -b cookies.txt \
    -X POST "http://localhost:8080/w/api.php" \
    -d "action=wikispeech-download-request" \
    -d "format=json" \
    -d "page=Main_Page" \
    -d "language=en" \
    -d "voice=cmu-slt-hsmm" \
    --data-urlencode "token=$TOKEN" | python3 -m json.tool
```

Expected: `{"wikispeech-download-request": {"request_id": N, "status": "pending", ...}}`.

### 6.3 Run the job

```bash
docker compose exec mediawiki php maintenance/run.php runJobs --type wikispeechDownload 2>&1
```

This takes ~10-60 seconds for Main_Page depending on Speechoid speed. Watch for it to complete.

### 6.4 Check status

```bash
curl -s "http://localhost:8080/w/api.php?action=wikispeech-download-status&request_id=N&format=json" \
    | python3 -m json.tool
```

Expected: `status: "done"`, `file_url` pointing to something like `/var/www/html/w/images/page-audio/Main Page.opus`, `completed_at` populated, `error` null.

### 6.5 Verify the file exists and plays

```bash
docker compose exec mediawiki ls -la /var/www/html/w/images/page-audio/
```

Should show the `.opus` file, ~100-400 KB for Main_Page.

**Copy it out of the container so you can play it:**

```bash
docker compose cp "mediawiki:/var/www/html/w/images/page-audio/Main Page.opus" /mnt/c/Users/Manoj/Downloads/MainPage.opus
```

Open in Windows Media Player, VLC, or any audio app. You should hear Main_Page being read aloud.

### 6.6 Test the dedup path with a completed request

Re-run the curl POST from §6.2. Expected: returns the same `request_id` with `existing: true` — because the previous request is in `done` status, which dedup treats as "active" (per our `findActiveByParameters` logic).

### 6.7 Test the failure path

Create a request for a nonexistent page (should fail at API level, before job runs — the T402522 validation catches this). Then create a request for a real page but simulate failure by temporarily uninstalling `opusdec`:

```bash
docker compose exec --user root mediawiki apt-get remove -y opus-tools
```

Create request, run job, check status. Expected: status `failed`, error something about "Program opusdec can't run" or similar.

Re-install:

```bash
docker compose exec --user root mediawiki apt-get install -y opus-tools
```

### 6.8 Cleanup

```bash
docker compose exec mediawiki php /var/www/html/w/extensions/Wikispeech/maintenance/cleanupDownloadRequests.php --days 0
```

Delete the produced audio file:

```bash
docker compose exec mediawiki rm "/var/www/html/w/images/page-audio/Main Page.opus"
```

---

## 7. Rebasing considerations

T402522 is on Gerrit as change 1270914 but not yet merged. This patch builds on top of T402522.

**Two rebase strategies:**

**Strategy A — Build on T402522's branch:**
```bash
cd ~/mediawiki/extensions/Wikispeech
git checkout T402522-download-request
git checkout -b T407468-download-job
```

This makes T407468's patch a chain on top of T402522. Gerrit will show them as a two-patch chain. When T402522 merges, T407468 rebases onto master automatically.

**Strategy B — Build on master + rebase later:**
```bash
git checkout master
git checkout -b T407468-download-job
```

This would create a conflict with T402522 (both modify `DownloadJob.php`, `extension.json`, etc.). You'd have to rebase T407468 after T402522 merges.

**Recommendation: Strategy A.** Chained patches are the normal Gerrit pattern for dependent work. The review UI shows them as a stack.

```bash
cd ~/mediawiki/extensions/Wikispeech
git checkout T402522-download-request
git pull origin T402522-download-request  # make sure you have latest
git checkout -b T407468-download-job
```

When pushing: `git review` will push to the same target branch (master) but Gerrit will show the dependency chain automatically.

---

## 8. Workflow — strict ordering

### Step A — Read

Use `view` on every file in §1. Don't skip any. Don't summarize unless asked.

### Step B — Recap design to Nityamittal

In chat, present the design recap covering:
- The 4-step transformation plan (§3.1).
- The file-path recomputation concern (§3.2).
- Service wiring approach (§3.4).
- Test plan (§5.1).
- Any deviations from the brief based on what you learned reading the code.

Wait for explicit "go" or adjustments.

### Step C — Implement, file by file

In this order. **Show the diff after each file and wait for review.**

1. `includes/ServiceWiring.php` — add `Wikispeech.PageFileGenerator`. Include `use RequestContext; use MediaWiki\Wikispeech\PageFileGenerator;` imports.
2. `extension.json` — update `JobClasses.wikispeechDownload.services` to include `Wikispeech.PageFileGenerator`, `TitleFactory`, `MainConfig`.
3. `includes/Download/DownloadJob.php` — update constructor signature, replace stub body in `run()`, add `computeProducedFilePath()` helper, update class docblock.
4. `tests/phpunit/Download/DownloadJobTest.php` — replace the 3 stub-behavior tests with the 5 new tests per §5.1.
5. `CHANGES.md` — entry: `* [T407468](https://phabricator.wikimedia.org/T407468) Generate article audio files via DownloadJob`.

After each: *"Here's the diff for X. OK to continue?"*

### Step D — Self-check

Walk through §14 of `wikispeech-code-style.md`. 12 questions. Flag anything that doesn't pass.

### Step E — Run tests

```bash
cd ~/mediawiki
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/Download/DownloadJobTest.php 2>&1 | tail -20
```

Show full output. Fix failures. Re-run.

### Step F — Run lint

```bash
cd ~/mediawiki
docker compose exec mediawiki bash -c "cd extensions/Wikispeech && composer test" 2>&1 | tail -30
```

**Important:** if `minus-x` complains about `docs/T407468-implementation-brief.md`, run `chmod -x docs/*.md` and rerun.

### Step G — Manual E2E testing

Walk through §6 step by step. Show me each command's output. Verify the `.opus` file is produced and (optionally) plays.

### Step H — Final review

Show me:
- `git status`
- `git diff --stat`
- Summary of what's in the patch.

Wait for explicit approval.

### Step I — Commit and push

Commit message in §9 below. Use `git commit -F /tmp/T407468-commit-msg.txt` with the message saved via VS Code `code` (to preserve blank lines).

### Step J — Phab comment

Nityamittal posts, not you. Content in §10.

---

## 9. Commit message

```
Implement DownloadJob to generate article audio files

Replaces the stub body of DownloadJob::run() with a call to the
existing PageFileGenerator, producing a downloadable .opus file
for the requested article and language.

The job transitions the DownloadRequest through
pending -> in_progress -> done|failed, matching the state machine
introduced in T402522. On successful generation, the produced
file's path is stored in wsd_file_path. On RuntimeException from
PageFileGenerator (invalid title, missing opus-tools, shell
disabled, synthesis failure), the job marks the request as failed
with the exception's message.

Service-wires PageFileGenerator as Wikispeech.PageFileGenerator
and injects it into DownloadJob along with TitleFactory and
MainConfig. Previously DownloadJob received only the store; the
maintenance script generatePageFile.php still constructs
PageFileGenerator manually and is unaffected.

Announcement integration (T403152's DownloadAnnouncement) is not
yet wired in — the right place for prepending an announcement to
the merged audio depends on design intent for PageFileGenerator
and is best resolved as a small follow-up.

Producer mode (consumer-url) support is not included, matching
T402522's deferral.

Bug: T407468
```

---

## 10. Phab comment Nityamittal posts after pushing

```
Initial patch up: https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/[NUMBER]

This replaces the T402522 stub in DownloadJob::run() with a call to PageFileGenerator::makePageFile(). The job now transitions pending → in_progress → done|failed, storing the produced file path on success.

A few notes for review:

1. Announcement integration (from T403152) is not wired into the produced audio. Two possible directions for a follow-up:
   (a) Extend PageFileGenerator::makePageFile() to accept an optional announcement and prepend it.
   (b) Keep PageFileGenerator focused; add a thin wrapper that handles prepending and calls PageFileGenerator for the main content.
   Happy to open a follow-up implementing whichever you prefer.

2. The job recomputes the produced file path by replicating PageFileGenerator's path construction ("{UploadDirectory}/page-audio/{title}.opus"). If makePageFile() returned the path it produced, we'd avoid this coupling. Small API change, worth considering.

3. The status API's `file_url` currently returns the raw server path, not a public URL. Turning it into a URL via $wgUploadPath feels like it belongs here, but adds scope — deferred unless you'd prefer it folded in.

4. Producer mode (consumer-url parameter) not supported, matching T402522's deferral.

Tested end-to-end locally through the full download lifecycle — API request through to produced .opus file that plays correctly.
```

---

## 11. Quality bar

Same as T402522: a senior PHP engineer reviewing this patch should not be able to tell whether it was AI-assisted. The patch reads like it belongs in the codebase: method names, variable names, comments, @since tags, test format.

One extra thing for this patch: **scope discipline.** The temptation is to do more than is asked — wire in the announcement, convert file paths to URLs, add consumer-url support. Resist all of these. Smaller patches merge faster. Scope-creeping a patch during implementation is a classic AI-assisted tell and a reviewer-relationship tell (it signals the contributor couldn't resist touching things outside the task).

---

## 12. Begin

Start with Step A. View every file in §1. Don't summarize unless asked. After reading, jump to Step B.
