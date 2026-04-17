# Wikispeech Code Style Guide

A practical reference for writing PHP that matches the Wikispeech codebase. Patterns extracted from `SpeechoidConnector.php`, `ApiWikispeechListen.php`, `extension.json`, and `tests/phpunit/SpeechoidConnectorTest.php`. When in doubt, **open the nearest neighbor file in the directory you're working in and copy its style** — that beats any rule in this document.

This guide is a defense against AI-generated code "tells." It is not a substitute for reading the surrounding code each time you write a new class.

---

## 1. File header

Every PHP file starts with this exact pattern:

```php
<?php

namespace MediaWiki\Wikispeech\<SubNamespace>;

/**
 * @file
 * @ingroup Extensions
 * @license GPL-2.0-or-later
 */

use SomeImport;
use AnotherImport;

/**
 * Brief one-line description of what the class does.
 *
 * Optionally a longer paragraph if the class has subtle responsibilities.
 *
 * @since 0.1.x
 */
class ClassName {
    // ...
}
```

Notes:
- `@ingroup Extensions` — every file. API files also add `@ingroup API` above it.
- `@license GPL-2.0-or-later` — exactly this string, no variations.
- `use` statements alphabetically ordered, blank line before the class docblock.
- `@since` on the class itself is the version it was *introduced*. Don't bump it on later edits.

---

## 2. Class structure

Order inside a class:
1. Properties (private, with `@var` docblocks).
2. Constructor.
3. Public methods.
4. Private/protected methods.

Properties are declared with PHPDoc `@var` even when the type is also given in the property declaration. Sebastian's code mostly uses `private $foo;` with `/** @var Type */` above it, *not* `private Type $foo;` — i.e. it predates property type declarations and hasn't been migrated. **Match that style** unless you're touching a class that already uses typed properties.

```php
/** @var Config */
private $config;

/** @var string Speechoid URL, without trailing slash. For non queued (non-TTS) operations. */
private $url;
```

The `@var` line often carries a one-sentence explanation of the property's role. Use that habit — saves having a separate comment.

---

## 3. Constructors and dependency injection

Constructors take dependencies; do **not** fetch services from `MediaWikiServices::getInstance()` inside business logic. The wiring happens in `extension.json`:

```json
"Wikispeech.YourNewService": {
    "class": "...",
    "services": [
        "Wikispeech.SpeechoidConnector",
        "Wikispeech.VoiceHandler",
        "MainConfig"
    ]
}
```

The constructor receives the dependencies in the same order as the services array:

```php
public function __construct(
    SpeechoidConnector $speechoidConnector,
    VoiceHandler $voiceHandler,
    Config $config
) {
    $this->speechoidConnector = $speechoidConnector;
    $this->voiceHandler = $voiceHandler;
    $this->config = $config;
}
```

`SpeechoidConnector`'s constructor uses untyped parameters (`$config, $requestFactory`) — that's older style. **For new code, use typed parameters.** API modules in `Api/` are the model.

The constructor docblock format Sebastian uses:

```php
/**
 * @since 0.1.5
 * @param Config $config
 * @param HttpRequestFactory $requestFactory
 */
public function __construct( $config, $requestFactory ) {
```

— `@since`, then `@param` for each parameter, no `@return` on constructor. **Description-less `@param` lines are normal here.** Don't write `@param Config $config The configuration object` — it's noise.

---

## 4. Method docblocks

Public method docblock template:

```php
/**
 * One-line description, ending with period.
 *
 * Optional second paragraph for non-obvious behavior, edge cases,
 * or links to related methods via @see or @link to a Phab task.
 *
 * @since 0.1.x
 * @param string $foo Optional short clarification if the name isn't enough.
 * @param int|null $bar
 * @return array Map of language => voice
 * @throws SpeechoidConnectorException On Speechoid I/O- or JSON parse errors.
 */
```

Conventions specifically observed in Wikispeech:

