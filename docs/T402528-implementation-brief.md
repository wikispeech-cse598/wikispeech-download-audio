# T402528 — Implementation Brief for Claude Code

> **For the Claude Code agent reading this:** This brief is a complete handoff from a planning conversation. Read it in full before doing anything. Then read the linked code files in the repo. Then, before writing implementation code, present a design recap to the human overseer (Nityamittal) for approval. Iterate until approved. Only then write code. Do not skip the design-approval step.

---

## 0. Top-level orientation

This patch implements **T402528: Subscribe to an article download** by integrating with the **Echo (Notifications) extension**.

**Scope overview:**

After T407468's `DownloadJob` produces a downloadable `.opus` file and marks the request as `done`, the user who originally requested the download should be notified. This patch wires up that notification using Echo, the standard MediaWiki extension for the bell-icon notification system.

The notification appears in the user's Echo notification panel ("bell icon") with a body like *"Your audio file for Neutral Milk Hotel is ready to download. [Download]"* and a link to the produced file. If the user has email-on-Echo enabled in their preferences, they'll also get an email automatically — Echo handles that for free.

**Decisions already made (do not relitigate):**

1. **Echo, not the other options.** The umbrella task doc (`docs/wikispeech-download-audio-tasks.md`, §3.3) ranked the four notification mechanisms. Echo wins for single-wiki use; talk-page/email/RSS are deferred. Sebastian has been asked to confirm via Phab comment but his response is pending. **We are building on the assumption Echo is correct;** if Sebastian disagrees, this branch becomes a reference, not a merge candidate.
2. **Soft dependency on Echo.** `extension.json` does NOT add Echo to `requires`. Instead, all Echo-touching code checks `ExtensionRegistry::getInstance()->isLoaded('Echo')` and no-ops if absent. This means Wikispeech still works on wikis without Echo — the user just doesn't get a notification. Acceptance criterion §3.7 of the umbrella doc.
3. **Notification fires from `DownloadJob::run()`** (Option P from planning discussion), not from `DownloadRequestStore::updateStatus()`. The store is just persistence; it should not have notification side effects. The job is the orchestrator and the natural place to fire on success.
4. **Anonymous users get no notification.** Echo notifications require a logged-in user. The `DownloadRequest` already stores `user_id` (or null for anonymous); we skip the notification path when `user_id` is null.
5. **No notification fires on `failed` status.** This patch is "ready to download" notifications only. A "your download failed" notification is a different design question and a follow-up if anyone asks.
6. **Helper class wraps Echo internals.** Per umbrella doc §3.6.6, we add a `NotifyDownloadReady` helper that takes the user, downloadId, fileUrl, and pageTitle. The job calls this helper, not Echo directly. Two benefits: (a) callers don't need to know Echo internals, (b) the no-op-when-absent check lives in one place.

**Size estimate:** ~250-400 lines of new code plus tests. Single patch.

**Out of scope:**

- Notifications for other status transitions (failed, in-progress).
- Cross-wiki / consumer-wiki notification routing (deferred — gadget users poll the status API).
- Email-only fallback (Echo handles email opt-in automatically).
- Talk-page or RSS notification mechanisms.
- Bundling multiple completed downloads into one notification (Echo supports this, but adds complexity without clear value yet).

---

## 1. Files Claude Code must read before writing anything

Use the `view` tool on every one of these. Read fully, not skim.

### Style and process references
1. `docs/wikispeech-code-style.md` — authoritative style guide. **Read fully.**
2. `docs/wikispeech-download-audio-tasks.md` lines 280-360 — the umbrella task spec for T402528, the design Sebastian's team agreed in principle.

### The Echo extension itself (most important)
3. `~/mediawiki/extensions/Echo/includes/Hooks/BeforeCreateEchoEventHook.php` — the hook interface we'll implement. Tells us the signature.
4. `~/mediawiki/extensions/Echo/includes/Model/Event.php` — read `Event::create()` method. This is the API we call to fire a notification. Note its required parameters (`type`, `agent`, optionally `extra`, `title`).
5. `~/mediawiki/extensions/Echo/includes/Formatters/EchoEventPresentationModel.php` — base class for presentation models. **Read fully.** Critical: understand `getHeaderMessage()`, `getBodyMessage()`, `getPrimaryLink()`, `getIconType()`. Our presentation model subclasses this.
6. `~/mediawiki/extensions/Echo/includes/Formatters/EchoEditUserPagePresentationModel.php` — concrete simple example. ~80 lines. Mirror its structure.

