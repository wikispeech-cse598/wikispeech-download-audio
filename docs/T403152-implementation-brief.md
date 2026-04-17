# T403152 — Implementation Brief for Claude Code

> **For the Claude Code agent reading this:** This brief is a complete handoff from a planning conversation. Read it in full before doing anything. Then read the linked code files in the repo. Then, before writing implementation code, present a design recap to the human overseer (Nityamittal) for approval. Iterate until approved. Only then write code. Do not skip the design-approval step.

---

## 1. Context you need

You're working in `~/mediawiki/extensions/Wikispeech` — a MediaWiki extension that adds text-to-speech to wiki pages. The maintainer is Sebastian_Berlin-WMSE (Wikimedia Sverige). Patches go through Wikimedia Gerrit at https://gerrit.wikimedia.org.

Before writing code, **you must read these files in full** via the `view` tool:

1. `docs/wikispeech-code-style.md` — authoritative style guide. Naming, comment density, error handling, test format, commit message format. Failure to match this guide causes review friction.
2. `docs/wikispeech-download-audio-tasks.md` — umbrella brief covering all 5 sub-tasks under T397023. §4 is for T403152.
3. `includes/SpeechoidConnector.php` — HTTP client to the TTS backend. Your new class will call `synthesizeText()` on this. Read fully.
4. `includes/Api/ApiWikispeechListen.php` — closest existing analog. Note specifically the default-voice fallback pattern around the `$voiceHandler->getDefaultVoice()` call. Read fully.
5. `includes/VoiceHandler.php` — the class that resolves default voices. Read fully so you understand what it returns and when it returns null.
6. `includes/ServiceWiring.php` — how services are registered. You'll add an entry. Read fully.
7. `extension.json` — manifest. You'll add a config key under `config:` and verify namespace mapping in `AutoloadNamespaces`. Skim relevant sections.
8. `i18n/en.json` and `i18n/qqq.json` — for the new message keys. Skim to learn the file structure and key-naming conventions.
9. `tests/phpunit/SpeechoidConnectorTest.php` — model for test class structure, naming, and mocking patterns.

Don't summarise these to the human in chat unless asked — your reading is for your own model. After reading, jump to Step B.

---

## 2. The task — verbatim from Phabricator

**T403152: Include information about license and such**
https://phabricator.wikimedia.org/T403152

> Read by the TTS. It should probably go at the start.
>
> Things that could or should be included:
> - Licens. Can be generated from a template, like the one on ENWP
> - The sound logo. It's a bit intense.
> - Something about Wikispeech and that the speech is from a TTS.

**Parent task:** T397023 ☂ Download audio
https://phabricator.wikimedia.org/T397023

The parent is an umbrella for letting users download whole-article audio files. T403152 is the sub-task that produces the announcement that plays at the start of those downloads.

**Linked references Sebastian implicitly pointed at:**

- *"Like the one on ENWP"* refers to: https://en.wikipedia.org/wiki/Wikipedia:Reusing_Wikipedia_content#Example_notice
  - The canonical template form is: *"This article uses material from the Wikipedia article \"\[Article Title\]\", which is released under the Creative Commons Attribution-Share-Alike License 4.0."*
  - **Mirror this structure in our wording.** Substituting "audio version" for "article" because the medium is different.

- The sound logo lives at: https://meta.wikimedia.org/wiki/Wikimedia_Foundation/Communications/Sound_Logo
  - It's a 4-second audio clip, CC BY-SA 4.0, but also a registered Wikimedia Foundation trademark.
  - Per Wikimedia Brand guidelines: most uses require a trademark agreement.
  - For T403152, **do not bundle the audio file or enable it by default.** Add a config hook (see §4.7) so the maintainer can enable it later when trademark + UX are resolved.

---

## 3. Scope decisions already made (don't relitigate)