- **`@since` is mandatory** on every public method. Use the current extension version (check `extension.json` → `version`). For unreleased work, use the next planned bump (e.g. if `version` is `0.1.14`, use `@since 0.1.15`).
- **`@param` lines are bare when the parameter name is self-evident.** A `@param string $language` line is complete; you don't need "the language code". Add description only when the parameter name is ambiguous.
- **`@return` description is encouraged when the return type is complex.** E.g. `@return array Map language => voice` adds real information; `@return int Result count` does not.
- **`@throws` is required for any thrown exception**, with a brief reason: `@throws SpeechoidConnectorException On Speechoid I/O errors.`
- **Don't write `@author` tags.** Wikipedia/MediaWiki uses git history for that.

Private methods often get a much shorter docblock or even none:

```php
/**
 * Converts the output from {@link parse_url} to an URL.
 *
 * @since 0.1.10
 * @param array $parsedUrl
 * @return string
 */
private function unparseUrl( array $parsedUrl ): string {
```

That's representative — a one-line description and bare `@param`/`@return`.

---

## 5. Naming conventions

Drawn from the actual codebase — match these exact patterns:

| Concept | Wikispeech convention | NOT |
|---|---|---|
| Language code | `$language` | `$languageCode`, `$lang` (except as API parameter) |
| Voice name | `$voice` | `$voiceName`, `$voiceId` |
| Method that returns a map | `listDefaultVoicePerLanguage()` | `getDefaultVoicesByLanguage()` |
| Method that returns one item | `findLexiconByLocale()` | `getLexiconForLocale()`, `lookupLexicon()` |
| Method that fetches from Speechoid | `requestX()`, `synthesize()`, `lookupX()` | `fetchX()`, `getXFromService()` |
| Boolean methods | `isQueueOverloaded()` | `checkIfQueueIsOverloaded()`, `getIsOverloaded()` |
| Constants | `NS_PRONUNCIATION_LEXICON` | camelCase |

Method-name verb choice is meaningful here:
- **`request...`** = a method that calls Speechoid via HTTP and returns the raw response shape.
- **`list...`** = a method that returns a parsed/typed collection.
- **`find...`** = returns one item or null.
- **`lookup...`** = returns one item, may throw on missing.
- **`get...`** = returns a property or computed value, no I/O.
- **`synthesize...`**, **`addX`**, **`deleteX`**, **`updateX`** = state-changing operations.

---

## 6. Comment style

Wikispeech's comment style is **terse and useful, not explanatory of the obvious.** Specifically:

**Do:**
```php
// whitespace and sentence ends counts too
$charactersInSegment += 1;

// Get the symbol set to convert to
$lexicon = $this->findLexiconByLanguage( $language );

// If successful, returns something like:
// deleted entry id '11' from lexicon 'sv'
// where the lexicon is the second part of the lexicon name:lang.
```

**Don't:**
```php
// Increment counter
$counter++;

// Loop through items
foreach ( $items as $item ) {

// Return the result
return $result;
```

Comments inside method bodies are sparse. They show up where the *intent* of the next line is non-obvious — usually because of a Speechoid/MediaWiki quirk. They're written like a senior engineer reminding themselves of a non-obvious thing, not like documentation.

Multi-line `/* ... */` comments inside method bodies are rare. Use `//` consistently.

`@todo` is used freely for known-incomplete handling:
```php
// @todo how do we know if this was successful? Always return 200
```

That's the right tone — "I know this is wrong, here's what I'd want to know."

---

## 7. Error handling

Two patterns, used distinctly:

### Pattern A — throw exceptions for I/O failures and programmer errors

This is the dominant style in `SpeechoidConnector`:

```php
if ( !$responseString ) {
    throw new SpeechoidConnectorException(
        'Unable to communicate with Speechoid. ' .
        $this->haproxyQueueUrl . var_export( $options, true )
    );
}
```

Use `SpeechoidConnectorException` for Speechoid I/O. Use `InvalidArgumentException` for programmer errors (bad input parameters):

```php
if ( $words === [] ) {
    throw new InvalidArgumentException( 'Must contain at least one word' );
}
```