### Your existing T407468 code (what we'll modify)
7. `includes/Download/DownloadJob.php` — your DownloadJob from T407468. We'll add a single line at the success path firing the notification helper. Constructor will gain one optional dependency (the helper).
8. `includes/Download/DownloadRequest.php` — confirm `getUserId()` returns `?int`, used to decide whether to notify.

### Wikispeech existing patterns
9. `extension.json` — see how existing hooks are registered in the `Hooks` and `HookHandlers` sections. Our new hook handler follows the same pattern.
10. `i18n/en.json` and `i18n/qqq.json` — see how existing message keys are structured. Echo requires specific key prefixes: `notification-header-*`, `notification-body-*`, `notification-link-*`.
11. `includes/Hooks/PlayerHooks.php` (or any other existing hook handler) — the registration pattern for hook handler classes.

### Test patterns
12. `tests/phpunit/Download/DownloadJobTest.php` — the test pattern we'll mirror for any new tests.

After reading: hold an internal model. Don't summarize unless asked. Present the design recap (§9 below).

---

## 2. The task — verbatim from Phabricator

**T402528: Subscribe to an article download**
https://phabricator.wikimedia.org/T402528

> This should give a way to be notified when an article (you've requested) is ready to download. A link to the file should be included.
>
> There are several ways the notification could be made:
> - Via Notifications. May only work if Wikispeech is installed on the same wiki. If so it won't work for gadget.
> - As a comment on the user's talk page.
> - As an email. We may not want to allow arbitrary emails. If so this requires the user to have an account with email on the wiki where Wikispeech is installed.
> - RSS feed?

**Direction picked:** Echo (Notifications). Talk-page, email, and RSS deferred per umbrella doc §3.3.

---

## 3. Architecture

### 3.1 The 7 pieces of new/changed code

1. **New hook handler** at `includes/Hooks/EchoHooks.php` — implements `BeforeCreateEchoEventHook`, registers the `wikispeech-download-ready` notification type with Echo at extension load.

2. **New presentation model** at `includes/Notifications/EchoDownloadReadyPresentationModel.php` — extends `EchoEventPresentationModel`, defines what the notification looks like in the bell icon (header text, body text, primary link, icon type).

3. **New helper class** at `includes/Notifications/NotifyDownloadReady.php` — wraps `EchoEvent::create()`. Single static method `notify(UserIdentity $user, int $downloadId, string $fileUrl, Title $pageTitle): void`. No-ops if Echo isn't loaded.

4. **Modify `DownloadJob`** — add one line at the success path that calls `NotifyDownloadReady::notify()` with the request's user, request ID, file URL, and Title. Skip if user is anonymous (`$request->getUserId() === null`).

5. **Modify `extension.json`** — register the hook handler in `HookHandlers` section, add the hook to `Hooks` section. Add `Hooks` to the existing structure.

6. **i18n strings** — add Echo's required keys to `en.json` and `qqq.json`:
   - `echo-category-title-wikispeech` (category label)
   - `notification-header-wikispeech-download-ready`
   - `notification-body-wikispeech-download-ready`
   - `notification-link-wikispeech-download-ready` (download link text)
   - `wikispeech-download-ready-email-subject`
   - `wikispeech-download-ready-email-batch-body`

7. **CHANGES.md entry** for T402528.

### 3.2 The notification firing flow

In `DownloadJob::run()`, after the successful path:

```php
// existing code from T407468
$filePath = $this->computeProducedFilePath( $title );
$this->downloadRequestStore->updateStatus(
    $this->requestId,
    DownloadRequest::STATUS_DONE,
    $filePath
);

// NEW: notify the requesting user
if ( $request->getUserId() !== null ) {
    $user = $this->userFactory->newFromId( $request->getUserId() );
    NotifyDownloadReady::notify(
        $user,
        $request->getRequestId(),
        $this->computeFileUrl( $filePath ),
        $title
    );
}
```

Two new things needed in DownloadJob:
- A `UserFactory` dependency injected via constructor.
- A `computeFileUrl()` private helper that converts the filesystem path to a URL using `$wgUploadPath`. This is finally implementing the "URL conversion" follow-up flagged in T407468.

### 3.3 The helper class

```php
class NotifyDownloadReady {
    public static function notify(
        UserIdentity $user,
        int $downloadId,
        string $fileUrl,
        Title $pageTitle
    ): void {
        if ( !ExtensionRegistry::getInstance()->isLoaded( 'Echo' ) ) {
            return;
        }
        Event::create( [
            'type' => 'wikispeech-download-ready',
            'agent' => $user,
            'title' => $pageTitle,
            'extra' => [
                'download-id' => $downloadId,
                'file-url' => $fileUrl,
                'page-title' => $pageTitle->getPrefixedText(),
            ],
        ] );
    }
}
```

Note: import `MediaWiki\Extension\Notifications\Model\Event` only conditionally — if the `Echo` namespace doesn't exist, the import would still be a problem. The simplest solution is to use the fully qualified class name inside the method, gated by the `isLoaded()` check. That way the file parses fine even if Echo isn't installed. Alternative: use a separate file for the actual `Event::create` call, only loaded when Echo is present. The umbrella doc's spec is the simpler approach — we use it.

### 3.4 The presentation model

Mirror `EchoEditUserPagePresentationModel` for structure. Rough shape:

```php
class EchoDownloadReadyPresentationModel extends EchoEventPresentationModel {

    public function getIconType() {
        return 'wikispeech-download-ready'; // matches the icon registered in BeforeCreateEchoEvent
    }

    public function getHeaderMessage() {
        $msg = $this->msg( 'notification-header-wikispeech-download-ready' );
        $msg->params( $this->event->getExtraParam( 'page-title' ) );
        return $msg;
    }

    public function getBodyMessage() {
        return $this->msg( 'notification-body-wikispeech-download-ready' )
            ->params( $this->event->getExtraParam( 'page-title' ) );
    }

    public function getPrimaryLink() {
        return [
            'url' => $this->event->getExtraParam( 'file-url' ),
            'label' => $this->msg( 'notification-link-wikispeech-download-ready' )->text(),
        ];
    }
}
```

### 3.5 The hook handler

```php
class EchoHooks implements BeforeCreateEchoEventHook {

    public function onBeforeCreateEchoEvent(
        array &$notifications,
        array &$notificationCategories,
        array &$notificationIcons
    ) {
        $notificationCategories['wikispeech'] = [
            'priority' => 3,
            'tooltip' => 'echo-pref-tooltip-wikispeech',
        ];

        $notifications['wikispeech-download-ready'] = [
            'category' => 'wikispeech',
            'group' => 'positive',
            'section' => 'message',
            'presentation-model' =>
                EchoDownloadReadyPresentationModel::class,
            'user-locators' => [
                [ 'EchoUserLocator::locateFromEventExtra', [ 'agent' ] ],
            ],
        ];

        $notificationIcons['wikispeech-download-ready'] = [
            'path' => 'Wikispeech/modules/icons/notification-download-ready.svg',
        ];
    }
}
```

The user-locator says "the user to notify is the one in the event's `agent` field" — which is the user we passed when creating the event. Echo handles delivery from there.

### 3.6 Constructor changes to DownloadJob

Current (from T407468):
```php
public function __construct(
    $title,
    $params,
    $downloadRequestStore,
    $pageFileGenerator,
    $titleFactory,
    $config
)
```

New:
```php
public function __construct(
    $title,
    $params,
    $downloadRequestStore,
    $pageFileGenerator,
    $titleFactory,
    $config,
    $userFactory  // NEW
)
```

`extension.json` `JobClasses.wikispeechDownload.services` becomes:
```json
"services": [
    "Wikispeech.DownloadRequestStore",
    "Wikispeech.PageFileGenerator",
    "TitleFactory",
    "MainConfig",
    "UserFactory"
]
```

### 3.7 The icon

We need an SVG for the bell-icon's notification entry. Minimal placeholder is fine — Sebastian or a designer can refine later. Use a simple speaker/audio icon. Place at `modules/icons/notification-download-ready.svg`. ~20 lines of SVG. Inkscape or hand-write one.

If you can't easily produce one, omit the icon entry from the hook handler and Echo falls back to a generic icon. Acceptable for first patch.

---

## 4. i18n strings

### `i18n/en.json` additions

```json
"echo-category-title-wikispeech": "Wikispeech",
"notification-header-wikispeech-download-ready": "Your audio download of \"$1\" is ready.",
"notification-body-wikispeech-download-ready": "Your Wikispeech audio file for \"$1\" has finished generating and is ready to download.",
"notification-link-wikispeech-download-ready": "Download",
"wikispeech-download-ready-email-subject": "Your Wikispeech audio download is ready",
"wikispeech-download-ready-email-batch-body": "Your Wikispeech audio file for \"$1\" is ready to download: $2"
```

### `i18n/qqq.json` additions (translator notes)

```json
"echo-category-title-wikispeech": "Echo notification category label for Wikispeech-related notifications.",
"notification-header-wikispeech-download-ready": "Echo notification header. Parameters: $1 — title of the article that was synthesised.",
"notification-body-wikispeech-download-ready": "Echo notification body. Parameters: $1 — title of the article.",
"notification-link-wikispeech-download-ready": "Label for the download link in the notification.",
"wikispeech-download-ready-email-subject": "Subject line of the email notification when the user has email-on-Echo enabled.",
"wikispeech-download-ready-email-batch-body": "Email body. Parameters: $1 — title of the article, $2 — URL to the file."
```

---

## 5. Tests

### 5.1 New test file: `tests/phpunit/Notifications/NotifyDownloadReadyTest.php`

Pure unit tests. Verify that:

1. `testNotify_echoNotLoaded_returnsWithoutCallingEchoEvent` — when `ExtensionRegistry::isLoaded('Echo')` returns false, the helper returns silently. Mock the registry. (This is hard to mock because it's a singleton. Acceptable: skip this test if the registry can't be cleanly mocked, document the no-op behavior in the class docblock.)
2. `testNotify_echoLoaded_callsEchoEvent` — when Echo is loaded, the helper calls `Event::create` with the right parameters. This requires running with Echo loaded; mark as `@requires extension Echo` if the test framework supports it.

If Echo's static `Event::create` is hard to spy on (it's static), wrap it: have `NotifyDownloadReady` accept an optional `eventCreator` callable that defaults to `[Event::class, 'create']` but can be overridden in tests. This is a small refactor for testability — mention it in the design recap.

