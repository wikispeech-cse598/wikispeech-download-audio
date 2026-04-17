# Local Demo Wiring — Integrating DownloadAnnouncement into DownloadJob

This document captures the local-only integration between T403152 (DownloadAnnouncement) and T407468 (DownloadJob) that we performed for demo purposes. This wiring is NOT in any pushed Gerrit patch — it's a follow-up task to be filed after Sebastian reviews the parent patches.

## What's missing upstream

T403152 introduces a class `DownloadAnnouncement` that synthesises a short spoken intro (article title, site name, license). T407468 introduces `DownloadJob` that produces the full article audio file. In the upstream patches, these don't call each other: the job produces audio without any announcement, and the announcement class is built but never invoked from the download flow.

Reason: each patch had its own reviewable scope. Coupling them would have ballooned T407468's diff and made review harder. The integration is a small follow-up.

## Local-only integration (for demo)

Three files changed:

1. **`extension.json`** — add `Wikispeech.DownloadAnnouncement` to the `wikispeechDownload` job's services list so the job gets the announcement class via DI
2. **`includes/ServiceWiring.php`** — register `Wikispeech.DownloadAnnouncement` service if not already present
3. **`includes/Download/DownloadJob.php`** — accept `DownloadAnnouncement` as a constructor dependency, call its `synthesize()` method after `PageFileGenerator::makePageFile()` produces the article audio, and prepend the resulting Opus bytes to the produced file

The prepend is done by raw Ogg stream concatenation: the announcement's `.opus` bytes are written to the front of the final file, followed by the article body. Opus-in-Ogg supports chained streams natively, so any standards-compliant player handles the result correctly.

The prepend is wrapped in a best-effort try/catch: if announcement synthesis fails (e.g. Speechoid unavailable, missing voice configuration), the failure is logged and the article-only file is preserved. The user still gets their download.

## Follow-up Phabricator task

Title: **Wire DownloadAnnouncement into DownloadJob**

Body outline:
- Reference T403152 and T407468 as the parent patches
- Note the design-time decision to defer integration
- Describe the integration: constructor injection, call site, concat approach
- Flag the Opus Vorbis comment metadata follow-up as related (machine-readable attribution, not just audible)

We'll file this task after both parent patches have been reviewed and merged by Sebastian.
