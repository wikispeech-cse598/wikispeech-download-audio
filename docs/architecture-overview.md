# Architecture Overview

The Download Audio feature is structured as five separable patches that together implement a request → job → deliver workflow. This document sketches how they fit together.

## Component map

```
User browser
    │
    ▼
Special:DownloadPageAudio        ← T402526 (SpecialDownloadPageAudio)
    │
    │ [POST: request new download]
    ▼
ApiWikispeechDownloadRequest     ← T402522 (request API)
    │
    │ [insert row, enqueue job]
    ▼
wikispeech_download_request      ← T402522 (schema + DownloadRequestStore)
    │
    │
    ▼
JobQueue: wikispeechDownload     ← T407468 (DownloadJob)
    │
    │ [ 1. synthesize announcement ]
    │       ↓ uses DownloadAnnouncement ← T403152
    │ [ 2. synthesize article audio ]
    │       ↓ uses PageFileGenerator (existing)
    │ [ 3. concatenate: announcement + article ]
    │ [ 4. write .opus file ]
    │ [ 5. update request status to done ]
    │ [ 6. fire Echo event ]
    │       ↓ uses EchoEvent::create   ← T402528
    ▼
Notification in user's bell
    │
    │ [user clicks notification]
    ▼
Downloaded .opus file
```

## Why separate patches

Each piece can be understood, reviewed, and merged independently. T402526's Special page tests mock the API module; T402522's API tests mock the store; T407468's job tests mock the audio pipeline. Each patch's tests don't need the others' code to pass.

Dependencies at merge time:
- T402526 → T402522 (the Special page dispatches to the API)
- T407468 → T402522 (the job processes rows in the table defined by the API patch)
- T402528 → T407468 (the notification fires from the job)
- T403152 is independent — the announcement class produces audio but nothing in the base patch set calls it

## The T403152 ↔ T407468 integration gap

By design, T407468's DownloadJob synthesises the article body but doesn't call DownloadAnnouncement. Wiring the two together is a small follow-up:

1. Inject DownloadAnnouncement into DownloadJob's constructor
2. After the article audio is produced, synthesise the announcement
3. Concatenate announcement bytes + article bytes as a chained Ogg Opus stream
4. Overwrite the final file

For our local demo we did exactly this — see `local-demo-wiring.md`. The upstream patch is deferred until Sebastian reviews the two parent patches and decides on final approach.

## MediaWiki integration points

- **Special page**: registered in `extension.json` under `SpecialPages`, accessible at `Special:DownloadPageAudio`
- **API module**: registered under `APIModules` as `action=wikispeech-download-request`
- **Background job**: registered under `JobClasses` as `wikispeechDownload`, picked up by MediaWiki's JobQueue and runnable via `maintenance/run.php runJobs --type=wikispeechDownload`
- **Database table**: `wikispeech_download_request`, created via a `LoadExtensionSchemaUpdates` hook handler
- **Echo notification type**: `wikispeech-download-ready`, registered via Echo's `BeforeCreateEchoEvent` hook with a soft dependency (skipped silently if Echo isn't installed)
- **Services**: `Wikispeech.DownloadRequestStore`, `Wikispeech.DownloadAnnouncement` (T403152), wired in `includes/ServiceWiring.php`