### 5.2 Update `DownloadJobTest`

Add 1-2 tests:

3. `testRun_validRequestWithUser_firesNotification` — mock the new dependency (the notification helper), assert it's called with the expected arguments.
4. `testRun_validRequestAnonymousUser_doesNotFireNotification` — request with `user_id = null`, assert helper is not called.

Update existing test fixtures to provide `userFactory` mock since the constructor signature changed.

### 5.3 Hook registration test

Optional but nice: a test that loads the extension and asserts that the `wikispeech-download-ready` notification type is registered in `$wgEchoNotifications`. Skip if it adds complexity.

---

## 6. Manual end-to-end testing

This is the demo path you'll show your professors.

### 6.1 Apply DB updates

```bash
cd ~/mediawiki
docker compose exec mediawiki php maintenance/run.php update --quick 2>&1 | tail -10
```

Should be a no-op (Echo schema already applied).

### 6.2 Create a test user with email

In the wiki UI, log in as Admin. Go to Special:CreateAccount. Make a new user `TestUser` with email `test@example.com`. Confirm email if your local stack has email confirmation enabled.

Or via maintenance script:

```bash
docker compose exec mediawiki php maintenance/run.php createAndPromote --force TestUser testpassword
```

### 6.3 Log in as TestUser via curl

```bash
cd ~
rm -f cookies-testuser.txt

LOGIN_TOKEN=$(curl -s -c cookies-testuser.txt -b cookies-testuser.txt \
    "http://localhost:8080/w/api.php?action=query&meta=tokens&type=login&format=json" \
    | python3 -c "import sys, json; print(json.load(sys.stdin)['query']['tokens']['logintoken'])")

curl -s -c cookies-testuser.txt -b cookies-testuser.txt \
    -X POST "http://localhost:8080/w/api.php" \
    -d "action=login" \
    -d "format=json" \
    -d "lgname=TestUser" \
    -d "lgpassword=testpassword" \
    --data-urlencode "lgtoken=$LOGIN_TOKEN" | python3 -m json.tool
```

