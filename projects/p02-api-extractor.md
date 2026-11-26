> Part of **01 · PYTHON (Days 1–6)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P02 — API Extractor · Days 4–6

### 1. Goal
Pull data from an API that fights back — paginated, rate-limited, authenticated — and be able to stop halfway, restart, and neither lose a record nor fetch one twice.

### 2. Why this project exists
Files are the easy case. Most real source data sits behind an API with a rate limit and no "give me everything" button. The concept you're really learning is the **watermark**: remembering where you got to, so tomorrow's run only fetches what's new. Every incremental pipeline you ever build rests on this.

### 3. Data source
**GitHub REST API** — `https://api.github.com`. Nothing to download; you *are* the extractor.
```bash
# Create a classic PAT, public_repo scope: github.com/settings/tokens
export GITHUB_TOKEN=ghp_...
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/rate_limit | python -m json.tool
```
Unauthenticated: 60 req/hour. With a token: 5,000. Target `/events`, `/repos/{org}/{repo}/commits`, and `/search/repositories` — that last one has a *stricter separate* limit, which is useful: it'll break naive retry logic early.

### 4. What you build

**Step 1 — Fetch one page.** Call `/repos/apache/spark/commits`, print the count. Confirm your token works.

**Step 2 — Paginate to exhaustion.** Follow the `Link: <...>; rel="next"` header until it's absent. Never guess page numbers.

**Step 3 — Handle rate limits properly.** On HTTP 429 or 5xx, retry with exponential backoff **plus jitter** (random 0–1s added, so parallel workers don't retry in lockstep). Read `X-RateLimit-Remaining` and `X-RateLimit-Reset` and sleep until reset when you're near zero. Cap at 5 attempts, then fail loudly.

**Step 4 — Write raw, untouched.** Save responses exactly as received to `data/raw/date=YYYY-MM-DD/`. No parsing, no cleaning. **Raw stays raw** — when you discover a parsing bug in six weeks, you reparse instead of refetching.

**Step 5 — Add the watermark.** Persist the last timestamp or cursor to `state/watermark.json` **after each page succeeds**, not at the end. On start, read it and resume.

**Step 6 — Prove the resume works.**
```bash
python -m src.extract --since 2024-01-01 --out data/raw
# ctrl-C mid-run, then:
python -m src.extract --resume
python -m src.verify_no_gaps --out data/raw
```

**Step 7 — Keep a full-refresh escape hatch.** `--full-refresh` ignores the watermark and starts over. Watermarks drift; you will need this.

### 5. The AI part — API Docs → Retry Policy Reviewer (~50 lines)

Write your retry logic by hand first. Then paste the GitHub rate-limit documentation **and your implementation** into one call and ask: *which documented signals does this code ignore?*

Generic backoff ignores API-specific hints. Claude will likely spot that you're sleeping a fixed interval instead of honouring `X-RateLimit-Reset`, or that you treat 403 as fatal when GitHub uses it for secondary rate limits.

**Also this project:** write your first `CLAUDE.md` — stack, conventions, test command, what not to touch. This file is the highest-leverage prompt you will ever write, because it's prepended to every future request in the repo.

### 6. What you learn
**Python:** `httpx`/`requests` · session reuse · retry and backoff with jitter · env-var secrets
**Data engineering:** pagination · rate limits · **watermarks and incremental extraction** · raw-zone immutability · resumability · full-refresh as a safety valve
**AI:** giving the model documentation as context, not just code · `CLAUDE.md` as persistent project context

### 7. Done checklist
- [ ] Paginates to the end without guessing page numbers
- [ ] Survives a 429 and continues
- [ ] Ctrl-C mid-run, restart, no gaps and no duplicates
- [ ] Watermark persists per page, not per run
- [ ] `--full-refresh` works
- [ ] No secret anywhere in the repo (`git grep ghp_` returns nothing)
- [ ] `CLAUDE.md` exists; `AI_NOTES.md` records what the reviewer caught

### 8. In production this would be
Backoff in a shared client library, not rewritten per pipeline · watermarks in a metadata table with a `run_id`, so two workers can't race · every call logged with latency and status for cost and SLA monitoring · raw landing lifecycled to cold storage after 90 days · an alert when extraction volume drops sharply, which usually means an upstream API changed silently.

### 9. Git

```bash
git switch -c p02-api-extractor
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: paginated GitHub client`
- `feat: retry with backoff and jitter`
- `feat: raw landing partitioned by date`
- `feat: watermark persistence per page`
- `fix: resume skips already-fetched pages`
- `test: pagination and retry`
- `docs: CLAUDE.md and AI_NOTES`

**PR description should answer:** how resume works, and what happens if the process dies mid-page.

**Before pushing:** `git grep -i 'ghp_\|token.*=' -- ':!*.md'` must return nothing.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p02 && git push --tags
```

---

[← P01](p01-file-ingestor.md) · [Roadmap](../PROJECTS.md) · [P03 →](p03-batch-pipeline.md)