- **Class name:** `DownloadAnnouncement`
- **Location:** `includes/Download/DownloadAnnouncement.php`
- **Namespace:** `MediaWiki\Wikispeech\Download` — new sub-namespace. `AutoloadNamespaces` already maps `MediaWiki\Wikispeech\` to `includes/`, so this works without any change to that mapping.
- **Class produces audio for the announcement only.** Prepending it to the merged article download file is T407468's responsibility. Don't try to handle file merging here.
- **License source:** `$wgRightsText` only. **Not** deriving from `$wgRightsUrl` (URLs synthesise badly via TTS, URL → name mapping is brittle).
- **Empty `$wgRightsText` fallback:** omit the license sentence. Don't fabricate. Don't throw. Use a separate i18n key for the no-license variant of the announcement.
- **Sound logo:** config hook only (off by default). Don't bundle audio. Don't auto-enable.
- **Producer mode (`consumer-url` parameter):** out of scope. Defer to T402522's broader work.
- **POC handling:** out of scope per Sebastian's comment on T402522 ("Since the use of POC is still limited I'd say you can skip it for now").

---

## 4. The design — proposal to recap to Nityamittal, then build

Once you've read the linked files, present this design back to Nityamittal **in chat as text, not yet as code.** Get explicit approval. Adjustments expected. Only after approval, write code.

### 4.1 Class shape

```php
namespace MediaWiki\Wikispeech\Download;

class DownloadAnnouncement {

    public function __construct(
        SpeechoidConnector $speechoidConnector,
        VoiceHandler $voiceHandler,
        Config $config
    );

    /**
     * Synthesise the announcement audio for a downloaded article.
     *
     * @return array Speechoid response with 'audio_data' and 'tokens' keys.
     */
    public function synthesize(
        Title $title,
        string $language,
        ?string $voice = null
    ): array;