Expected: `{"login": {"result": "Success", ...}}`.

### 6.4 Create a download request as TestUser

Get a CSRF token (now logged-in this won't be `+\`):

```bash
TOKEN=$(curl -s -c cookies-testuser.txt -b cookies-testuser.txt \
    "http://localhost:8080/w/api.php?action=query&meta=tokens&format=json" \
    | python3 -c "import sys, json; print(json.load(sys.stdin)['query']['tokens']['csrftoken'])")
echo "Token: $TOKEN"
```

Create the download:

```bash
curl -s -c cookies-testuser.txt -b cookies-testuser.txt \
    -X POST "http://localhost:8080/w/api.php" \
    -d "action=wikispeech-download-request" \
    -d "format=json" \
    -d "page=Main_Page" \
    -d "language=en" \
    -d "voice=cmu-slt-hsmm" \
    --data-urlencode "token=$TOKEN" | python3 -m json.tool
```

Note the `request_id`.

### 6.5 Run the job

```bash
cd ~/mediawiki
docker compose exec mediawiki php maintenance/run.php runJobs --type wikispeechDownload 2>&1
```

Watch for the `good` status. The notification fires inside the job.

### 6.6 Check the notification appears

```bash
curl -s -c cookies-testuser.txt -b cookies-testuser.txt \
    "http://localhost:8080/w/api.php?action=echonotifications&format=json&notformat=model" \
    | python3 -m json.tool
```

Expected: a notification object with `category: 'wikispeech'` and `type: 'wikispeech-download-ready'`. The `*` field (rendered notification text) should include the page title and a download link.

### 6.7 Check it appears in the UI

Open `http://localhost:8080/wiki/Main_Page` in a browser, log in as TestUser. Look at the top-right corner — there should be a bell icon with a red badge showing `1`. Click it. The notification appears with the body text and a Download link.

**Click Download. The browser should download/open the `.opus` file.** This is the demo moment.

### 6.8 Test anonymous-user case (no notification)

Log out. Create a download as anonymous (the same curl as in T407468 testing). Run the job. Check `echonotifications` API — anonymous users get no notifications because they can't have any. Verify by creating the request when not logged in: the `wsd_user_id` column will be null, and the job will skip the notify call.

### 6.9 Test Echo-not-loaded case (graceful fallback)

Temporarily disable Echo:

```bash
# Comment out Echo in LocalSettings
sed -i.bak 's/^wfLoadExtension( "Echo" )/\/\/ wfLoadExtension( "Echo" )/' ~/mediawiki/LocalSettings.php
```

Run a download request and job. The download should still complete successfully (just no notification). Re-enable:

```bash
mv ~/mediawiki/LocalSettings.php.bak ~/mediawiki/LocalSettings.php
```

Verify: the job's success path doesn't fatal-error when Echo is missing.

---

## 7. Workflow — strict ordering

### Step A — Read

Use `view` on every file in §1. Don't skip any. Don't summarize unless asked.

### Step B — Recap design to Nityamittal

In chat, present:
- The 7-piece architecture (§3.1).
- How the notification flows through the job (§3.2).
- Approach for the static `Event::create` testability problem (§5.1).
- Whether you'll include the SVG icon or skip it (§3.7).
- Any deviations from the brief based on what you learned reading the code.

Wait for explicit "go" or adjustments.

### Step C — Implement, file by file

Order:

1. `includes/Notifications/NotifyDownloadReady.php` — the helper.
2. `includes/Notifications/EchoDownloadReadyPresentationModel.php` — the renderer.
3. `includes/Hooks/EchoHooks.php` — the registration.
4. `i18n/en.json` and `i18n/qqq.json` — message keys.
5. `extension.json` — register the hook handler, update JobClasses services.
6. `includes/Download/DownloadJob.php` — add the notification call, UserFactory dependency.
7. `tests/phpunit/Download/DownloadJobTest.php` — add 2 tests, update fixtures.
8. `tests/phpunit/Notifications/NotifyDownloadReadyTest.php` — new file.
9. `modules/icons/notification-download-ready.svg` — optional simple SVG.
10. `CHANGES.md` entry.

After each: *"Here's the diff for X. OK to continue?"*

### Step D — Self-check

Walk through §14 of `wikispeech-code-style.md`. 12 questions. Flag anything.

### Step E — Tests

```bash
cd ~/mediawiki
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/Download/ 2>&1 | tail -20
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/Notifications/ 2>&1 | tail -20
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/integration/Download/ 2>&1 | tail -20
```

All green.

### Step F — Lint

```bash
chmod -x ~/mediawiki/extensions/Wikispeech/docs/*.md
cd ~/mediawiki
docker compose exec mediawiki bash -c "cd extensions/Wikispeech && composer test" 2>&1 | tail -20
```

All three stages clean.

### Step G — Manual E2E (the demo)

Walk through §6 step by step. Show me each command's output. The bell icon notification appearing in the browser is the success criterion — that's what you'll show your professors.

### Step H — Final review

- `git status`
- `git diff --stat`
- Summary of what's in the patch.

Wait for explicit approval before commit.

### Step I — Commit

Commit message in §8 below. Use the printf-to-file pattern from T407468 (VS Code keeps stripping blank lines):

```bash
printf 'Add Echo notification when a download file is ready\n\n[body...]\n\nBug: T402528\n' > /tmp/T402528-commit-msg.txt
```

Use `cat -n /tmp/T402528-commit-msg.txt` to verify blank lines are present, then `git commit -F /tmp/T402528-commit-msg.txt`.

### Step J — DO NOT push yet

This branch waits for Sebastian's response on T402528 Phab. **Do not run `git review` until Nityamittal explicitly says so.** The commit sits locally.

---

## 8. Commit message

```
Add Echo notification when a download file is ready

Integrates with the Echo (Notifications) extension to alert the
requester via the bell icon when a Wikispeech audio download has
finished generating. Echo's email-on-event mechanism delivers an
email automatically if the user has opted in via their preferences.

Echo is a soft dependency: if it is not loaded, the notification
helper is a no-op and the download feature continues to work on
wikis without Echo installed.

Anonymous users do not receive notifications, since Echo
notifications require a target user. Anonymous downloaders can
poll the wikispeech-download-status API to discover completion.

Talk-page and RSS notification mechanisms discussed on the task
were considered and deferred. Cross-wiki notification support
(for gadget-mode users) is not addressed by this patch — those
users can rely on the status API.

Adds a new wikispeech-download-ready notification type registered
via the BeforeCreateEchoEvent hook, a presentation model for
rendering, and a NotifyDownloadReady helper that wraps EchoEvent
creation and isolates Echo-specific knowledge from DownloadJob.

Bug: T402528
```

---

## 9. Phab comment Nityamittal posts after Sebastian agrees and patch is pushed

```
Initial patch up: https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/[NUMBER]

Implements Echo (Notifications) integration as discussed. The job from T407468 fires a `wikispeech-download-ready` event on success; Echo handles bell-icon display and (if user opts in) email delivery.

Notes for review:

1. Echo is a soft dependency — wikis without Echo installed see the helper no-op silently. Acceptance criterion §3.7 of the umbrella doc.
2. Anonymous users skip notification (no target user). They can poll the status API.
3. The `file_url` field in DownloadRequest is now converted to a public URL via $wgUploadPath (this also addresses the deferred URL-conversion item from T407468's review notes).
4. Talk-page, RSS, and cross-wiki notification mechanisms remain out of scope per design discussion.

Tested locally end-to-end: created request as a logged-in user, ran job, notification appeared in the bell icon with a working download link to a playable .opus file. Anonymous-user and Echo-not-loaded fallbacks both verified.
```

---

## 10. Quality bar

Same as previous patches. A senior reviewer should not be able to tell this was AI-assisted. Match Sebastian's terse comment style. Match the test naming convention. `@since 0.1.15` everywhere. No unnecessary abstraction.

Specific to this patch: the presentation model is the place where reviewers will look hardest, because it touches user-facing UI. Make sure the message text reads naturally and the link target is correct.

---

## 11. Begin

Start with Step A. View every file in §1. After reading, jump to Step B.
