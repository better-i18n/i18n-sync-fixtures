# i18n-sync-fixtures

Test fixtures for Better i18n's GitHub sync. Every directory here is a real
repository layout we have seen from a customer, including the ones that broke.

Nothing in this repo is a real product. It exists so a sync can be exercised
against a known-good expectation instead of against a customer's private repo.

## Why this repo exists

"Source language files (X) not found" was the most common sync failure: 71
failed jobs across 10 organizations. Working out the cause meant pulling each
customer's tree by hand, and it turned out to be three unrelated problems that
all produced the same sentence:

| cause | example | fix |
|---|---|---|
| source language code does not match the filename | project `zh-hant`, file `zh-TW.json` | locale aliasing |
| file structure chosen wrong | `faq/en.json` read as `{locale}/{namespace}` | connect-time check |
| repository has no locale files at all | only `package.json` at the root | connect-time check |

Each directory below isolates one of those.

## Scenarios

Connect this repository, then create one project per row. "Structure" is the
file-structure option picked when connecting.

| # | Base path | Structure | Source language | Expected |
|---|---|---|---|---|
| 1 | *(repository root)* | single file | `en` | imports `en` + `tr`. `package.json`, `package-lock.json`, `tsconfig.json` and `keys.json` must NOT be treated as locales |
| 2 | `locales` | single file | `en` | imports `en`, `tr`, `de` |
| 3 | `zh-hant-alias` | single file | `zh-hant` | imports, matching `zh-TW.json` by alias. **This is the twbc case: it fails before the aliasing fix and passes after** |
| 4 | `zh-hans-alias` | single file | `zh-hans` | imports, matching `zh-CN.json` by alias |
| 5 | `locale-folders` | locale folders | `en` | imports `en` (namespaces `common`, `auth`) + `de` |
| 6 | `namespace-folders` | namespace folders | `en` | imports `en` (namespaces `faq`, `home`) + `fr` |
| 7 | `pt-fallback` | single file | `pt-br` | imports `pt.json` via the base-language fallback |
| 8 | `nested` | single file | `en` | imports, and the key format is detected as nested (not flat) |
| 9 | `deep/apps/web/src/i18n/locales` | single file | `en` | imports. A deep monorepo path must work |
| 16 | `region-tokens` | single file | `en` | imports, matching `en-US.json`. **This is the CapitalLayer case: a bare source code reaching a region file.** `ja` matches `ja-JP.json`, `zh-hant` matches `zh-TW.json` |
| 17 | `region-both` | single file | `en` | imports `en.json`, **not** `en-US.json`. An exact filename always wins; the region file must never be preferred when the bare one exists |
| 18 | `region-ambiguous` | single file | `pt-br` | imports `pt-BR.json` exactly. Compare with scenario 19 |

## Scenarios that MUST fail

A clean failure is the feature here. The error should name where it looked and
what it found, and the activity row should carry a diagnosis.

All of these are verified against the real tree of this repository.

| # | Base path | Structure | Source language | Expected failure |
|---|---|---|---|---|
| 10 | `namespace-folders` | **locale folders** (deliberately wrong) | `en` | Diagnosis `structure-mismatch`, suggesting `namespace_folders`. Reading the tree the wrong way resolves the locale as `faq` |
| 11 | `tooling-only` | single file | `en` | Diagnosis `wrong-base-path`, pointing at `locales`. See the note below |
| 12 | `empty-dir` | single file | `en` | Diagnosis `wrong-base-path`, pointing at `locales` |
| 13 | *(repository root)* | single file | `fr` | Diagnosis `source-language-mismatch`, listing `en` and `tr` as the locales that do exist |
| 14 | `does-not-exist` | single file | `en` | Diagnosis `wrong-base-path`. The path is absent from the tree |
| 15 | `zh-hant-alias` | single file | `zh-hans` | Diagnosis `source-language-mismatch`. Simplified is not Traditional, so the alias must NOT match |
| 19 | `region-ambiguous` | single file | `pt` | Diagnosis `source-language-mismatch`, listing `pt-br` and `pt-pt`. **Failing here is the feature.** A repository shipping both has deliberately separated two products, and picking Brazilian because CLDR ranks it first would choose one of them for the customer |

### Why 16 works but 19 must not

Both are "a bare code, only region files present", and the difference is whether
the repository itself distinguishes variants of that language.

`en` and `en-US` both maximize to `en-Latn-US` under CLDR likely-subtags, so
scenario 16 matches with tier `maximized`. This is the direction RFC 4647 Lookup
deliberately does not cover — it walks specific to general only, never the
reverse — which is why the naive implementation left CapitalLayer with four
failed imports and nothing to change.

Maximization is not truncation, and that is what keeps it safe:

- `en-GB` maximizes to `en-Latn-GB`, so it is **not** pulled in for an `en`
  project. Different content, not a different spelling.
- `region-both` never reaches the tier at all, because `en.json` matched exactly
  (scenario 17).
- `region-ambiguous` refuses, because two files claim the same base language
  (scenario 19).

### Round-tripping: 16 also tests the publish direction

Import and publish must agree, or a publish lands in a file the import never
read. Scenario 16 has a token policy the platform infers from the repository
itself (`language_region`, hyphen, BCP-47 case, confidence `unique` since all
three files agree), so adding a target language writes:

| Add | Writes | Not |
|---|---|---|
| `fr` | `region-tokens/fr-FR.json` | `fr.json` |
| `de` | `region-tokens/de-DE.json` | `de.json` |

A bare `fr.json` next to `en-US.json` is a file the customer's app never reads.

Worth knowing while testing: the stored language code is not the file token with
its region removed. `zh-TW` is Traditional Chinese, stored as `zh-hant`, while a
bare `zh` maximizes to Simplified (`zh-CN`). Anything recovering a language by
splitting the token on `-` turns this directory's Traditional file into a
Simplified one silently.

### Why 11, 12 and 14 report `wrong-base-path`

Because this repository *does* hold locale files, just not at the path being
tested. "Your locales are in `locales`" is the more useful answer than "there
are no locale files here", so the diagnosis prefers it.

That means the `no-locale-files` and `no-json-under-path` causes cannot be
reproduced on `main` at all — every path here has locale-bearing siblings. Use
the **`no-locales` branch** for those.

## The `no-locales` branch

A repository with JSON files but no locale files anywhere, which is the
ta-na-mao case. Connect this branch instead of `main`:

| Base path | Source language | Expected |
|---|---|---|
| *(repository root)* | `pt-br` | Diagnosis `no-locale-files` — 4 JSON files present, none of them a locale |
| `src` | `en` | Diagnosis `no-json-under-path` |
| `does-not-exist` | `en` | Diagnosis `no-json-under-path` |

Switching branches is also the cheapest way to test the "right path, wrong
branch" failure: point the base path at `locales` while connecting
`no-locales`.

## What this repo cannot test

The tree here is small, so it cannot reproduce a truncated tree. GitHub's Git
Trees API returns `truncated: true` with an incomplete listing past roughly
100k entries or a 7MB response, which would silently hide locale files in a
large monorepo. Testing that needs a repository big enough to trip the limit.

## Offline equivalent

Most of the above also runs with no network, no credentials and no deploy, by
feeding a file list to the real classifier:

```
bun test ./apps/sync-worker/src/integrations/__tests__/github-import-flow.test.ts
bun test ./apps/sync-worker/src/processors/utils/__tests__/diagnose-source-resolution.test.ts
```

Use this repository for what those cannot cover: the GitHub App installation,
queue retries, the activity rows, and the connect UI.