    private function buildAnnouncementText(
        Title $title,
        string $language
    ): string;
}
```

### 4.2 Behavior — `synthesize()`

1. If `$voice` is null, call `$this->voiceHandler->getDefaultVoice( $language )`.
2. If voice is still null, throw `ConfigException( 'Invalid default voice configuration.' )` — copy this exact phrasing from `ApiWikispeechListen::execute()` so error messages stay consistent across the codebase.
3. Call `buildAnnouncementText( $title, $language )`.
4. Call `$this->speechoidConnector->synthesizeText( $language, $voice, $announcementText )`.
5. Return the response unmodified.

### 4.3 Behavior — `buildAnnouncementText()`

1. Get `$siteName` from `$this->config->get( 'Sitename' )`.
2. Get `$rightsText` from `$this->config->get( 'RightsText' )`.
3. Get `$titleText` from `$title->getText()`.
4. **If `$rightsText` is non-empty:**
   ```
   $attribution = wfMessage( 'wikispeech-download-announcement-attribution' )
       ->params( $siteName, $titleText, $rightsText )
       ->inLanguage( $language )
       ->text();
   ```
5. **Else (empty `$rightsText`):**
   ```
   $attribution = wfMessage( 'wikispeech-download-announcement-attribution-no-license' )
       ->params( $siteName, $titleText )
       ->inLanguage( $language )
       ->text();
   ```
6. Build the TTS notice:
   ```
   $ttsNotice = wfMessage( 'wikispeech-download-announcement-tts-notice' )
       ->inLanguage( $language )
       ->text();
   ```
7. Return `$attribution . ' ' . $ttsNotice`.

**Why `->inLanguage( $language )`:** the synthesised audio must match the voice's language, not the wiki's interface language. Without this, an English-voice download on a Swedish-interface wiki would synthesise a Swedish announcement in an English voice — unintelligible.

### 4.4 i18n keys to add

In `i18n/en.json` (insert in alphabetical position — check existing file structure):

```json
"wikispeech-download-announcement-attribution": "This audio version uses material from the $1 article \"$2\", which is released under $3.",
"wikispeech-download-announcement-attribution-no-license": "This audio version uses material from the $1 article \"$2\".",
"wikispeech-download-announcement-tts-notice": "The audio was generated by Wikispeech, a text-to-speech extension for MediaWiki."
```

In `i18n/qqq.json` (descriptions for translators):

```json
"wikispeech-download-announcement-attribution": "Spoken attribution at the start of a downloaded article audio file. Modeled after the standard English Wikipedia reuse template at https://en.wikipedia.org/wiki/Wikipedia:Reusing_Wikipedia_content#Example_notice. Parameters:\n* $1 = wiki name (from $wgSitename, e.g. 'English Wikipedia')\n* $2 = article title (e.g. 'Earth')\n* $3 = license text (from $wgRightsText, e.g. 'Creative Commons Attribution-ShareAlike 4.0')",
"wikispeech-download-announcement-attribution-no-license": "Variant of {{msg-mw|wikispeech-download-announcement-attribution}} used when $wgRightsText is empty (the wiki has not configured a license). Parameters:\n* $1 = wiki name (from $wgSitename)\n* $2 = article title",
"wikispeech-download-announcement-tts-notice": "Spoken notice identifying the audio as machine-generated, played after the attribution at the start of a downloaded article audio file."
```

The `{{msg-mw|...}}` syntax is a translatewiki.net convention for cross-referencing. Verify by skimming the existing `qqq.json` to see if Wikispeech actually uses it; if not, drop the `{{msg-mw|...}}` wrapper and write the reference plainly.

### 4.5 Service wiring

Add to `includes/ServiceWiring.php`:

```php
'Wikispeech.DownloadAnnouncement' => static function ( MediaWikiServices $services ): DownloadAnnouncement {
    return new DownloadAnnouncement(
        $services->getService( 'Wikispeech.SpeechoidConnector' ),
        $services->getService( 'Wikispeech.VoiceHandler' ),
        $services->getMainConfig()
    );
},
```

**Verify before writing:** open `ServiceWiring.php` and confirm `Wikispeech.SpeechoidConnector` and `Wikispeech.VoiceHandler` actually exist as service IDs there. If they're named differently, use the actual names. If they aren't service-wired (you instantiate them directly), match that pattern instead.

### 4.6 Tests

Location: `tests/phpunit/unit/Download/DownloadAnnouncementTest.php`

Use `MediaWikiUnitTestCase` (no DB needed). `@group small`. `@covers \MediaWiki\Wikispeech\Download\DownloadAnnouncement` (full backslash-prefixed path).

Tests, using exact `test<Method>_<scenario>_<outcome>` naming (see style guide §11):

1. `testSynthesize_voiceProvided_skipsVoiceHandler` — pass an explicit voice; assert `VoiceHandler::getDefaultVoice` is never called.
2. `testSynthesize_voiceNull_usesDefaultFromVoiceHandler` — pass null voice; assert `getDefaultVoice` is called once with the right language.
3. `testSynthesize_noDefaultVoice_throwsConfigException` — `VoiceHandler` returns null; assert `ConfigException` is thrown with the matching message.
4. `testSynthesize_returnsSpeechoidResponse` — assert the return value matches what `SpeechoidConnector` returned (mocked).
5. `testSynthesize_emptyRightsText_omitsLicenseSentence` — assert that when `$wgRightsText` is empty, the text passed to `SpeechoidConnector::synthesizeText` does NOT contain "released under" (i.e. the no-license variant of the message was used). Capture the `$text` argument with PHPUnit's `$this->callback()` or by setting up the mock to record args.
6. `testSynthesize_withRightsText_includesLicenseInText` — mirror of #5; assert the synthesised text DOES contain the rights text string when configured.

Use `createPartialMock` on `SpeechoidConnector` and `VoiceHandler`. Inline JSON / response fixtures as heredoc strings inside the test method. Don't extract fixture files unless they get truly unwieldy.

### 4.7 Sound logo config hook (no behavior, just the config key)

Add to `extension.json` under `config:`:

```json
"WikispeechDownloadAnnouncementSoundLogoFile": {
    "description": [
        "Optional path or URL to a sound logo audio file (e.g. the Wikimedia ",
        "sound logo) to be prepended to downloaded article audio. ",
        "If empty (default), no sound logo is prepended. ",
        "Note: the Wikimedia sound logo is trademarked; verify your use ",
        "complies with https://meta.wikimedia.org/wiki/Wikimedia_Foundation/Communications/Sound_Logo ",
        "before configuring this. The actual file mixing is performed by ",
        "the download merge job (T407468); this class only exposes the file path."
    ],
    "value": ""
}
```

`DownloadAnnouncement` does **not** read or use this config in its current methods. The config exists so that T407468's merge job can read it and prepend the file. We're adding the config key here because conceptually it belongs with the announcement feature, and putting it in now means T407468 doesn't need to add it later.

If the merge job reads it and the file path is set, the merge job is responsible for prepending the audio. If the file is missing or unreadable when the merge job runs, that's the merge job's problem to log and handle — not ours.

**Mention this in the commit message:** "Adds `WikispeechDownloadAnnouncementSoundLogoFile` config (default empty) for future sound logo support; no behavior tied to it in this patch."

### 4.8 Commit message

```
Add DownloadAnnouncement for downloaded article audio