Use `RuntimeException` for unexpected runtime conditions. Don't invent new exception classes unless there's a real need.

### Pattern B — return `Status` for operations the caller will want to handle conditionally

When a method is part of a higher-level workflow where partial failure is meaningful:

```php
public function lookupLexiconEntries(
    string $lexicon,
    array $words
): Status {
    // ...
    return FormatJson::parse( $responseString );
}
```

`Status::newGood( $value )`, `Status::newFatal( $reasonString )`. Look at `addLexiconEntry()` for a model of accumulated validation — it doesn't throw for "no ids" or "multiple ids", it returns `Status::newFatal()` so the caller sees a well-typed response.

**Choice rule:** I/O failures and contract violations throw. Domain-level "this didn't work, but we expected it might not" returns Status. When unsure, throw — Sebastian's code throws more often than it returns Status.

### Don't pre-validate things you don't have to

This is anti-AI advice. Sebastian's code trusts its callers within the extension. Don't write:

```php
// AI tendency:
public function synthesize( $language, $voice, $parameters ) {
    if ( !is_string( $language ) ) {
        throw new InvalidArgumentException( '$language must be a string' );
    }
    if ( empty( $language ) ) {
        throw new InvalidArgumentException( '$language is required' );
    }
    // ...
}
```

Instead use type declarations on the signature (`string $language`) and let bad callers fail fast naturally. PHP's type system handles the easy validations. Reserve runtime validation for *semantic* problems the type system can't catch (empty arrays of words, malformed JSON, missing array keys in deserialized data).

---

## 8. HTTP and JSON patterns

When calling Speechoid (or any HTTP service):

```php
$responseString = $this->requestFactory->get( $url, [], __METHOD__ );
if ( !$responseString ) {
    throw new SpeechoidConnectorException( 'Unable to communicate with Speechoid.' );
}
$status = FormatJson::parse(
    $responseString,
    FormatJson::FORCE_ASSOC
);
if ( !$status->isOK() ) {
    throw new SpeechoidConnectorException( 'Unexpected response from Speechoid.' );
}
$value = $status->getValue();
if ( !is_array( $value ) ) {
    throw new SpeechoidConnectorException( 'Unexpected non-array response' );
}
return $value;
```

Note the four-step pattern repeated throughout `SpeechoidConnector`:
1. Make request.
2. Check for empty/false response.
3. Parse JSON via `FormatJson::parse(..., FormatJson::FORCE_ASSOC)`.
4. Validate the parsed shape (`is_array`, `array_key_exists`).

`__METHOD__` is **always** passed as the third argument to `$requestFactory->get/post`. It's used for logging.

`FormatJson::FORCE_ASSOC` is used everywhere — Wikispeech consistently treats JSON objects as PHP associative arrays, not stdClass objects.

---

## 9. Configuration access

Read config via the injected `Config` object:

```php
$voices = $this->config->get( 'WikispeechVoices' );
$language = $parameters['lang'];
$validLanguages = array_keys( $voices );
```

**Never** access `$wgFoo` globals directly. Use `Config::get()`. If you need a config key that doesn't exist yet, add it to `extension.json` under `config:` with a `description` (an array of lines for multi-line) and a default `value`.

For a new config key, copy the structure of an existing one:

```json
"WikispeechDownloadAnnouncementEnabled": {
    "description": [
        "If true, downloaded audio files include a brief introduction ",
        "stating the article name, source wiki, and license."
    ],
    "value": true
}
```

Don't introduce new globals (`$wgX`) without going through this mechanism.

---

## 10. i18n (internationalization)

Every user-visible string goes through `wfMessage()` or `$this->msg()` (in API modules), never inlined as PHP strings. New message keys go in `i18n/en.json` with English text and in `i18n/qqq.json` with a description for translators.

Key-naming convention:
- Prefix all keys with `wikispeech-`.
- For sub-features, add a section: `wikispeech-download-...`, `wikispeech-error-...`, `wikispeech-lexicon-...`.
- Use kebab-case (hyphens, not underscores or camelCase).
- API error messages: prefix with `apierror-wikispeech-`. Example: `apierror-wikispeech-listen-invalid-language`.
- Preference labels: prefix with `prefs-wikispeech-`.

