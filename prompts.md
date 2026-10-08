# How I build: prompts

*By [Saibal Roy](https://www.saibalroy.com/) · [LinkedIn](https://www.linkedin.com/in/roysaibal) · [GitHub](https://github.com/saibal-roy)*

I build open source from **real production requirements**: a problem a business actually has, with a budget, a deadline, confidential data and a team that has to keep it running. The aim isn't a demo; it's a **reusable business solution** that someone else can deploy, trust and afford. That means designing the whole system, not just the code:

- **Cost first.** Start from the unit economics (cost per page, per document, per server hour) and size the solution down to the smallest machine that measurably works.
- **Reliability you can prove.** Every requirement gets an acceptance check, and nothing ships until a go-ahead gate rehearses a fresh server on the supported platform and target size.
- **Evidence before decisions.** Benchmarks, not opinions. When there's a trade-off, the numbers go to the owner, and the decision and its evidence are written down.
- **Secure and confidential by default.** Services bound to localhost, no stored credentials, client data kept out of the repository by mechanism rather than memory.
- **Built for the people who maintain it.** Pinned versions, one supported platform, scripts for setup, demo and cleanup, upgrades only through the gate.
- **Grow in stages.** One small server first; the next stage is written up as a proposal with the triggers that would justify it.

These are the prompts I use with Claude to work this way. The first set rebuilds **[docling-batch-extract](https://github.com/saibal-roy/docling-batch-extract)** (batch PDF → Markdown for RAG on one CPU-only 2 vCPU / 8 GB server; [documentation](https://saibal-roy.github.io/docling-batch-extract/), [how it was built](https://saibal-roy.github.io/docling-batch-extract/STORY/)) with every validated decision built in. The second is the template I start any similar solution with.

> The maintained original lives in the project: [`docling-batch-extract/prompts.md`](https://github.com/saibal-roy/docling-batch-extract/blob/main/prompts.md), which also has prompts for operating, maintaining, investigating and extending it. This page is a copy of its build and template sections as of release 0.1.0.

**How to use them:** run the build prompts in order in an empty folder and review each result before moving on. Fill in anything in `<angle brackets>`. Never paste confidential documents into a prompt: refer to them by path and keep them in git-ignored folders.

**Validated baseline these prompts describe** (go-ahead gate passed 2026-10-08):

| Area | Validated choice |
|------|------------------|
| Target machine | **2 vCPU / 8 GB RAM**, CPU only, no GPU |
| OS | **Ubuntu 26.04 LTS only**: the latest LTS; setup refuses others without `--force` |
| Engine | `docling-serve-cpu` **v1.36.0** (pinned), **1 worker × 2 threads**, ~6.5 GB limit, sized through `.env` |
| Security | API published on **127.0.0.1 only** (check A25); never open port 5001; no stored cloud keys |
| Dependencies | Latest LTS where one exists, otherwise the latest stable release, **always pinned**; upgraded only through the gate |
| Release | Semantic versioning from 0.1.0; `extractor_version` + `engine` in every JSON |
| Gate | `tests/ubuntu_container_test.sh`: Ubuntu 26.04 on a simulated 2 vCPU / 8 GB server, 18 acceptance checks |

---

## Build docling-batch-extract from scratch

### 1. Context and plan

```text
I'm building a batch pipeline that converts PDFs (legal and claims-insurance documents, many scanned) into
Markdown for a RAG search index. It has to be a cost-effective alternative to cloud OCR APIs such
as AWS Textract or vision-LLM APIs. Target machine: 2 vCPU / 8 GB RAM, CPU only, no GPU, running
Ubuntu 26.04 LTS (only the latest Ubuntu LTS is supported). The pilot is about 500 documents
averaging about 47 pages. It's maintained part-time, so stability beats features: use the latest
LTS of everything that has one, otherwise the latest stable release, always pinned, and upgrade
only through the test gate.

Use docling-serve in Docker as the conversion engine: the CPU-only image
ghcr.io/docling-project/docling-serve-cpu pinned to an exact stable tag (never latest/main, never
a GPU variant). The container stays running between batches; the script must never start, stop
or restart it.

Before writing any code, write versions/plan-v1.md containing:
1. My requirements as a numbered list (E1, E2, …).
2. The design.
3. A table of acceptance criteria (A1, A2, …), each with a concrete check and a pass condition,
   mapped to the requirements.
4. A list of the defaults you chose that I might want to change.
Confirm the docling-serve API against http://localhost:5001/docs for the pinned version rather
than relying on memory. Whenever the plan changes, write a new plan-vN.md that starts with a
"Changes from vN-1" section (change → reason → evidence) and mark the old one superseded.
```

### 2. Container and sizing

```text
Write docker-compose.yml with a fixed project name (docling-batch-extract) for
docling-serve-cpu:${DOCLING_TAG:-v1.36.0}, container name docling_ocr_worker_cpu, restart: always,
UI disabled, local engine, 1 uvicorn worker. Publish the port on loopback only
("127.0.0.1:5001:5001"); the API has no authentication. Take sizing from .env with defaults for
the 2 vCPU / 8 GB target: DOCLING_SERVE_ENG_LOC_NUM_WORKERS=${DOCLING_WORKERS:-1}, threads
(DOCLING_NUM_THREADS / OMP_NUM_THREADS / MKL_NUM_THREADS)=${DOCLING_THREADS:-2}, memory limit
${DOCLING_MEMORY:-6500M}. Add a .env.example and git-ignore .env. Start it, wait for /health, and
report the image size, pull time and idle memory.
```

### 3. The extraction script

```text
Write extract.py: a single file using only the standard library plus requests and pypdf (pinned
to their latest stable versions in requirements.txt).

Folders and lifecycle
- Process inputs/*.pdf in sorted order. On success, write outputs/<stem>.json with source_file,
  status, pages, chunks, force_ocr, extractor_version, engine (docling_serve and docling versions
  from GET /version, pdf_backend), markdown, processing_time, errors and converted_at. Write it to
  a temp file and os.replace it, and only then move the PDF to completed/.
- A PDF that can't be converted (unreadable, docling error, retries used up) moves to errors/ and
  the run continues with the other files.
- If docling-serve itself goes down and doesn't come back, stop the run and leave the remaining
  PDFs in inputs/. A server outage must never be blamed on a document.
- inputs/ is the queue: running the script again processes only what is still there. The URL comes
  from --url or $DOCLING_URL (default http://localhost:5001).

Conversion
- Use the async API: POST /v1/convert/file/async with to_formats=md and page_range=[a, b], poll
  /v1/status/poll/{id}, then GET /v1/result/{id}. Treat success, partial_success, failure and
  skipped as finished states.
- Always send pdf_backend=pypdfium2. docling's default docling_parse used more than 6 GB on a single
  JBIG2-compressed scanned page and crashed the container.
- Detect scanned PDFs with pypdf: a page counts as scanned if it carries an image at least as wide
  as the page. If at least 50 % of pages are scanned, send force_ocr=true; scanners' embedded text
  layers are often garbled. Make this --ocr auto|force|default.
- Split text PDFs into 5-page ranges (--chunk-pages 5). Send scanned PDFs as one job
  (--scanned-chunk-pages 0); splitting scans raised peak memory.
- Queue the ranges in document order. --max-inflight defaults to the container's
  DOCLING_SERVE_ENG_LOC_NUM_WORKERS (read with docker inspect; fall back to 1), so client and server
  can't drift apart. A background thread reads container memory every 5 s, and new submissions
  pause above --mem-high 0.75; fall back to the in-flight limit if docker stats isn't available.

Robustness
- docling-serve stops answering HTTP while it converts. A poll timeout means "still busy, keep
  waiting", never "resubmit". Resubmit a range only if the container's RestartCount (docker inspect)
  changed or the task returns 404. Then wait for /health and retry, up to --retries 2.
- Allow --timeout 60 seconds per page for each job, with a minimum of 10 minutes.

Logging
- One log per PDF at logs/<stem>.log, replaced each time that PDF is processed, containing only
  that document's lines: DOC_START (pages, ranges, scanned %, force_ocr), SUBMIT, a PROGRESS
  heartbeat every 30 s, CHUNK_DONE, DOC_DONE (total seconds, pages, seconds per page) and
  DOC_FAILED / RETRY / ABORTED. The console shows the same lines tagged with the PDF name, plus
  RUN_START (incl. chunk sizes, version, docling version), THROTTLE and RUN_END. No cumulative log.

Exit codes: 0 all succeeded, 1 some PDFs failed, 2 server unreachable or down.

Versioning: semantic versioning from 0.1.0; __version__ in extract.py and --version. Define the
public contract (JSON schema, folders, options, exit codes, log lines) and the bump rules in
CHANGELOG.md (Keep a Changelog).
```

### 4. Tests, benchmarks and the go-ahead gate

```text
Create tests/ (pin reportlab in tests/requirements.txt):
- make_test_pdfs.py: text PDFs of 3, 5, 15 and 60 pages (headings, paragraphs, a table per page),
  a 20-page simulated scan (each page one full-page bitmap, no text layer), and a corrupt bad.pdf.
- measure_job.py: one job for a page range with any docling options; prints seconds and peak memory.
- run_acceptance.sh: every acceptance criterion as PASS/FAIL in an isolated tests/work/ directory
  (never the real folders): corrupt PDF, empty rerun, memory throttle, memory monitor unavailable,
  container restart mid-run, server down, one log per PDF, OCR only on scans, JSON schema incl.
  extractor_version/engine, and A25 "port published on 127.0.0.1 only". --bench-only runs
  per-PDF benchmarks after a warm-up job. Include PDFs from git-ignored tests/fixtures/.
- profile_benchmark.sh: recreate the container with --cpus/--workers/--threads/--memory/--tag,
  limit it with docker update --cpus, warm up, convert a fixed 95-page batch (text + scan), and
  record time, process memory (cgroup "anon", not docker stats, which counts reclaimable cache),
  memory-limit events and OOM kills. Restore the normal container afterwards.
- ubuntu_container_test.sh, the release gate: a fresh ubuntu:26.04 container pinned to 2 CPUs
  (--cpuset-cpus 0-1) against the host's Docker daemon (mounted socket,
  DOCLING_URL=host.docker.internal), as a non-root sudo user: setup_ubuntu.sh (must write the
  2 vCPU .env) → cap docling at 2 CPUs → checks → setup again → smoke test → cleanup.sh →
  demo_run.sh → run_acceptance.sh --quick. Pass the steps as an argument, give every step stdin
  from /dev/null, count a run as passed only if its log reaches a final marker line, and recreate
  the normal container afterwards.
- Everything must work with macOS's bash 3.2 and pass shellcheck -S warning.
Run them and fix anything that fails.
```

### 5. Documentation and open-source readiness

```text
Write the README for developers and operators: background, how it works, developer-machine and
Ubuntu 26.04 server setup, first demo run (with a real screenshot rendered from a gate run by
docs/render_demo_screenshot.py), operations, options, log format, server specification for the
validated 2 vCPU / 8 GB target, memory and throughput measured on that target, the pilot estimate
(pages × measured seconds per page) against cloud OCR pricing with sources and an "as of" date,
AWS EC2 deployment (m7i.large for batches; t3.large for small batches, with its burstable-credit
caveat; larger sizes marked "not validated"), production security on AWS (never open port 5001,
SSH from one IP or Session Manager, SSH tunnel for the API, IAM roles instead of keys, encrypted
EBS), the maintenance policy table of pinned versions, versioning and releases, publishing on
GitHub Pages, and test results. Use only measured numbers, always with the hardware they came from.
Update CLAUDE.md. Add an MIT LICENSE (copyright <name> (<github-url>)), a CHANGELOG.md, and
.gitignore rules: *.pdf (except licensed demo files), .env, .venv, tests/work, site output,
.claude/, plus a .gitignore inside each data folder (inputs, outputs, completed, errors, logs,
tests/fixtures) that keeps the folder but ignores its contents. Credit the author with links to
<website>, <LinkedIn> and <GitHub>. Finally, search the repository for client file names and
quoted document text, and remove them.
```

### 6. Continuous integration and releases

```text
Add .github/workflows/ci.yml (push to main, pull requests, manual, and workflow_call):
- Job 1 on ubuntu-26.04: fail if any PDF (other than licensed demo files listed in
  scripts/demo-files/README.md) or data-folder content is tracked; ruff with the committed
  ruff.toml (pinned ruff version); py_compile; shellcheck -S warning; docker compose config.
- Job 2 on ubuntu-26.04 only: free disk space; provision with scripts/setup_ubuntu.sh and check
  container, .env, venv, folders and cron; run setup again (one cron entry); remove the cron entry;
  smoke test (exit code 1 for a corrupt PDF); cleanup.sh; demo_run.sh; run_acceptance.sh --quick
  (--bench-only on manual runs with "benchmarks"). Put this run's summary in the job summary,
  upload it as an artifact, dump container logs on failure.
Add release.yml: tag vX.Y.Z → verify tag = __version__ = a dated CHANGELOG section → reuse ci.yml →
create the GitHub Release from that changelog section (relative links made absolute). Use the
latest major versions of actions/checkout, setup-python, upload-artifact, upload-pages-artifact
and deploy-pages. Validate with actionlint (declare ubuntu-26.04 in .github/actionlint.yaml if
your actionlint predates it).
```

---

## The reusable template: start any solution like this

Use this as the **first prompt** for a new open-source tool built with Claude, for a client or not. It builds in the working method that made this project fast and safe: requirement IDs, versioned plans, evidence-based decisions, real-data testing early, sizing for cost on measured numbers, client-data protection, security by default, and CI from day one.

```text
You are acting as the engineering manager and lead engineer for a new open-source tool.

Context
- Problem: <what is slow, expensive or manual today, and for whom>
- Business constraint: <budget / cost per unit, data confidentiality, deadline>
- Target machine and platform: <e.g. 2 vCPU / 8 GB, CPU only, latest Ubuntu LTS only>
- Scale for the first real run: <e.g. N documents × M pages>
- Alternatives being replaced and their cost: <e.g. a cloud API at $X per unit>
- Engine or library to build on: <e.g. docling-serve-cpu, exact stable tag>
- Maintenance reality: <e.g. part-time maintainer → stability over features>
- License and owner: <MIT, name, website / LinkedIn / GitHub>

Working method (follow strictly)
1. Before any code, write versions/plan-v1.md: my requirements as E1…En, a design, acceptance
   criteria A1…An (concrete check + pass condition, mapped to requirements), and the defaults you
   chose. Ask me only questions whose answers change the design; when a choice is a trade-off,
   show me the measured numbers and let me decide.
2. Verify every external API, price and product claim against the real service or its current
   docs; never rely on memory. Cite prices with an "as of" date.
3. Within the first hour, run the design on ONE real input I provide (by path, never pasted).
   Measure time and real process memory per unit of work, and read the actual output.
4. Each time a requirement is added or evidence contradicts the plan, write plan-vN+1.md starting
   with "Changes from vN" (change → reason → evidence) and mark the old one superseded.
5. Build: minimal, pinned dependencies (latest LTS / stable); one obvious entry point; no input
   ever stops the batch; per-item logs; idempotent reruns; "busy" is not "down"; services bound to
   localhost unless they must be public; no stored credentials.
6. Tests: synthetic inputs with the real input's hard properties (no client data in tests), a
   PASS/FAIL acceptance script in an isolated directory, a profile benchmark for the target
   machine with repeated runs, and a go-ahead gate that rehearses a fresh server on the supported
   OS and target size. A gate passes only on an explicit final marker. Publish failed checks.
7. Client data: per-folder .gitignore files, a CI data guard, and a repository search for client
   identifiers before any release.
8. Ship: README (background, how it works, setup, operations, spec and capacity from measured
   numbers, cost comparison, security, maintenance policy, versioning, troubleshooting, test
   results), LICENSE, CHANGELOG (semantic versioning with a defined public contract), CLAUDE.md,
   setup / demo / cleanup scripts, CI + release workflows, a documentation site checked locally
   before publishing, and a prompts.md that rebuilds the validated design.
9. Keep STORY.md: a timeline with real times, what was caught and how, the system-thinking
   principles, and results tables that say "pending" until real numbers exist.
10. Before anything goes public, run a pre-publish review: what exactly is published, privacy
    and secrets, credibility of every claim, tone, first impressions, client confidentiality.
11. Don't commit, push or publish unless I ask.
Start with step 1.
```
