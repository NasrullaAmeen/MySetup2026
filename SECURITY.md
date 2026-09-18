# Security Policy

This is a private, personal notes repo (see [LICENSE](LICENSE)) - there is
no public issue tracker or bounty program for it.

## Scope

The docs here describe home lab/network topology (private IP ranges,
interface names, VM layout) for personal reference. Real credentials
(Wi-Fi passwords, API keys, tokens) should never appear in this repo -
existing docs use placeholders (e.g. `psk="..."`, `wpa-psk "YOURPASSWORD"`).

## Reporting an issue

If you're the owner and you spot a real secret committed by mistake:

1. Rotate/change that credential immediately (removing it from git history
   does not undo prior exposure).
2. Remove it from the file(s) and commit the fix.
3. Rewrite history to purge it if it was ever pushed (e.g. `git filter-repo`
   or BFG Repo-Cleaner), then force-push.

If someone else finds a leaked secret in this repo, please reach out to the
repo owner directly rather than opening a public issue.
