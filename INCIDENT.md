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

## 2026-10-22 — product-finder-hierarchy.csv now too large to push at all; judgeme-ratings-supplemental.csv pushed with CRLF stripped. Run FAILED.

This run's full pipeline (Shopify pull, CSV build, sanity checks) completed cleanly — the failure is entirely in STEP 4 (push), and is a worse version of the 2026-09-17 issue now that `product-finder-hierarchy.csv` has grown to ~272KB / 1,544 data rows.

**`product-finder-hierarchy.csv` — push never completed (2 attempts, both crashed):**
- Expected (local, freshly generated): sha `8973331b94dd2d2397103753a3767639941b9778`, 272,221 bytes, 1,545 lines.
- Actual (live on `main`, unchanged): sha `99ed889fd56c13338226fe3778c1d51cc5c89384`, 244 bytes — this is the stale file from a much earlier run, never updated today.
- Direct `git clone`/`git push` was tried first per protocol: clone succeeded, but `git push origin main` was refused with 403 "access denied by the git proxy: tvm-emil/tvm-product-feeds is not in this session's authorized repository set." `gh api` and raw `curl` to the GitHub REST API hit the identical proxy block.
- Fell back to `mcp__github__create_or_update_file` in an isolated subagent, per protocol. Both the first attempt and the one retry crashed mid-generation with "API Error: Claude's response exceeded the 64000 output token maximum" — the file's ~1,545 lines require roughly 68,000 output tokens to reproduce as a literal tool-call argument, which exceeds the hard per-turn output cap. No API call to GitHub was ever completed; the live file is untouched.
- This is a structural limitation, not a flaky error: the file is now simply too large to transcribe through any tool whose content must be generated as model output. It will keep failing on every future run unless the underlying transport changes (see recommendation below). Splitting the file into parts was considered and deliberately NOT done — it is explicitly prohibited by this automation's instructions (it caused the stale `*_part2.csv` / `*_delta.csv` / `*_test.csv` cleanup seen earlier in this repo's history).

**`judgeme-ratings-supplemental.csv` — pushed twice, both times with all CRLF line endings silently stripped to LF:**
- Expected (local, freshly generated): sha `269afa32ffdf614566e499bd6da8b05355c7b30f`, 6,454 bytes, 316 CRLF-terminated lines.
- Actual (live on `main` after 2 push attempts): sha `cdd8fd9c763d833110dec48fcb92712f0c529d9d`, 6,138 bytes (exactly 316 bytes smaller — one `\r` missing per line), LF-only. Content/row values are otherwise correct.
- First push attempt (commit `0274fb7`) produced this LF-only file. Per protocol it was retried once (commit message "... (retry)"); the retry produced the byte-identical LF-only result. A diagnostic sub-task confirmed this is deterministic: no escape sequence (literal `\r\n`, XML `&#13;`, etc.) placed in the `content` argument of `create_or_update_file` / `push_files` survives — the tool-call text channel normalizes all line endings to LF before the file reaches GitHub. This is unrelated to the file's size (6.4KB, well under the output-token ceiling); it is a separate, CRLF-specific normalization in the same write path.
- Per protocol ("if it still does not match... mark the run FAILED. Do not keep retrying beyond that"), no further retries were made. The live file is left in this LF-only state; its data content (sku/rating/review_count values) is correct and complete, only the line-ending byte pattern differs from the local source-of-truth file.

**`install-guides-content-feed.csv` — unchanged, not pushed, matches exactly:**
- Local sha `aaca728d47240b953707cdb123dd7685d14f1c0c` (19,050 bytes) equals the live sha on `main`, so per protocol this file was correctly skipped (no push needed) and needs no fix.

**Overall run status: FAILED.** Two of three feeds are not in their correct, verified state on GitHub as of this run (`judgeme-ratings-supplemental.csv` has a line-ending mismatch only; `product-finder-hierarchy.csv` is still serving the old stale 244-byte file from a prior run — Searchspring/Athos is currently NOT seeing current catalog data for the product finder).

**Recommendation (now more urgent than the 2026-09-17 note):** `product-finder-hierarchy.csv` has crossed the size threshold where it can no longer be pushed through any LLM-content-transcription tool (`create_or_update_file` / `push_files`) at all — not even unreliably, it now hard-fails every time. The only durable fix is a write path that does not require the file's bytes to pass through a model's generated output: either (a) add `tvm-emil/tvm-product-feeds` to this session's authorized git-proxy repository set so the already-working `git clone` → copy file → `git push` path (confirmed functional for read/clone in this run) can complete the commit without any re-typing of content, or (b) a small external job/action (not model-mediated) that pulls the regenerated CSV from a drop location and commits it directly. Either would also fix the CRLF-stripping issue on `judgeme-ratings-supplemental.csv`, since that only occurs on the same model-mediated write path.
