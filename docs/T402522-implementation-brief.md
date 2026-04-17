# T402522 — Implementation Brief for Claude Code

> **For the Claude Code agent reading this:** This brief is a complete handoff from a planning conversation. Read it in full before doing anything. Then read the linked code files in the repo. Then, before writing implementation code, present a design recap to the human overseer (Nityamittal) for approval. Iterate until approved. Only then write code. Do not skip the design-approval step.

---

## 0. Top-level orientation

This patch implements **T402522: Make a request to download an article** as a single, comprehensive patch. The previous task on this codebase (T403152) was a small focused patch. This one is intentionally larger — it covers the full request-management lifecycle in one cohesive submission.

**Scope overview:**

This patch adds the entry point a user (or external tool) hits to say *"please generate a downloadable audio file for article X."* It is the front door of the download feature. It does **not** produce the audio file itself — that's the merge job's responsibility, tracked separately as T407468. This patch creates the request, validates it, deduplicates against existing requests, queues a job, exposes a way to check on the job's progress, and provides cleanup.

**Decisions already made (do not relitigate):**

1. **Public HTTP API**, not internal-only. Two new MediaWiki API modules following the pattern of `ApiWikispeechListen` and `ApiWikispeechSegment`.
2. **Sound logo handling deferred to T407468.** The config key `WikispeechDownloadAnnouncementSoundLogoFile` (added in T403152) is read by the merge job, not by this patch. This patch ignores it.
3. **Single patch submission**, not split into multiple chained patches. Nityamittal chose this path with full understanding of the tradeoffs.
4. **Includes a stub job class** that gets enqueued by the request API but does not yet produce the audio file. T407468 will replace the stub's body with real implementation. The stub exists so the request lifecycle can be tested end-to-end locally.
5. **The download request table is new.** No existing schema to extend.

**Out of scope:**

