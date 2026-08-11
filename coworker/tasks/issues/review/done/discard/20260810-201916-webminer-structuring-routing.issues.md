# Issues: webminer-structuring-routing

> **Source:** `20260810-201916-webminer-structuring-routing.full.md` | **Date:** 20260810-201916 | **Mode:** dev

## Scenario Background

### Task

**Partially Successful (75%)** — 4 of 5 acceptance criteria completed fully. The swarm high-throughput path (AC5) was demonstrated end-to-end but the backend workers couldn't process localhost URLs, which is itself a valuable finding.

### What Worked Well

| AC | Criterion | Status |
|----|-----------|--------|
| AC3 | Single-page HTML export (5 product pages) | ✅ |
| AC1 | WebMiner SMILE pipeline (10 files, k=68, silhouette=0.44) | ✅ |
| AC4 | Crawl bulk fetch (6/8 pages, depth 0) | ✅ |
| AC2 | Production-scale decision point documented | ✅ |
| AC5 | Swarm commands demonstrated (create/submit/status) | ⚠️ Backend blocked |

### Issues Found: 8

1. **Critical** — Swarm workers fail with `Protocol not found` for localhost URLs
2. **High** — `doctor log` command name is circularly broken (both forms reject)
3. **High** — Swarm worker pool starvation with default settings
4. **Medium** — Swarm `--wait` progress conflates succeeded/failed jobs
5. **Medium** — WebMiner views output path differs from documentation
6. **Medium** — Crawl occasionally reports 0-byte fetch for valid pages
7. **Low** — `webminer.ps1` not discoverable from repo root
8. **Low** — WebMiner `--max-files` default of 40 may surprise users

### Deliverable Files

- **JSON findings:** `.test-sessions/evaluation-findings-structuring-extracted-pages.json`
- **Markdown report:** `.test-sessions/evaluation-findings-structuring-extracted-pages.md`
- **AC2 decision doc:** `.test-sessions/production-scale-decision.md`
- **HTML corpus:** `.test-sessions/html-corpus/` (10 files)
- **WebMiner output:** `.test-sessions/html-corpus-ml-output/` + `/tmp/pulsar-vincent/ml/tasks/unsupervised/result/`

### Overall Rating: **6/10**

The core acquisition paths (goto → htmlsnapshot → export, crawl --seed-file) are solid and well-documented. The SKILL.md §4d decision tree is excellent. The swarm path has critical reliability gaps that block the high-throughput story for local testing, and several CLI inconsistencies (doctor log, swarm status reporting) add friction for first-time users.

---

## Issues Found (0)

No issues could be parsed from Section C of the agent output.

See `20260810-201916-webminer-structuring-routing.full.md` for the complete evaluation output.