Provides spoken attribution and TTS-identification text to be played
at the start of downloaded article audio files. The wording follows
the English Wikipedia reuse template at
https://en.wikipedia.org/wiki/Wikipedia:Reusing_Wikipedia_content#Example_notice
adapted for the audio medium ("audio version" rather than "article").

DownloadAnnouncement.synthesize() produces the audio; prepending it
to the merged article file is the download merge job's responsibility
(T407468).

License source: $wgRightsText only. Not derived from $wgRightsUrl
because URLs synthesise badly in TTS and URL-to-name mapping is
brittle across license versions and non-standard URLs. When
$wgRightsText is empty (wiki has not configured a license), the
license sentence is omitted via a separate i18n key rather than
fabricating attribution — matches MediaWiki's own behavior of not
displaying license text on the page footer when unconfigured.

Adds WikispeechDownloadAnnouncementSoundLogoFile config (default
empty) as a hook for future sound logo support; no behavior is tied
to it in this patch. Sound logo audio file is not bundled, pending
trademark and UX decisions per Sebastian's comment on the task.

Bug: T403152
```

---

## 5. Workflow — strict ordering

### Step A — Read

Use `view` on each file listed in §1. Read fully, not skim. After reading, hold an internal model of the codebase before writing.

### Step B — Recap design to Nityamittal

In chat:

> "I've read the briefs and the linked files. Quick design recap before I write code:
>
> - `DownloadAnnouncement` class in `includes/Download/`, namespace `MediaWiki\Wikispeech\Download`.
> - `synthesize(Title, string $language, ?string $voice = null): array`.
> - Default voice fallback via `VoiceHandler`; throws `ConfigException` if none.
> - Text uses ENWP-style template wording. Two sentences: attribution + 'generated by Wikispeech' notice.
> - License from `$wgRightsText` only; if empty, switches to a no-license i18n variant (omits the 'released under' clause).
> - Service wiring as `Wikispeech.DownloadAnnouncement`.
> - 6 unit tests covering voice fallback, response passthrough, and license/no-license branching.
> - Adds `WikispeechDownloadAnnouncementSoundLogoFile` config key (no behavior in this patch, hook for T407468).
>
> Anything to adjust before I start writing?"

Wait for explicit "go" or adjustments.

### Step C — Implement, file by file

In this order, **showing the diff to Nityamittal after each file and waiting for review**:

1. `includes/Download/DownloadAnnouncement.php` — the class itself.
2. `includes/ServiceWiring.php` — add the service entry.
3. `i18n/en.json` — add the three message keys (alphabetical insertion).
4. `i18n/qqq.json` — add the three descriptions.
5. `extension.json` — add the `WikispeechDownloadAnnouncementSoundLogoFile` config key.
6. `tests/phpunit/unit/Download/DownloadAnnouncementTest.php` — the test class.

Don't pile up multiple file changes before checking in. After each: *"Here's the diff for X. OK to continue?"*

### Step D — Self-check against the style guide

Walk through §14 of `wikispeech-code-style.md` (the smell test). Each of the 12 questions should pass. For any that don't, fix and re-verify. Show the answers to Nityamittal as a numbered list.

### Step E — Run tests locally

```bash
cd ~/mediawiki
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/unit/Download/
```

If tests fail, fix and re-run. Show all output to Nityamittal.

### Step F — Lint

Look in the Wikispeech repo root for PHPCS configuration (`.phpcs.xml`, `phpcs.xml`, or `composer.json` script entries) to find the canonical lint command. A common invocation is:

```bash
cd ~/mediawiki
docker compose exec mediawiki bash -c "cd extensions/Wikispeech && composer test"
```

Fix every warning. The Wikispeech codebase is clean — any new warning your patch introduces is an immediate -1 in review.

### Step G — Commit and push (only after explicit approval)

Confirm with Nityamittal that they've reviewed everything and are OK to push. Then:

```bash
cd ~/mediawiki/extensions/Wikispeech
git checkout -b T403152-download-announcement
git add includes/Download/DownloadAnnouncement.php \
        includes/ServiceWiring.php \
        i18n/en.json i18n/qqq.json \
        extension.json \
        tests/phpunit/unit/Download/DownloadAnnouncementTest.php