- Audio composition (T407468).
- The UI button that calls the API (T402526, Pallak31's task).
- Notification when the file is ready (T402528 — this is its own task; the API exposes status via polling, not push notifications).
- Producer-mode handling (`consumer-url` parameter for cross-wiki requests). Defer per Sebastian's comment on T402522.
- POC (Part of Content) parameters per Sebastian's comment ("Since the use of POC is still limited I'd say you can skip it for now").

---

## 1. Files Claude Code must read before writing anything

Use the `view` tool on every one of these. Read fully, not skim.

### Style and process references
1. `docs/wikispeech-code-style.md` — authoritative style guide. Failure to match this guide causes review friction. **Read fully.**
2. `docs/wikispeech-download-audio-tasks.md` §2 (T402522 section, lines roughly 152-300) — original umbrella brief. Read for context, but this implementation brief supersedes it where they conflict. The brief you're holding is the authority.

### Code patterns to mirror
3. `includes/Api/ApiWikispeechListen.php` — closest existing analog of a public HTTP API in this codebase. **Your `ApiWikispeechDownloadRequest` should structurally mirror this file.** Read the parameter declarations, the `execute()` shape, the error handling, the `getAllowedParams()`, `needsToken()`, and `getExamplesMessages()` methods especially closely.
4. `includes/Api/ApiWikispeechSegment.php` — second example of an API class, smaller. Cross-reference with `ApiWikispeechListen` to understand which patterns are conventions vs. one-offs.
5. `includes/Utterance/FlushUtterancesFromStoreByPageIdJob.php` — example of a MediaWiki Job class. Your new download job will mirror this structure: extends `Job`, implements `run()`, takes constructor params for the work to do.
6. `includes/Utterance/UtteranceStore.php` — example of a class that does database work via MediaWiki's database abstraction layer. Your `DownloadRequestStore` (or whatever you name it) should follow the same patterns for `IDatabase` access, query construction, and result mapping.
7. `includes/Download/DownloadAnnouncement.php` — Nityamittal's existing class in the same namespace your new code will join. Confirms namespace, file header, docblock conventions for this directory.
8. `includes/ServiceWiring.php` — service registration. You'll add new entries.
9. `includes/Hooks/DatabaseHooks.php` — handles `LoadExtensionSchemaUpdates`. You'll add a new schema entry here.
10. `extension.json` — manifest. You'll add: a new `AvailableRights` entry, a new `APIModules` entry (or two), a new `JobClasses` entry, and possibly new config keys.

### Schema files to mirror
11. `sql/tables.json` — the abstract schema definition. Read fully to understand the format. You'll add a new table here.
12. `sql/abstractSchemaChanges/patch-wikispeech_utterance-wsu_message_key.json` — example of an abstract schema change. The pattern is what you'll follow for your new table's schema files (though you're adding a new table, not patching an existing one).
13. `sql/sqlite/tables-generated.sql` — example of generated per-database SQL. You'll need a new SQL file for your new table per supported database (sqlite, mysql, postgres).

### Tests to mirror
14. `tests/phpunit/SpeechoidConnectorTest.php` — example test class structure (you used this for T403152).
15. `tests/phpunit/ApiWikispeechListenTest.php` — example of an integration test for an API class. **This is your primary test template.** Read fully to understand how API tests are structured in this codebase.
16. `tests/phpunit/integration/Utterance/UtteranceStoreTest.php` — example of a database-backed integration test. Your `DownloadRequestStore` tests will follow this pattern.

### i18n
17. `i18n/en.json` and `i18n/qqq.json` — main UI messages. You may add a few new keys for any user-visible strings (likely none for this patch since the API is machine-facing, but check).
18. `i18n/api/en.json` and `i18n/api/qqq.json` — API parameter docs and error messages. **You will add many keys here.** Skim the existing structure to learn the conventions.

After reading: hold an internal model of the codebase. Don't summarize to Nityamittal unless asked. Jump to Step B (design recap).

---

## 2. The task — verbatim from Phabricator

**T402522: Make a request to download an article**
https://phabricator.wikimedia.org/T402522

> This could be internal or an API if we want to allow external tools to use it. It should specify things that affect what is read and how, like:
> - Consumer wiki, if made on a producer wiki.
> - Article.
> - Voice.
> - Announcing POC? (This may differ from when you can interact with the page.)
>
> It'll take a while to generate a whole article so there should be some way to find it when it's done. Perhaps a URL where it will be available?
>
> If a request is made for an identical set of parameters and a file already exists or is currently being made, it should be rejected. A reference to the file should be given.

**Sebastian's comment on POC** (relevant): *"POC stands for 'part of content'. ... Since the use of POC is still limited I'd say you can skip it for now."*

**Parent task:** T397023 ☂ Download audio (umbrella).

**Sibling tasks referenced in this brief:**
- T403152 (Nityamittal's existing patch — DownloadAnnouncement)
- T407468 (the merge job — not yet implemented; this patch's stub job will be replaced when T407468 lands)
- T402526 (Pallak31's UI button — depends on this patch's API existing)

---

## 3. Architecture

### 3.1 The components

This patch introduces **6 new pieces of functionality**:

1. **Database schema** — a new table `wikispeech_download_request` storing one row per request.
2. **`DownloadRequest`** — a value object representing a single download request. Pure data, no behavior.
3. **`DownloadRequestStore`** — persists and queries `DownloadRequest` objects. Talks to the database.
4. **`ApiWikispeechDownloadRequest`** — public HTTP API to create a new request. Validates parameters, deduplicates, enqueues the job, returns the request reference.
5. **`ApiWikispeechDownloadStatus`** — public HTTP API to check the status of an existing request. Reads from store, returns current state.
6. **`DownloadJob`** — MediaWiki job class. **Stub implementation**: marks the request as `failed` with a "not yet implemented" reason. T407468 will replace this body with real audio generation.

Plus supporting changes: service wiring, schema migration registration, API module registration, job class registration, new user right, i18n.

### 3.2 The data flow

```
Client (UI button or external tool)
    │
    ▼ HTTP POST
ApiWikispeechDownloadRequest::execute()
    │ validates params (title, voice, language)
    │ checks user has 'wikispeech-download' right
    │ asks DownloadRequestStore: does identical request exist?
    │   yes → return existing request_id and status_url
    │   no  → create new DownloadRequest with status='pending'
    │         insert via DownloadRequestStore
    │         enqueue DownloadJob with the new request_id
    │         return new request_id and status_url
    │
    └──▶ JSON response: { request_id, status, status_url }


Client polls later:
    │
    ▼ HTTP GET
ApiWikispeechDownloadStatus::execute()
    │ reads request from DownloadRequestStore by request_id
    │ returns: status, file_url (if done), error (if failed)
    │
    └──▶ JSON response: { request_id, status, file_url, error }


Meanwhile, in the background:
    │
    ▼ MediaWiki job runner
DownloadJob::run()
    │ STUB — marks request as 'failed' with reason
    │       'Download generation not yet implemented (T407468).'
    │ updates the row via DownloadRequestStore
    └──▶ done
```

### 3.3 The status state machine

A request moves through these states:

```
                    pending
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
        in_progress           failed
            │
            ▼
          done
```

Valid transitions:
- `pending` → `in_progress` (job picks it up)
- `pending` → `failed` (validation error, stub failure)
- `in_progress` → `done` (job completed successfully)
- `in_progress` → `failed` (job hit an error)

For this patch's stub job, the path is `pending → failed` (the stub immediately fails). When T407468 lands, the path will be `pending → in_progress → done`.

### 3.4 Deduplication

Two requests are "identical" if they share the same:
- Page ID (or page title + revision if no page ID)
- Language
- Voice

When a duplicate is detected:
- If the existing request is in `pending`, `in_progress`, or `done` status → return that request's reference.
- If the existing request is in `failed` status → create a new request (don't return the failed one; the user is asking again presumably to retry).

**Implementation note:** put a unique index on the `(page_id, language, voice)` combination is tempting but **don't do this.** A failed request shouldn't block a retry. Handle deduplication in PHP, not at the DB level.

---

## 4. The schema

### 4.1 Table: `wikispeech_download_request`

Field naming uses the `wsd_` prefix (Wikispeech Download), following the existing convention (`wsu_` for Wikispeech Utterance).

| Column | Type | Notes |
|---|---|---|
| `wsd_request_id` | integer (autoincrement, unsigned, notnull) | Primary key. Returned to the client as `request_id`. |
| `wsd_page_id` | integer (unsigned, notnull) | The article being downloaded. From `page.page_id`. |
| `wsd_revision_id` | integer (unsigned, notnull) | Specific revision. From `revision.rev_id`. |
| `wsd_language` | binary, length 35 (notnull) | Language code, e.g. `en`, `sv`. Same type as `wsu_lang`. |
| `wsd_voice` | binary, length 100 (notnull) | Voice identifier, e.g. `cmu-slt-hsmm`. |
| `wsd_status` | binary, length 20 (notnull) | One of: `pending`, `in_progress`, `done`, `failed`. |
| `wsd_file_path` | string, length 255 (nullable) | File backend path to the produced audio file. NULL until status is `done`. |
| `wsd_error_message` | string, length 1000 (nullable) | Error description. NULL unless status is `failed`. |
| `wsd_requested_at` | mwtimestamp (notnull) | When the request was created. |
| `wsd_completed_at` | mwtimestamp (nullable) | When the request reached `done` or `failed`. NULL while pending/in_progress. |
| `wsd_user_id` | integer (unsigned, nullable) | User who made the request. NULL for anonymous. From `user.user_id`. |

**Indexes:**
- Primary key on `wsd_request_id`.
- Index on `(wsd_page_id, wsd_language, wsd_voice, wsd_status)` for deduplication queries. Name: `wsd_dedup`.
- Index on `wsd_status` for cleanup queries (find old completed requests). Name: `wsd_status`.
- Index on `wsd_requested_at` for time-based cleanup. Name: `wsd_requested_at`.

### 4.2 Schema files to create

1. `sql/tables.json` — add the new table definition to the existing JSON array.
2. `sql/sqlite/tables-generated.sql` — regenerated SQL for SQLite. Append the new table.
3. `sql/mysql/tables-generated.sql` — same for MySQL. Append.
4. `sql/postgres/tables-generated.sql` — same for Postgres. Append.

**How to regenerate:** MediaWiki provides a maintenance script. From the container:
```bash
docker compose exec mediawiki php maintenance/generateSchemaSql.php \
    --json extensions/Wikispeech/sql/tables.json \
    --sql extensions/Wikispeech/sql/{type}/tables-generated.sql \
    --type {type}
```
Run for each of `sqlite`, `mysql`, `postgres`. This regenerates the per-DB SQL from the abstract JSON definition.

If the `generateSchemaSql.php` script requires different invocation, follow whatever the existing `tables-generated.sql` files were generated with — there's a comment at the top of those files that usually mentions it.

### 4.3 Schema migration registration

In `includes/Hooks/DatabaseHooks.php`, the `LoadExtensionSchemaUpdates` hook handler runs on wiki install/update. Currently it adds the `wikispeech_utterance` table. Add a similar block for `wikispeech_download_request`:

```php
$updater->addExtensionTable(
    'wikispeech_download_request',
    "$dir/sql/$dbType/tables-generated.sql"
);
```

Where `$dbType` is determined from the database backend. Look at how the existing call is structured and mirror it exactly.

---

## 5. The PHP classes

### 5.1 `DownloadRequest` (value object)

Location: `includes/Download/DownloadRequest.php`
Namespace: `MediaWiki\Wikispeech\Download`

```php
namespace MediaWiki\Wikispeech\Download;

class DownloadRequest {

    public const STATUS_PENDING = 'pending';
    public const STATUS_IN_PROGRESS = 'in_progress';
    public const STATUS_DONE = 'done';
    public const STATUS_FAILED = 'failed';

    public function __construct(
        ?int $requestId,        // null for not-yet-persisted requests
        int $pageId,
        int $revisionId,
        string $language,
        string $voice,
        string $status,
        ?string $filePath,
        ?string $errorMessage,
        string $requestedAt,    // MW timestamp format
        ?string $completedAt,
        ?int $userId
    );

    public function getRequestId(): ?int;
    public function getPageId(): int;
    public function getRevisionId(): int;
    public function getLanguage(): string;
    public function getVoice(): string;
    public function getStatus(): string;
    public function getFilePath(): ?string;
    public function getErrorMessage(): ?string;
    public function getRequestedAt(): string;
    public function getCompletedAt(): ?string;
    public function getUserId(): ?int;
}
```

Pure value object. No business logic. Constructor takes all fields, getters return them. No setters — to update a request, create a new instance with the new values and pass it to the store (or use the store's update methods directly).

`@since 0.1.15` on the class and constructor.

### 5.2 `DownloadRequestStore` (database access)

Location: `includes/Download/DownloadRequestStore.php`
Namespace: `MediaWiki\Wikispeech\Download`

```php
namespace MediaWiki\Wikispeech\Download;

use Wikimedia\Rdbms\IConnectionProvider;

class DownloadRequestStore {

    public function __construct( IConnectionProvider $connectionProvider );

    /**
     * Create a new request. Returns the request with its assigned ID.
     */
    public function createRequest( DownloadRequest $request ): DownloadRequest;

    /**
     * Find a request by its ID. Returns null if not found.
     */
    public function findById( int $requestId ): ?DownloadRequest;

    /**
     * Find an existing non-failed request matching these parameters.
     * Returns null if no matching active request exists.
     */
    public function findActiveByParameters(
        int $pageId,
        string $language,
        string $voice
    ): ?DownloadRequest;

    /**
     * Update the status of a request. If status is 'done' or 'failed',
     * sets wsd_completed_at to the current timestamp.
     */
    public function updateStatus(
        int $requestId,
        string $status,
        ?string $filePath = null,
        ?string $errorMessage = null
    ): void;

    /**
     * Delete requests older than the given timestamp.
     * Used by cleanup maintenance scripts.
     */
    public function deleteOlderThan( string $timestamp ): int;
}
```

**Implementation notes:**

- Use `$this->connectionProvider->getPrimaryDatabase()` for writes, `getReplicaDatabase()` for reads. Pattern from existing MediaWiki code.
- Use `IDatabase::insert()` for createRequest. After insert, get the auto-incremented ID via `IDatabase::insertId()` and return a new `DownloadRequest` with the ID populated.
- Use `IDatabase::selectRow()` for `findById` and `findActiveByParameters`.
- Use `IDatabase::update()` for `updateStatus`.
- Use `IDatabase::delete()` for `deleteOlderThan`. Return the affected row count.

`@since 0.1.15` on the class and all public methods.

### 5.3 `ApiWikispeechDownloadRequest` (public HTTP API for creating requests)

Location: `includes/Api/ApiWikispeechDownloadRequest.php`
Namespace: `MediaWiki\Wikispeech\Api`

Extends `ApiBase`. Closely mirrors `ApiWikispeechListen.php` in structure.

**Module name (registered in extension.json):** `wikispeech-download-request`

**Parameters:**

| Param | Type | Required | Notes |
|---|---|---|---|
| `page` | string | yes | Article title. |
| `language` | string | yes | Language code. Must be in `$wgWikispeechVoices` keys. |
| `voice` | string | no | Voice identifier. If omitted, uses default for the language via `VoiceHandler::getDefaultVoice()`. |

**Permission check:** user must have the `wikispeech-download` right (added in this patch — see §6).

**Behavior of `execute()`:**

1. Get the page by title; resolve to a `Title` and a current revision.
2. If page doesn't exist → die with `apierror-wikispeech-download-request-page-not-found`.
3. If language is not in `$wgWikispeechVoices` → die with `apierror-wikispeech-download-request-invalid-language`.
4. If voice param missing, resolve via `VoiceHandler::getDefaultVoice($language)`. If still null → die with `apierror-wikispeech-download-request-no-default-voice`.
5. Call `$store->findActiveByParameters($pageId, $language, $voice)`.
6. If active request exists → return its `request_id` and `status_url`. Set `result['existing'] = true` so client knows it's a dupe.
7. Otherwise:
   a. Build a new `DownloadRequest` with status `pending`, `wsd_requested_at` = now, `wsd_user_id` = current user (or null if anon).
   b. Call `$store->createRequest($newRequest)` to persist; gets back the request with ID.
   c. Enqueue a `DownloadJob` with the new request_id (see §5.5 for job).
   d. Return the new `request_id` and `status_url`.

**Response shape:**

```json
{
  "wikispeech-download-request": {
    "request_id": 123,
    "status": "pending",
    "status_url": "/w/api.php?action=wikispeech-download-status&request_id=123",
    "existing": false
  }
}
```

**`getAllowedParams()`, `needsToken()`, `mustBePosted()`, `getExamplesMessages()`** — follow patterns from `ApiWikispeechListen`. This API mutates state, so `mustBePosted()` should return `true` and `needsToken()` should return `'csrf'`.

`@since 0.1.15` everywhere.

### 5.4 `ApiWikispeechDownloadStatus` (public HTTP API for checking status)

Location: `includes/Api/ApiWikispeechDownloadStatus.php`
Namespace: `MediaWiki\Wikispeech\Api`

Extends `ApiBase`. Smaller than the request API.

**Module name:** `wikispeech-download-status`

**Parameters:**

| Param | Type | Required | Notes |
|---|---|---|---|
| `request_id` | integer | yes | The request to check. |

**Permission check:** same `wikispeech-download` right.

**Behavior:**

1. Look up request by ID via `$store->findById()`.
2. If not found → die with `apierror-wikispeech-download-status-not-found`.
3. Return the request's current state.

**Response shape:**

```json
{
  "wikispeech-download-status": {
    "request_id": 123,
    "status": "in_progress",
    "file_url": null,
    "error": null,
    "requested_at": "2026-04-14T10:00:00Z",
    "completed_at": null
  }
}
```

When `status` is `done`, `file_url` is populated (a URL the client can GET to download the audio). When `status` is `failed`, `error` contains the error message.

**HTTP method:** `mustBePosted()` returns `false`. `needsToken()` returns `false`. This is a read-only endpoint.

`@since 0.1.15` everywhere.

### 5.5 `DownloadJob` (stub job class)

Location: `includes/Download/DownloadJob.php`
Namespace: `MediaWiki\Wikispeech\Download`

Extends MediaWiki's `Job` class. **Mirror the structure of `FlushUtterancesFromStoreByPageIdJob.php` exactly.** That's the canonical job pattern in this codebase, and review will reject deviations.

**Job name (registered in extension.json):** `wikispeechDownload`

**Constructor pattern — mirror exactly:**

```php
namespace MediaWiki\Wikispeech\Download;

use Job;
use MediaWiki\Logger\LoggerFactory;
use Psr\Log\LoggerInterface;

class DownloadJob extends Job {

    /** @var DownloadRequestStore */
    private $downloadRequestStore;

    /** @var LoggerInterface */
    private $logger;

    /** @var int */
    private $requestId;

    /**
     * @since 0.1.15
     * @param Title $title
     * @param array $params [ 'request_id' => int ]
     * @param DownloadRequestStore $downloadRequestStore
     */
    public function __construct( $title, $params, $downloadRequestStore ) {
        parent::__construct( 'wikispeechDownload', $title, $params );
        $this->logger = LoggerFactory::getInstance( 'Wikispeech' );
        $this->requestId = $params['request_id'];
        $this->downloadRequestStore = $downloadRequestStore;
    }

    public function run() {
        // STUB — see body below
    }
}
```

**Critical details — do not deviate:**

- **Untyped constructor parameters** (`$title, $params, $downloadRequestStore`), not typed. Matches the existing job pattern; Sebastian's codebase uses untyped constructor params on Job classes.
- **Service comes last** in the parameter list, after `$title` and `$params`. MediaWiki's job-injection mechanism passes services in the order declared in `extension.json`'s `"services"` array, after the standard Job constructor args.
- **Parent constructor:** `parent::__construct( 'wikispeechDownload', $title, $params )` — first arg is the job name registered in extension.json's JobClasses.
- **Logger and store assigned in constructor**, mirroring the existing pattern.
- **`run()` returns bool** but no return type declared (the parent `Job::run()` doesn't declare one in older MW versions; match the existing job).

**`run()` implementation (STUB):**

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

Note the difference from a typical "fetch service from MediaWikiServices" pattern: because the store is injected via the constructor, you use `$this->downloadRequestStore` directly. **Do not call `MediaWikiServices::getInstance()` inside `run()`** — that would defeat the point of service injection and not match the existing pattern.

**Important: the stub returns `true` (job succeeded) even though it marks the request as failed.** The job *as a job* did its work — it processed the request. The *download itself* failed because there's no implementation yet. Don't conflate the two.

When T407468 is implemented, this `run()` body will be replaced with: fetch segments, synthesize each, prepend `DownloadAnnouncement`, optionally prepend sound logo, merge into one file, store the file, mark as `done`. That's not your concern in this patch.

**Comment in the class docblock:** explicitly state that this is a stub awaiting T407468. Example:

```php
/**
 * Job that processes a queued download request.
 *
 * Currently a stub: marks the request as failed with a "not yet
 * implemented" message. T407468 will replace the body of run() with
 * actual audio generation — fetching segments, synthesising each,
 * prepending the DownloadAnnouncement, and storing the merged file.
 *
 * @since 0.1.15
 */
```

`@since 0.1.15` on the class and constructor.

### 5.6 Cleanup maintenance script

Location: `maintenance/cleanupDownloadRequests.php`

A simple maintenance script that calls `$store->deleteOlderThan()` with a configurable cutoff (default: 7 days).

**This is a nice-to-have, not load-bearing.** Include it because:
- Good hygiene — without cleanup, the table grows forever.
- Sebastian's existing `flushUtterances*` scripts establish the precedent of including cleanup with feature work.
- It's small (~50 lines).

Pattern: extend `Maintenance`, define options (`--days N`), call store method, log results.

---

## 6. extension.json changes

### 6.1 New API modules

Under `APIModules`:

```json
"wikispeech-download-request": {
    "class": "\\MediaWiki\\Wikispeech\\Api\\ApiWikispeechDownloadRequest",
    "services": [
        "MainConfig",
        "RevisionStore",
        "Wikispeech.DownloadRequestStore",
        "Wikispeech.VoiceHandler",
        "JobQueueGroup",
        "TitleFactory"
    ]
},
"wikispeech-download-status": {
    "class": "\\MediaWiki\\Wikispeech\\Api\\ApiWikispeechDownloadStatus",
    "services": [
        "Wikispeech.DownloadRequestStore"
    ]
}
```

Verify against `ApiWikispeechListen`'s entry to make sure the service names are correct.

### 6.2 New job class

Under `JobClasses`:

```json
"wikispeechDownload": {
    "class": "\\MediaWiki\\Wikispeech\\Download\\DownloadJob",
    "services": [
        "Wikispeech.DownloadRequestStore"
    ]
}
```

### 6.3 New user right

Under `AvailableRights`, add `wikispeech-download` to the array. Result:

```json
"AvailableRights": [
    "wikispeech-listen",
    "wikispeech-read-lexicon",
    "wikispeech-edit-lexicon",
    "wikispeech-edit-lexicon-raw",
    "wikispeech-download"
]
```

Under `GroupPermissions.*`, add the right for all users by default (matches the pattern for `wikispeech-listen`):

```json
"*": {
    "wikispeech-listen": true,
    "wikispeech-read-lexicon": true,
    "wikispeech-download": true
}
```

### 6.4 No new config keys in this patch

We're not adding new config — the sound logo config from T403152 is enough. If during implementation you discover something must be configurable (e.g. cleanup retention period), surface it for discussion before adding.

---

## 7. Service wiring (`includes/ServiceWiring.php`)

Add one new service:

```php
'Wikispeech.DownloadRequestStore' => static function (
    MediaWikiServices $services
): DownloadRequestStore {
    return new DownloadRequestStore(
        $services->getConnectionProvider()
    );
},
```

Sorted alphabetically per `@phpcs-require-sorted-array`. Goes between `Wikispeech.DownloadAnnouncement` (your T403152 entry) and `Wikispeech.LexiconHandler`.

Add the `use` statement at the top:
```php
use MediaWiki\Wikispeech\Download\DownloadRequestStore;
```

Also alphabetical, between `DownloadAnnouncement` and `Lexicon\ConfiguredLexiconStorage`.

---

## 8. i18n

### 8.1 New keys in `i18n/api/en.json` and `i18n/api/qqq.json`

These are API parameter descriptions and error messages. Convention: `apihelp-{module}-{thing}` and `apierror-{module}-{thing}`.

For `wikispeech-download-request` module:

```json
"apihelp-wikispeech-download-request-summary": "Request a downloadable audio file for an article.",
"apihelp-wikispeech-download-request-param-page": "Title of the article to download.",
"apihelp-wikispeech-download-request-param-language": "Language code for the audio.",
"apihelp-wikispeech-download-request-param-voice": "Voice identifier. If omitted, the default voice for the language is used.",
"apihelp-wikispeech-download-request-example-1": "Request a download for the article \"Earth\" in English.",

"apierror-wikispeech-download-request-page-not-found": "The requested page does not exist.",
"apierror-wikispeech-download-request-invalid-language": "Invalid language code.",
"apierror-wikispeech-download-request-no-default-voice": "No default voice is configured for this language."
```

For `wikispeech-download-status` module:

```json
"apihelp-wikispeech-download-status-summary": "Check the status of a download request.",
"apihelp-wikispeech-download-status-param-request_id": "The download request ID returned by wikispeech-download-request.",
"apihelp-wikispeech-download-status-example-1": "Check the status of download request 123.",

"apierror-wikispeech-download-status-not-found": "No download request found with the given ID."
```

Also examples need invocation strings — see how `ApiWikispeechListen` declares its examples in `getExamplesMessages()` and add the equivalent strings. Format matches:

```php
'action=wikispeech-download-request&format=json&page=Earth&language=en'
    => 'apihelp-wikispeech-download-request-example-1',
```

### 8.2 New right description in `i18n/en.json` and `i18n/qqq.json`

For the `wikispeech-download` right:

```json
"right-wikispeech-download": "Request downloads of articles as audio files",
"action-wikispeech-download": "request a download of an article as an audio file"
```

Add in alphabetical position, matching the pattern of `right-wikispeech-listen` and friends. qqq descriptions follow `{{doc-right|...}}` and `{{doc-action|...}}` conventions you've already established.

---

## 9. CHANGES.md entry

At the top of the `### 0.1.15-SNAPSHOT` section, in the same format as your T403152 entry:

```
* [T402522](https://phabricator.wikimedia.org/T402522) Add API for requesting article audio downloads
```

(No period at the end, matching the surrounding entries.)

---

## 10. Tests

This is the largest test-writing effort in this patch. Plan for ~15-25 test methods spread across 3-4 test classes.

### 10.1 `tests/phpunit/integration/Download/DownloadRequestStoreTest.php`

Extends `MediaWikiIntegrationTestCase`. `@group medium`, `@group Database` (this hits the DB). `@covers \MediaWiki\Wikispeech\Download\DownloadRequestStore`.

Tests (each named `test<Method>_<scenario>_<outcome>`):

1. `testCreateRequest_validInput_returnsRequestWithId` — basic create, verify ID populated.
2. `testFindById_existingRequest_returnsRequest` — round-trip create then read.
3. `testFindById_nonExistent_returnsNull`.
4. `testFindActiveByParameters_pendingRequestExists_returnsRequest`.
5. `testFindActiveByParameters_inProgressRequestExists_returnsRequest`.
6. `testFindActiveByParameters_doneRequestExists_returnsRequest`.
7. `testFindActiveByParameters_failedRequestExists_returnsNull` — failed requests don't block retries.
8. `testFindActiveByParameters_differentVoice_returnsNull`.
9. `testFindActiveByParameters_differentLanguage_returnsNull`.
10. `testFindActiveByParameters_noRequest_returnsNull`.
11. `testUpdateStatus_toDone_setsCompletedAt`.
12. `testUpdateStatus_toFailed_setsCompletedAtAndError`.
13. `testUpdateStatus_toInProgress_doesNotSetCompletedAt`.
14. `testDeleteOlderThan_removesOldRequests_returnsCount`.
15. `testDeleteOlderThan_keepsRecentRequests`.

### 10.2 `tests/phpunit/Download/ApiWikispeechDownloadRequestTest.php`

Extends `ApiTestCase` (MediaWiki provides this for testing API modules). `@group medium`, `@group Database`, `@group API`. `@covers \MediaWiki\Wikispeech\Api\ApiWikispeechDownloadRequest`.

Tests:

1. `testExecute_validRequest_returnsNewRequestId` — happy path.
2. `testExecute_existingActiveRequest_returnsExistingId` — dedup behavior.
3. `testExecute_pageNotFound_diesWithError`.
4. `testExecute_invalidLanguage_diesWithError`.
5. `testExecute_noVoiceProvidedAndNoDefault_diesWithError`.
6. `testExecute_noVoiceProvidedWithDefault_usesDefault`.
7. `testExecute_userLacksRight_throwsPermissionError`.
8. `testExecute_anonymousUser_succeedsAndStoresNullUserId`.
9. `testExecute_validRequest_enqueuesJob` — verify `JobQueueGroup` was called.

### 10.3 `tests/phpunit/Download/ApiWikispeechDownloadStatusTest.php`

Extends `ApiTestCase`. Same groups.

Tests:

1. `testExecute_existingPendingRequest_returnsPendingStatus`.
2. `testExecute_existingDoneRequest_returnsFileUrl`.
3. `testExecute_existingFailedRequest_returnsError`.
4. `testExecute_nonExistentRequest_diesWithError`.

### 10.4 `tests/phpunit/Download/DownloadJobTest.php`

Extends `MediaWikiIntegrationTestCase`. `@group medium`, `@group Database`, `@group JobQueue`.

Tests:

1. `testRun_validRequest_marksRequestAsFailedWithStubMessage` — verifies the stub behavior.
2. `testRun_returnsTrue` — the job *as a job* succeeds.
3. `testRun_nonExistentRequest_returnsFalse` — defensive: if the request was deleted between enqueue and run, fail gracefully.

### 10.5 Test patterns

- Use `MediaWikiIntegrationTestCase` not `MediaWikiUnitTestCase` for everything (these all touch the DB or API framework).
- For `ApiTestCase`, the standard pattern is `$this->doApiRequest(['action' => 'wikispeech-download-request', ...])`. Look at `ApiWikispeechListenTest.php` for examples.
- Use `$this->setMwGlobals()` to override config in tests.
- For permission tests, use the test-user system: `$this->getTestUser('wikispeech-download')->getUser()` for a user with the right; `$this->getTestUser()->getUser()` for one without.

---

## 11. Local testing plan (before pushing)

Once all code is written and unit tests pass, you'll do end-to-end manual testing on the local wiki. **This is what justifies the single-patch approach** — testing as a coherent whole.

### 11.1 Apply the schema migration

```bash
docker compose exec mediawiki php maintenance/run.php update --quick
```

Watch the output for `wikispeech_download_request` table creation. If it errors, the schema files are wrong.

Verify the table exists:
```bash
docker compose exec mediawiki php maintenance/run.php sql --query "SHOW CREATE TABLE wikispeech_download_request;"
```

(Or for SQLite: `.schema wikispeech_download_request`)

### 11.2 Test the request API with curl

From your host shell:

```bash
# Get a CSRF token first
TOKEN=$(curl -s -c cookies.txt -b cookies.txt \
    "http://localhost:8080/w/api.php?action=query&meta=tokens&format=json" \
    | python3 -c "import sys, json; print(json.load(sys.stdin)['query']['tokens']['csrftoken'])")

# Create a download request
curl -s -c cookies.txt -b cookies.txt \
    -X POST "http://localhost:8080/w/api.php" \
    -d "action=wikispeech-download-request" \
    -d "format=json" \
    -d "page=Main_Page" \
    -d "language=en" \
    -d "token=$TOKEN" | python3 -m json.tool
```

Expected: JSON with `request_id`, `status: "pending"`, `status_url`.

### 11.3 Test deduplication

Run the same request twice. Second response should have `existing: true` and the same `request_id`.

### 11.4 Run the job manually

```bash
docker compose exec mediawiki php maintenance/run.php runJobs --type wikispeechDownload
```

Watch the output. Should report 1 job processed.

### 11.5 Check the status

```bash
curl -s "http://localhost:8080/w/api.php?action=wikispeech-download-status&request_id=1&format=json" \
    | python3 -m json.tool
```

Expected: `status: "failed"`, `error: "Download generation not yet implemented (T407468)."`.

This confirms the full lifecycle works: create → enqueue → run job → status updated → readable via status API.

### 11.6 Verify in DB

```bash
docker compose exec mediawiki php maintenance/run.php sql \
    --query "SELECT * FROM wikispeech_download_request;"
```

Should show your test rows.

### 11.7 Test cleanup script

```bash
docker compose exec mediawiki php extensions/Wikispeech/maintenance/cleanupDownloadRequests.php --days 0
```

Should delete all requests (since `--days 0` means everything is "older than 0 days ago"). Re-query to verify table is empty.

---

## 12. Workflow — strict ordering

### Step A — Read

Use `view` on every file in §1. Don't skip any. Don't summarize unless asked.

### Step B — Recap design to Nityamittal

In chat, present the design recap covering:
- The 6 components and how they fit together (§3).
- The schema design (§4).
- Confirmation of the public-HTTP-API decision and stub-job approach (§3, §5.5).
- Any deviations from the brief based on what you learned reading the code.

Wait for explicit "go" or adjustments.

### Step C — Implement, file by file

In this order. **Show the diff after each file and wait for review.** No piling up of changes.

1. `sql/tables.json` — schema definition.
2. Generate `sql/sqlite/tables-generated.sql`, `sql/mysql/tables-generated.sql`, `sql/postgres/tables-generated.sql`. (Use `generateSchemaSql.php` per §4.2.)
3. `includes/Hooks/DatabaseHooks.php` — register schema migration.
4. `includes/Download/DownloadRequest.php` — value object.
5. `includes/Download/DownloadRequestStore.php` — persistence.
6. `includes/Download/DownloadJob.php` — stub job.
7. `includes/Api/ApiWikispeechDownloadRequest.php` — request API.
8. `includes/Api/ApiWikispeechDownloadStatus.php` — status API.
9. `includes/ServiceWiring.php` — register new service.
10. `extension.json` — register API modules, job class, user right.
11. `i18n/en.json` — right description.
12. `i18n/qqq.json` — right description.
13. `i18n/api/en.json` — API params and errors.
14. `i18n/api/qqq.json` — API descriptions.
15. `maintenance/cleanupDownloadRequests.php` — cleanup script.
16. `tests/phpunit/integration/Download/DownloadRequestStoreTest.php`.
17. `tests/phpunit/Download/ApiWikispeechDownloadRequestTest.php`.
18. `tests/phpunit/Download/ApiWikispeechDownloadStatusTest.php`.
19. `tests/phpunit/Download/DownloadJobTest.php`.
20. `CHANGES.md` — entry.

After each: *"Here's the diff for X. OK to continue?"*

### Step D — Self-check against the style guide

Walk through §14 of `wikispeech-code-style.md`. For each of the 12 questions, give a one-line answer about this patch. Show as numbered list. Flag any that don't pass.

### Step E — Run unit/integration tests

```bash
cd ~/mediawiki
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/Download/
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/integration/Download/
```

Show full output. Fix failures. Re-run.

### Step F — Run lint

```bash
cd ~/mediawiki
docker compose exec mediawiki bash -c "cd extensions/Wikispeech && composer test"
```

Fix every warning on your changed files. Ignore pre-existing warnings on unchanged files.

### Step G — Apply schema migration locally

```bash
docker compose exec mediawiki php maintenance/run.php update --quick
```

Verify the table exists. If anything fails here, the schema files are wrong.

### Step H — Manual end-to-end testing

Walk through §11 step by step. **This is the testing that justifies the one-patch approach.** Don't skip any step. Show Nityamittal each command's output.

### Step I — Final review before push

Show Nityamittal:
- `git status` (verify only intended files changed)
- `git diff --stat` (verify line counts are reasonable; expect 800-1200 lines added)
- A summary of what's in the patch

Wait for explicit approval.

### Step J — Commit and push

```bash
cd ~/mediawiki/extensions/Wikispeech
git checkout -b T402522-download-request
git add <list all modified and new files>
git commit  # paste commit message from §13
git review
```

### Step K — Phab comment text (Nityamittal posts, not you)

See §14.

---

## 13. Commit message

```
Add API for requesting article audio downloads

Implements the request lifecycle for downloading whole articles as
audio files. Provides two new API modules:

* wikispeech-download-request creates a new download request,
  validates parameters, deduplicates against active existing
  requests, and enqueues a background job to produce the audio.
* wikispeech-download-status returns the current state of a
  download request by ID.

Persistence is via a new wikispeech_download_request table.
Requests progress through pending -> in_progress -> done/failed.
Identical requests with status pending, in_progress, or done are
deduplicated; failed requests do not block retries.

The DownloadJob class is a stub: it marks the request as failed
with a "not yet implemented" reason. T407468 will replace the
stub's body with actual audio generation (segment synthesis, merge,
optional sound logo, prepended announcement from T403152).

Adds the wikispeech-download user right (granted to all users by
default, matching wikispeech-listen) and a maintenance script for
cleaning up old request records.

Producer mode (consumer-url parameter) and POC handling are out of
scope, deferred per Sebastian's earlier comment on the task.

Bug: T402522
```

---

## 14. Phab comment Nityamittal posts after pushing

Short, in the style of the T403152 comment:

```
Initial patch up: [Gerrit URL]

Design structure:

- Two API modules: `wikispeech-download-request` (create) and `wikispeech-download-status` (check). Both follow the public-HTTP-API pattern of `ApiWikispeechListen`/`ApiWikispeechSegment`.
- New `wikispeech_download_request` table tracks request lifecycle (pending → in_progress → done/failed). Deduplication is in PHP, not enforced at DB level, so failed requests don't block retries.
- `DownloadJob` is a stub that marks requests as failed with a "not yet implemented" reason. T407468 will replace the stub body. Submitting it as a stub now lets the request lifecycle be tested end-to-end locally.
- New `wikispeech-download` user right, granted by default like `wikispeech-listen`.
- Producer mode and POC parameters skipped per your earlier comment.

Tested locally via curl through the full lifecycle. Looking forward to feedback, happy to iterate.
```

---

## 15. Quality bar

Same as T403152: a senior PHP engineer reviewing this patch should not be able to tell whether it was AI-assisted. In particular:

- Method names match codebase verb conventions.
- Variable names match the codebase (`$language` not `$languageCode`, etc.).
- Comments are terse and useful, not narrating obvious code.
- Error handling matches Sebastian's pattern: throw on I/O errors, no defensive over-validation.
- Test names use `test<Method>_<scenario>_<outcome>()` format.
- `@since 0.1.15` everywhere.
- No interfaces/factories/patterns the surrounding code doesn't use.
- The patch reads like it belongs in this codebase.

If the §14 self-check fails on any item, fix before pushing.

---

## 16. Begin

Start with Step A. View every file in §1. Don't summarize unless asked. After reading, jump to Step B.