Example for a download announcement (T403152):
```json
"wikispeech-download-announcement-intro": "This is a Wikispeech reading of \"$1\" from $2.",
"wikispeech-download-announcement-license": "Content is available under $1."
```

`$1`, `$2` etc. are positional parameters. In PHP:
```php
$introText = wfMessage( 'wikispeech-download-announcement-intro' )
    ->params( $articleTitle, $siteName )
    ->inLanguage( $language )
    ->text();
```

Use `->inLanguage( $language )` when synthesising audio — the message must match the voice's language, not the wiki's interface language.

`qqq.json` description for translators (don't skip this — it's required for translatewiki):
```json
"wikispeech-download-announcement-intro": "Spoken introduction at the start of a downloaded article audio file. $1 is the article title, $2 is the wiki name (e.g. 'English Wikipedia')."
```

---

## 11. Tests

Test classes go in `tests/phpunit/` (integration) or `tests/phpunit/unit/` (pure unit, no DB). Pick **unit** unless your code touches the database, file system, network, or MediaWiki services. Sebastian prefers unit tests where possible — they're faster and more reliable.

### Test file naming and structure

```php
<?php

namespace MediaWiki\Wikispeech\Tests;

/**
 * @file
 * @ingroup Extensions
 * @license GPL-2.0-or-later
 */

use MediaWiki\Wikispeech\YourClass;
use MediaWikiIntegrationTestCase; // or MediaWikiUnitTestCase for unit tests

/**
 * @group medium
 * @covers \MediaWiki\Wikispeech\YourClass
 */
class YourClassTest extends MediaWikiIntegrationTestCase {
    // tests go here
}
```

- `@group medium` for integration tests (`small` for unit tests).
- `@covers` with the **fully qualified class name including the leading backslash**.

### Test method naming

The Wikispeech convention from `SpeechoidConnectorTest.php` is **distinctive** and you must match it exactly to avoid an AI tell:

```php
public function testListDefaultVoicePerLanguage_speechoidHasDefaultVoices_givesVoicesPerLanguage() {
public function testFindLexiconByLanguage_speechoidHasTextProcessors_givesLexiconsPerLanguage() {
```

Format: `test<MethodName>_<scenarioInWords>_<expectedOutcomeInWords>`

- Underscores between three sections.
- Each section is camelCase.
- Sections are descriptive but not full English sentences.

Avoid:
- `testGetVoiceReturnsDefaultWhenLanguageNotProvided()` — too sentence-like, wrong format.
- `test_get_voice_default()` — all-snake_case is wrong.
- `testGetVoiceDefault()` — missing the scenario/expectation split.

### Mocking pattern

Use `createPartialMock` to stub specific methods on the real class, leaving the rest intact:

```php
$connectorMock = $this->createPartialMock(
    SpeechoidConnector::class,
    [ 'requestDefaultVoices' ]
);
$connectorMock
    ->expects( $this->once() )
    ->method( 'requestDefaultVoices' )
    ->willReturn( $defaultVoicesJson );

$defaultVoicePerLanguage = $connectorMock->listDefaultVoicePerLanguage();

$this->assertCount( 4, $defaultVoicePerLanguage );
$this->assertEquals( 'stts_sv_nst-hsmm', $defaultVoicePerLanguage['sv'] );
```

Note:
- `createPartialMock`, not `createMock`, when you only want to stub one or two methods.
- `expects( $this->once() )` is common; don't use it gratuitously, only when call count matters.
- Multiple `assertEquals` lines, one per important value. Don't try to assert the whole array equality — failures are easier to read this way.

### Test fixtures

JSON fixtures are usually inlined as heredoc-style strings inside the test:

```php
$defaultVoicesJson = '[
    {
        "default_voice": "cmu-slt-hsmm",
        "lang": "en"
    }
]';
```

Don't move them to separate fixture files unless they're really long (>50 lines).

### What to test