git commit  # opens editor; paste the commit message from §4.8; Change-Id added by hook
git review
```

`git review` prints a Gerrit URL. Save it — that's the deliverable.

### Step H — Phab comment text (Nityamittal posts, not you)

Give Nityamittal this text to post on https://phabricator.wikimedia.org/T403152:

> Initial patch up: \[Gerrit URL\]
>
> Design notes:
> - Wording follows the ENWP reuse template, adapted for the audio medium.
> - Two sentences: attribution + a short notice that the audio was generated by Wikispeech (covers two of the three bullets in the task description).
> - License is read from `$wgRightsText` only; not deriving from `$wgRightsUrl` because (a) URL → name mapping is brittle across CC versions and non-standard URLs, (b) URLs synthesise terribly in TTS. When `$wgRightsText` is empty, the license sentence is omitted via a separate i18n key rather than fabricating attribution. This matches what MediaWiki itself does in its UI on unconfigured wikis.
> - Sound logo: skipped per your "a bit intense" note. Added a config key `WikispeechDownloadAnnouncementSoundLogoFile` (default empty) as a hook for later. No audio file bundled, no behavior wired up — the actual prepending will be the merge job's job in T407468.
> - Producer mode and POC handling: out of scope, deferred to T402522.
>
> Happy to iterate on text wording, scope, or anything else.

---

## 6. What to ask Nityamittal, when

You are an agent helping an overseer, not autonomously contributing. Ask:

- **Step B:** any design adjustments before writing.
- **After each file in Step C:** "here's the diff, OK to continue?"
- **Before Step E:** "ready to run tests?"
- **Before Step G:** "ready to push to Gerrit?"
- **Whenever uncertain:** stop and ask. Better an obvious question than a wrong assumption.

Do not:
- Push to Gerrit without explicit human approval.
- Post on Phabricator (Nityamittal does that).
- Modify files outside the scope listed in §5 Step C.
- Add features not listed in §4. If you think something is missing, ask first.
- Silently fix something you encounter (e.g. an unrelated lint warning) — surface it.

---

## 7. Quality bar

This patch is being submitted to a real Wikimedia maintainer who will read it carefully and has the right to reject it. Bar: **a senior PHP engineer reviewing this patch should not be able to tell whether it was AI-assisted.** Specifically:

- Method names match the codebase's verb conventions (style guide §5).
- Comments are terse and useful, not narrating obvious code (style guide §6).
- Error handling matches Sebastian's pattern: throw on I/O / config errors, no defensive over-validation (style guide §7).
- Tests use `test<Method>_<scenario>_<outcome>()` format (style guide §11).
- Variable names follow the codebase, not generic AI defaults (`$language` not `$languageCode`, `$voice` not `$voiceName`).
- No `// Returns the result` style comments above `return $result`.
- No defensive validation of things the type system catches (`if (!is_string($x))` is wrong when the signature is `string $x`).
- No section-header comments inside methods.
- No interfaces / factories / patterns the surrounding code doesn't use.
- The patch reads like it belongs in this codebase.

If after Step D (smell test) any of these fail, fix before pushing. If uncertain whether something passes, ask Nityamittal to read it.

---

## 8. Begin

Start with Step A. View each file in §1. Don't summarise unless asked. After reading, jump to Step B.
