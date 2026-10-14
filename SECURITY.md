# Security — xchan08.github.io (bio site)

## Architecture & trust boundaries

```
Google Sheet "Bio Site CMS"  --(publish to web, CSV)-->  Google public CSV endpoint
        ^                                                        |
        | (write: owner's Google account only)                   | (read: anyone)
                                                                 v
GitHub Actions "sync-sheet" (hourly cron + manual) --commits--> data/*.csv in this repo
                                                                 |
                                                                 v
GitHub Pages serves index.html + data/*.csv  -->  visitors' browsers (static, same-origin)
```

- **No server, no database, no secrets, no analytics, no cookies.** The site is static files.
- **Write paths to content:** exactly two — the owner's Google account (sheet) and the owner's
  GitHub account (repo). Nothing else can change what the site shows.

## Defenses in the renderer (index.html)

1. **Output escaping.** Every sheet-sourced string passes through `esc()` (HTML-escapes
   `& < > "`) before entering the DOM. `**bold**` markup is applied only after escaping, so
   sheet text can never become live HTML/JS.
2. **URL scheme allowlist.** All sheet-sourced URLs (`avatar_url`, `resume_url`, `linkedin`,
   `github`, `call_url`) pass `safeUrl()`: only `http(s)://`, `mailto:`, `tel:`, or bare domains
   (auto-prefixed with `https://`) are accepted. `javascript:`, `data:`, `vbscript:` yield empty
   string, which hides the element instead of rendering it. Emails pass `safeMail()` format check.
3. **No eval / no raw innerHTML of unescaped data.** The only innerHTML assignments receive
   strings built from escaped values.
4. **Workflow least privilege.** `sync-sheet.yml` uses only the built-in `GITHUB_TOKEN` with
   `contents: write`, commits `data/*.csv` only, and fetches from the fixed published-sheet URL.

## Threat model

| Threat | Outcome | Mitigation |
|---|---|---|
| Attacker edits sheet values (needs Google account) | Text defacement only — cannot execute code in visitors' browsers (escaping + scheme allowlist) | 2FA on Google account; git history + sync commits give audit trail & instant rollback |
| Poisoned URL field in sheet (`javascript:` etc.) | Element hidden, not rendered | `safeUrl`/`safeMail` allowlist (verified by ad-hoc poison-fixture tests) |
| Attacker compromises GitHub account | Full site control | 2FA on GitHub account; rename/restore from history |
| Visitor-side attack surface | Minimal: static HTML/CSS/JS, same-origin CSV fetch, no third-party scripts except Google Fonts stylesheet | Keep renderer dependency-free |
| Supply chain | Workflow pins `actions/checkout@v4`; no other actions | Review workflow file on change |

## Incident response

1. **Freeze content:** Sheet ▸ File ▸ Share ▸ Publish to web ▸ *Stop publishing* (site keeps last synced CSVs).
2. **Roll back:** revert the offending sync commit on `main` (or push a corrected `data/*.csv`).
3. **Rotate:** change Google/GitHub passwords, revoke sessions, re-enable 2FA.
4. Re-publish only after the sheet is clean.

## Reporting

Issues: open a GitHub issue on this repo or contact the owner via the site's Contact card.
