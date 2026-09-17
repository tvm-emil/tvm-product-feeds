# Incident log — Athos feed automation

## 2026-09-17 — product-finder-hierarchy.csv repeatedly corrupted on push, now fixed

**Not a sanity-check failure** (row counts were fine: 2118/408/39 rows, all above threshold), but worth flagging because it is a *recurring* problem, not a one-off.

**What happened:** the scheduled run(s) today pushed `product-finder-hierarchy.csv` (~2,118 rows, ~83KB) via `mcp__github__push_files` / `create_or_update_file`. Because these tools require the file content to be generated inline by an LLM call, large pushes are unreliable: earlier attempts today (commits between `b901ef8` at 11:37 and `1102bd5` at 16:00) silently truncated the file to as few as 559 of 2,119 lines, or corrupted embedded quote characters (rows with wheelbase values like `144"`/`170"`), while still reporting "success" — the GitHub write succeeds even when the content handed to it is wrong. The same pattern (multi-part "recovery"/"combine" commits) appears on 2026-09-15 and 2026-09-16 in the commit history, so this has been happening for at least 3 days.

**What I did this run:** after pushing, I verified every file byte-for-byte against the freshly-generated source (via a local git clone + field-level CSV diff, not just trusting the push tool's self-reported success). This caught:
- `product-finder-hierarchy.csv` truncated to 559/2119 lines — re-pushed and re-verified, now matches exactly (commit `323e948`).
- `install-guides-content-feed.csv` — one row had the word "REQUIRED:" dropped from its `content` field during transcription — fixed and re-verified (commit `5b65069`).
- `judgeme-ratings-supplemental.csv` — matched exactly on first push, no fix needed.

**Current state (verified byte-exact against source data as of this run):**
- `product-finder-hierarchy.csv` — 2118 data rows, 83,556 bytes, blob sha `96608824a7c63ee3f2a9a0de3614c38664741c89`
- `judgeme-ratings-supplemental.csv` — 408 data rows, 8,640 bytes, blob sha `3922f828c1b4d84375b64190e6d0c2562c6aba4c`
- `install-guides-content-feed.csv` — 39 data rows (54 in-scope products still lack a guide, unchanged/expected), 16,987 bytes, blob sha `b7b2d0ebae2bffc6589331e9d981a5adee0fe8bd`

**Recommendation:** the root cause is that content must be generated as model output tokens to push via these MCP tools, and large files (>~40-50KB) are not reliably transcribed this way — sometimes truncated, sometimes silently missing a word. A direct `git push` from the automation's shell would sidestep this (no re-typing of content), but this session's git-credential proxy currently refuses `tvm-emil/tvm-product-feeds` ("not in this session's authorized repository set"). Adding this repo to the session's authorized git sources (or building a small dedicated CI job that regenerates and pushes the feed without going through an LLM content-transcription step) would remove the need for this manual byte-level verification every run.
