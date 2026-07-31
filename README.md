# no-locales branch

Reproduces the ta-na-mao case: a repository with JSON files but no locale
files anywhere. Connecting this branch must fail with diagnosis
`no-locale-files`, not with a suggestion to look in another directory.

See the README on `main` for the full scenario list.
