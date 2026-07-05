# T403152 — Responding to Patchset 1 Review (Viktoria Hillerud, WMSE)

Change: https://gerrit.wikimedia.org/r/c/mediawiki/extensions/Wikispeech/+/1270609
Reviewer verdict: no vote, 4 unresolved code comments + 1 unresolved overall comment
("Over all well implemented and well documented! I just have some small comments…").
CI is Verified +2. This is the friendly "fix these and we're good" review — respond
within a day or two while context is fresh.

The plan: address all four code comments in one amended commit, upload Patchset 2,
reply "Done" on each thread, and wait for Sebastian's pass.

---

## 1. The four comments and their fixes

All code snippets below are indicative — adapt to the actual Patchset 1 code
(`git review -d 1270609` gives you the exact tree).

### 1.1 `DownloadAnnouncement.php` line ~74 — exception should name the language

> "This could include the language for better troubleshooting."

Patchset 1 copied `ApiWikispeechListen`'s exact phrasing:

```php
throw new ConfigException( 'Invalid default voice configuration.' );
```

She's right that at this call site the language is known and losing it makes the
error useless for debugging a per-language voice config problem. Change to:

```php
throw new ConfigException(
    "Invalid default voice configuration for language \"$language\"."
);
```

Note: this diverges from the `ApiWikispeechListen` phrasing deliberately, at the
reviewer's request — no need to also change the API class (out of scope for this
patch). If the test `testSynthesize_noDefaultVoice_throwsConfigException` asserts
the exact message string, update it (`expectExceptionMessage`) to match.

### 1.2 `DownloadAnnouncementTest.php` (file-level) — mocks should include `tokens`

> "Should some of the mocked response include tokens as well, to match the
> documented return shape?"

`synthesize()`'s docblock promises a Speechoid response with `'audio_data'` and
`'tokens'` keys, but every mock returns only `[ 'audio_data' => '' ]`. Make at
least the passthrough test (`testSynthesize_returnsSpeechoidResponse`) use a
realistic shape, mirroring what Speechoid actually returns:

```php
$response = [
    'audio_data' => 'ZGF0YQ==',
    'tokens' => [
        [ 'orth' => 'This', 'expanded' => 'This', 'endtime' => 250 ],
    ],
];
$this->speechoidConnector->method( 'synthesizeText' )
    ->willReturn( $response );
// …
$this->assertSame( $response, $announcement->synthesize( $title, 'en' ) );
```

Check `SpeechoidConnectorTest.php` for an existing token fixture and copy its
field names exactly rather than inventing them. The other tests, where the return
value is irrelevant, can keep the minimal `[ 'audio_data' => '' ]` — she said
"some of," not "all."

### 1.3 Test line ~85 — assert the default voice is actually used

> "Good to verify the fallback to getDefaultVoice(), but we could also assert
> that the returned voice is actually passed on to synthesizeText()."

In `testSynthesize_voiceNull_usesDefaultFromVoiceHandler`, the mock currently
only verifies `getDefaultVoice( 'en' )` is called and returns `'en-US'` — nothing
proves `'en-US'` then flows into the synthesis call. Tighten the
`synthesizeText` mock:

```php
$this->speechoidConnector->expects( $this->once() )
    ->method( 'synthesizeText' )
    ->with( 'en', 'en-US', $this->anything() )
    ->willReturn( [ 'audio_data' => '' ] );
```

(Match the real parameter order of `SpeechoidConnector::synthesizeText()` —
verify against the actual signature before writing `->with()`.)

### 1.4 Test line ~164 — don't assert on English message wording

> "This seems a bit brittle. If the i18n message should change, the test could
> begin to fail, even if the test itself is correct. Maybe it could be better to
> check that the no-licence path is used, or that no rights text is included."

The current assertion greps the synthesized text for the literal English phrase
`'released under'`, which breaks the moment anyone rewords
`wikispeech-download-announcement-attribution`. Assert on the **parameter**, not
the message wording — the rights text value is under the test's control:

```php
// testSynthesize_emptyRightsText_omitsLicenseSentence
// RightsText configured as '' in the config mock.
$this->callback( static function ( $text ) {
    return !str_contains( $text, 'CC BY-SA 4.0' );
} )
```

…is still wrong (empty rights text means there's nothing to search for). The
robust version: in the *empty* case assert the text equals / matches the
no-license message output, or — simpler and wording-proof — configure a sentinel
in the *positive* test and assert its presence, and in the empty test assert the
text passed to `synthesizeText()` contains no sentinel:

```php
// Positive test: RightsText = 'SENTINEL LICENSE 9.9'
return str_contains( $text, 'SENTINEL LICENSE 9.9' );

// Empty test: RightsText = ''
return !str_contains( $text, 'SENTINEL' );
```

That checks exactly what the code promises — "the license sentence is built from
$wgRightsText, and omitted when it's empty" — without depending on any English
word from the i18n file. If you'd rather check the message-key branch directly,
mocking the message localizer is the alternative, but the sentinel approach is
smaller and matches the existing test style.

---

## 2. Uploading Patchset 2 (the mechanics)

From your Wikispeech checkout (`~/mediawiki/extensions/Wikispeech`):

```bash
# 1. Get the exact Patchset 1 commit onto a local branch
git review -d 1270609

# 2. Make the four fixes above; run tests + lint
docker compose exec mediawiki composer phpunit -- extensions/Wikispeech/tests/phpunit/unit/Download/
docker compose exec mediawiki bash -c "cd extensions/Wikispeech && composer test"

# 3. Amend — do NOT create a new commit, and keep the Change-Id line
#    (Change-Id: I888de74a334b0b97104611c5fcf4d4d2b294ecfe) intact
git add -A
git commit --amend
# The commit message body doesn't need changes for these fixes —
# they're test/message tweaks, not design changes.

# 4. Push — becomes Patchset 2 of the same change
git review
```

If master has moved since April, `git pull --rebase origin master` before
step 3, or just use the Rebase button in the Gerrit UI afterward.

## 3. Replying on Gerrit (do this — silence looks like abandonment)

After Patchset 2 is up, on the change page:

1. Open each of the four comment threads → click **Reply** → write "Done"
   (or a one-liner if you deviated, e.g. on 1.4 explain the sentinel approach)
   → mark **Resolved**.
2. On her overall comment, reply with a short thank-you + summary, e.g.:
   > Thanks for the review! All four addressed in PS2: exception message now
   > includes the language, the passthrough test mocks the full response shape
   > incl. tokens, the default-voice test asserts the voice reaches
   > synthesizeText(), and the license tests now assert on the RightsText
   > parameter instead of message wording.
3. Click **Reply** at the top → **Send** (drafts aren't visible until sent).
4. Add Viktoria and Sebastian to the **attention set** so it lands back in
   their queue.

Jenkins will re-run CI on the new patchset automatically; check it goes
Verified +2 again.

## 4. After this patch

- The same review pattern will likely hit the sibling patches (1270914,
  1271020, 1271979) — the `tokens`-shape and wording-brittleness comments
  generalize; consider preemptively auditing those tests before Sebastian
  reviews them.
- Once merged, file the follow-up Phabricator task for wiring
  DownloadAnnouncement into DownloadJob (see README "Known follow-ups").
