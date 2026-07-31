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

## Scenarios that MUST fail

A clean failure is the feature here. The error should name where it looked and
what it found, and the activity row should carry a diagnosis.

| # | Base path | Structure | Source language | Expected failure |
|---|---|---|---|---|
| 10 | `namespace-folders` | **locale folders** (deliberately wrong) | `en` | fails. Diagnosis `structure-mismatch`, suggesting `namespace_folders` |
| 11 | `tooling-only` | single file | `en` | fails. Diagnosis `no-locale-files` — JSON is present but none of it is a locale file |
| 12 | `empty-dir` | single file | `en` | fails. Diagnosis `no-json-under-path` |
| 13 | *(repository root)* | single file | `fr` | fails. Diagnosis `source-language-mismatch`, listing `en` and `tr` as the locales that do exist |
| 14 | `does-not-exist` | single file | `en` | fails. Diagnosis `no-json-under-path`, and the path is absent from the tree |
| 15 | `zh-hant-alias` | single file | `zh-hans` | fails. Simplified is not Traditional, so the alias must NOT match |

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