Focus on:
- **Inputs that the type system can't catch:** empty arrays, missing keys, malformed JSON.
- **Branches:** if your method has 3 distinct paths, write at least 3 test cases.
- **Error conditions:** assert that the right exception type is thrown for the right input.

Don't test:
- Type coercion that PHP handles (`string $foo` will reject integers natively).
- Constructor assignment (assign-and-read tests are noise).
- Other classes' behavior — mock them instead.

---

## 12. Service wiring

When you add a new service-like class (a class that's reused, holds dependencies, and would be expensive to construct ad-hoc), wire it through `includes/ServiceWiring.php`. Pattern:

```php
// includes/ServiceWiring.php
return [
    'Wikispeech.DownloadAnnouncement' => static function ( MediaWikiServices $services ): DownloadAnnouncement {
        return new DownloadAnnouncement(
            $services->getService( 'Wikispeech.SpeechoidConnector' ),
            $services->getService( 'Wikispeech.VoiceHandler' ),
            $services->getMainConfig()
        );
    },
    // ...
];
```

Then list it in any consumer's `services:` array in `extension.json`.

A class that's only ever instantiated once at a known place (e.g., a job or maintenance script) doesn't need service wiring.

---

## 13. Commit message format

Mandatory format for Wikimedia Gerrit:

```
Short imperative subject, max ~70 chars

Longer explanation paragraph. What does this change do, and *why*?
The "why" is more important than the "what" — the diff already shows
the what. Wrap at 72 chars.

If there are tradeoffs or things you considered and rejected, note them.

Bug: T403152
Change-Id: I0000000000000000000000000000000000000000
```

Wikispeech-specific norms:

- **Subject:** imperative mood ("Add download announcement class", not "Added" or "Adds").
- **Don't prefix the subject with a feature name** — the Wikispeech repo doesn't use `[wikispeech] ...` prefixes.
- **Bug:** line is required for any non-trivial change. Use `Bug: TXXXXXX` (Phab task ID, capital T).
- **Change-Id:** is added automatically by the `commit-msg` git hook. Don't write it yourself; it's there after `git review -s`.

For T403152, your subject might be:
```
Add DownloadAnnouncement for downloaded article audio
```

Body:
```
Provides spoken introduction text (article name, site name, content
license) to be prepended to downloaded article audio. The
DownloadAnnouncement class produces the audio for the introduction;
prepending it to the merged file is handled by the download job
(T407468).

Sound logo and producer-mode handling are deferred — the former pending
design decisions on its intensity, the latter pending the broader
download flow design in T402522.

Bug: T403152
```

---

## 14. The smell test — review your patch before pushing

Before `git review`, walk through this checklist. Each "no" is a red flag:

1. **Did I open a neighbor file in the same directory and read 30 lines?** If no, do it now and adjust.
2. **Are my method names verb-typed correctly?** (`request...` vs `list...` vs `find...` vs `get...`)
3. **Are my variable names matching the codebase?** (`$language` not `$languageCode`, `$voice` not `$voiceName`)
4. **Are my comments terse and useful?** Did I delete any "Increment counter" or "Loop through items" style comments?
5. **Are my docblocks bare on `@param` lines unless the parameter name is genuinely ambiguous?**
6. **Do I have `@since 0.1.x` on every public method and on the class?**
7. **Did I write tests in the `test<Method>_<scenario>_<outcome>()` format?**
8. **Did I add `@group medium` (integration) or `@group small` (unit) and `@covers \\Fully\\Qualified\\Class`?**
9. **Are i18n strings in `i18n/en.json` AND `i18n/qqq.json`?**
10. **Did I avoid pre-validating things the PHP type system already catches?**
11. **Did I use `Config::get()` rather than `$wg...` globals?**
12. **Does the commit message have `Bug: TXXXXXX` and an imperative-mood subject under 70 chars?**

If anything fails, fix it before pushing — `git commit --amend` is your friend.

---

## 15. Anti-patterns (specifically AI-generated tells to avoid)

These are the things that flag a patch as machine-written. Each one corresponds to something I've seen AI produce that doesn't match Wikispeech's actual style.

| Anti-pattern | Why it stands out | What to do instead |
|---|---|---|
| Section-header comments inside methods (`// === Validate input ===`) | Sebastian's code never does this. | Break into separate methods if a section is meaningful enough to label. |
| Docblock on every private one-liner | Wikispeech privates often have minimal or no docblocks. | Match the surrounding density. |
| `// Returns the audio buffer` above `return $audio;` | Sebastian's code doesn't restate `return`. | Delete the comment. |
| Try/catch around lines that don't throw | Defensive paranoia. | Let it not catch what it can't throw. |
| Validating parameters the type system catches (`if (!is_string($x))`) | Type declarations make this redundant. | Use `string $x` in the signature. |
| `$result`, `$response`, `$data` as variable names | Generic AI fallback. | Pick something descriptive: `$utterance`, `$speechoidResponse`, `$voicesByLanguage`. |
| `getXForY()` or `findXByY()` when codebase uses `findYsX()` | Slight naming-pattern mismatch. | Mirror the surrounding files exactly. |
| Long, English-sentence test method names | Wrong format for this codebase. | `testMethod_scenario_outcome()`. |
| `array_map` + arrow function where neighbors use `foreach` | Subtle stylistic difference. | Use `foreach` to match the codebase. |
| Excessive use of `match` / `enum` (PHP 8 features) | Wikispeech uses older patterns mostly. | Match the surrounding code. Use new features only where neighbors do. |
| Catch-all `Throwable` for things that should fail loudly | AI over-defends. | Catch specific exception types. |
| New interfaces / abstract classes / factories not matched in surrounding code | AI over-architects. | Plain classes unless polymorphism is genuinely needed. |

---

## 16. When to deviate from this guide

This guide describes the *current* style. Sometimes there's a reason to break it:

- If you're writing in a directory with a different style (e.g., `Utterance/` vs `Api/`), match that subdirectory's local conventions over the global ones.
- If you're modernizing a section of code (e.g., adding type declarations), do it consistently within that file rather than half-and-half.
- If a real architectural reason calls for an interface or factory, use one — but write a sentence in the commit message explaining why.

Don't break a convention "because it's better" without explanation. Sebastian and Lokal_Profil have spent years getting this codebase to a consistent shape; uniformity is itself a value.

---

## 17. Quick cheatsheet

```php
<?php

namespace MediaWiki\Wikispeech\Subdir;

/**
 * @file
 * @ingroup Extensions
 * @license GPL-2.0-or-later
 */

use Config;
use MediaWiki\Wikispeech\SpeechoidConnector;

/**
 * Brief description.
 *
 * @since 0.1.15
 */
class MyClass {

    /** @var SpeechoidConnector */
    private $speechoidConnector;

    /** @var Config */
    private $config;

    /**
     * @since 0.1.15
     * @param SpeechoidConnector $speechoidConnector
     * @param Config $config
     */
    public function __construct(
        SpeechoidConnector $speechoidConnector,
        Config $config
    ) {
        $this->speechoidConnector = $speechoidConnector;
        $this->config = $config;
    }

    /**
     * Brief description of what this does.
     *
     * @since 0.1.15
     * @param string $language
     * @param string $voice
     * @return array Synthesized audio response from Speechoid.
     * @throws SpeechoidConnectorException On Speechoid I/O errors.
     */
    public function doThing( string $language, string $voice ): array {
        $text = $this->buildIntroText( $language );
        return $this->speechoidConnector->synthesizeText( $language, $voice, $text );
    }

    /**
     * @since 0.1.15
     * @param string $language
     * @return string
     */
    private function buildIntroText( string $language ): string {
        $siteName = $this->config->get( 'Sitename' );
        return wfMessage( 'wikispeech-mything-intro' )
            ->params( $siteName )
            ->inLanguage( $language )
            ->text();
    }
}
```

That's the shape. Open `SpeechoidConnector.php` next to your editor when you start writing — the closer you mirror it, the less reviewer friction you'll see.
