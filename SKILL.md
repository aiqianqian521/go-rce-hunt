---
name: go-rce-hunt
description: Hunt remote code execution in Go/golang source trees. Drive a six-path RCE model (shell concatenation, template evaluation, deserialization, write-then-execute, argument injection, dynamic loading) over a real checkout, run scripts/scan_sinks.py to get ranked candidates, trace source-to-sink reachability through Go idioms (interface dispatch, middleware ordering, handler binding, goroutines, callback registries), and grade every finding L0-L3 before writing anything down. Use for Go 代码审计, 挖 RCE, 命令注入, SSTI, gob/cookie 反序列化, git hook 写入, 参数注入, k8s/容器组件漏洞, CVE 复现校验, or building a Go RCE scanner. Never treats a sink hit, a grep match, a CVE description or a model's "exploitable" claim as a vulnerability.
---

# Go RCE Hunt

Find RCE in Go code without drowning in false positives. The scanner produces
candidates; **you** produce verdicts, and only after a named evidence gate.

## 0. Non-negotiables

- No source, no verdict. Without a readable checkout you may deliver a method
  and a checklist, never a finding. Say `BLOCKED: no source` and stop.
- A sink hit is `L0 SUSPECT` by definition. Calling it a vulnerability before
  `L2 INJECTABLE` is the single most common failure of automated Go auditing.
- Never cite a CVE number, affected version or CVSS score you have not opened.
  PoC repos, vendor advisories and the patch diff beat blog summaries. Content
  farms invent CVE-2024-34567-style numbers that do not exist; see
  [case library](references/case-library.md) for the named fakes.
- Authorization first for anything that executes. Running a payload against a
  third-party instance is out of scope unless the user owns it.

## 1. The six paths (why Go RCE exists at all)

Go is memory-safe. There is no deserialization gadget chain, no `__reduce__`,
no JNDI. RCE therefore always has one of six shapes, and each has a *different*
falsification test:

| # | Path | What makes it true | Typical sink | Falsifier |
|---|------|--------------------|--------------|-----------|
| E1 | **Shell concatenation** | a shell binary parses the string | `exec.Command("sh","-c", …)` | args passed separately → false |
| E2 | **Template evaluation** | template text is attacker-controlled, **or** FuncMap exposes an exec/file function | `template.Parse(userStr)`, `.Funcs(...)` | literal template + safe FuncMap → false |
| E3 | **Deserialization** | decoded data is later used as a command/path/permission | `gob`/`yaml`/cookie decode | decode alone never executes → false |
| E4 | **Write-then-execute** | a write lands where something later executes it | write into `hooks/`, cron, `authorized_keys`, plugin dir, webroot | no downstream executor → false |
| E5 | **Argument injection** | a controllable argv token starts with `-` | `exec.Command("git", userArg)` | literal prefix (`"/reports/"+n`) → false |
| E6 | **Dynamic loading** | the load target is controllable | `plugin.Open(v)`, `syscall.Exec`, DB `LOAD` | fixed literal path → false |

Two facts follow, and they invert the usual grep habit:

1. **`exec.Command` is not E1.** Only a shell turns a string into a command
   line. `exec.Command("cat", "/reports/"+name)` is a path-traversal bug, not
   command injection. If you report it as RCE, you are wrong and a reviewer will
   find out in one minute.
2. **Most real Go RCE contains no `exec` at all.** Gitea's 2026 in-the-wild RCE
   wrote a git hook. Grafana's needed a DuckDB extension. Both are E4-shaped:
   a *write* or a *plugin*, not a command string.

Read [six paths](references/six-paths.md) for per-path mechanics, payload shapes
and the exact conditions that make each falsifiable.

## 2. Five counter-intuitive conclusions

Print these before touching a codebase. Each one is a false-positive class you
will otherwise spend a day on.

1. **`grep exec.Command` has a terrible signal ratio.** In a typical Go service
   most hits are `exec.Command("git", …)` with constant args. Triage by argv
   construction, not by call presence.
2. **`gob`/session deserialization in Go usually does not reach RCE.** Go has no
   object graph that calls `exec` on decode. Gogs/Gitea CVE-2018-18925 is a
   *privilege escalation* (forged session identity) that then abuses a git-hook
   feature. Tracing decode → exec directly is a category error. Correct chain:
   decode → identity/permission → E4.
3. **Go templates are safe by default; the door is `FuncMap`.** `html/template`
   escapes, and `text/template` cannot call arbitrary functions. Audit
   `.Funcs(...)` registrations harder than `.Parse(...)` calls. A `FuncMap` with
   a `run`/`exec`/`readfile` key is E2-critical on its own.
4. **`filepath.Join` is not a sanitizer.** It resolves `..`. In Go 1.24+
   `os.Root` is the real fix. Anything else needs `Clean` + prefix check.
5. **A CVE description is a press release; the patch diff is the truth.** Read
   the fix commit before claiming a root cause. Grafana's patch *deleted* the
   SQL Expressions feature — the "SQL injection" label hides an extension-load
   path that is closer to E6.

## 3. Workflow

### Phase 0 — Scope (2 min, blocks everything else)
Record: path, commit SHA (`git -C <dir> rev-parse HEAD`), Go version from
`go.mod`, whether `vendor/` exists, entry surface list (HTTP/gRPC/CLI/cron/
watcher), and the deployment user. **Without the deployment user you cannot
state impact** — `git` user in a container is not root on the host.

### Phase 1 — Candidate generation (mechanical)
```bash
python3 scripts/scan_sinks.py <path> --out scans/<name>.md --json scans/<name>.json
# narrowed passes once you know the shape of the project:
python3 scripts/scan_sinks.py <path> --paths E4 --min-severity high
```
Read section A first (sink + source in the same func). Section B needs Phase 2.
The scanner skips `vendor/` and `_test.go` and says so in the report — that
report is therefore **not** a negative result for those trees. If the project
vendors its own fork, rerun with `--include-vendor`.

Do not trust the scanner's silence. Its rules are regexes over a language with
generics, build tags and generated code. Run at least one non-scanner pass:
```bash
rg -n 'sh", "-c|/c", |CommandContext' --type go
rg -n 'hooks/|hooks\\\\|\.git/hooks|authorized_keys|crontab' --type go
rg -n 'Funcs\(|ParseFiles|plugin\.Open|syscall\.Exec' --type go
```

### Phase 2 — Reachability (the actual work)

Before doing this per-candidate, if you have more than ~30 candidates, do the
ordering pass in [route map](references/route-map.md) §2–§3: it decides which
candidates are worth a reachability trace at all. Tracing all of them is how
a 2-hour review becomes a 2-day one with no better result.

**Trace backwards, not forwards.** Sinks are a bounded set (the scanner gave you
all of them); entries are not. Forward taint analysis forces you to enumerate
every route and middleware to be sound, and Go's interface dispatch makes it
worse. Reverse slice instead: from the sink line, grep for callers, and **stop at
the first handler boundary** (`gin.Context`, `http.ResponseWriter`,
`echo.Context`, `macaron.Context`). Go has no dynamic dispatch, so a 2–5 hop
chain is enumerable by hand in minutes.

Write the hypothesis as three slots before tracing — `<entry controllability> +
<sink shape> + <nature of the transforms in between>` — then go falsify it. If it
dies, form a new hypothesis; do not fall back to a full scan. A hypothesis
missing any of the three slots means you have not decided what you are looking
for yet. Worked examples: `prompts/architect-intake.md`.

Per candidate, answer three questions in writing:

1. **Entry**: which handler/CLI/cron reaches this func? Name the route and the
   middleware chain. Auth state matters more than the sink.
2. **Hop count**: how many funcs between request data and the sink? In Go this
   is usually 2–5, because there is no magic. If you need 8 hops with interface
   dispatch, you probably have a false positive.
3. **Shape of the data at the sink**: literal / validated enum / length-limited
   / raw. Quote the line that does the validation, or state `none`.

Go-specific traps that break naive taint tracking — see
[reachability](references/reachability.md):
- interface method dispatch hides the callee (`Executor` with 4 impls)
- middleware `c.Next()` reordering, and handlers registered on a group
- `go func(){...}()` captures variables you did not trace
- callback registries (`map[string]func`) resolved by user-supplied key
- reflection on struct tags; generated `*_gen.go` / `*.pb.go`
- build tags: the file you read may not be the file compiled

### Phase 3 — Injectability (proof, not plausibility)
Write the payload *and* the reason it survives every transform on the path.
Record the exact characters that must survive: space, `;`, `|`, `{{`, `/`, `..`,
leading `-`. If a sanitizer strips one, the finding is dead — say so.

Class-specific minimum proof:
- E1: the shell line, and that the injected segment is not `shellescape.Quote`d
- E2: which `FuncMap` key or method is reachable from `{{ }}`, and that the
  template text (not just data) is attacker-controlled
- E3: what the decoded value is *used for* afterwards (this is the finding)
- E4: the write path, permissions, and the **downstream executor** plus its
  trigger (next `git` call, next cron tick, next plugin load). No executor = no RCE.
- E5: the option that upgrades to execution (`--upload-pack=`, `-exec`, `--filter`)
- E6: how the artifact gets on disk

### Phase 4 — Local reproduction (optional, authorized, isolated)
Only in a container/VM with no network egress, on a build you own. Freeze the
commit. Capture: build cmd, payload, observed side effect, and the *negative*
control (same request with the payload neutralized must not fire). A finding
without a negative control is not reproduced, it is a coincidence.

### Phase 5 — Report
Follow the [report contract](references/report-template.md): pick short form or
the 8-section form, then run its pre-delivery checklist (counts must sum, every
`path:line` read back, `L3-repro` = number of `EV-###` records). Every claim
carries an evidence level. Rerun the scanner after any code change; the candidate
set is not stable across commits.

## 4. Evidence levels (the gate)

| Level | Meaning | Allowed wording |
|-------|---------|-----------------|
| L0 | sink hit only | "候选 / candidate, unverified" |
| L1 | source→sink path traced, entry named | "疑似可达 / suspected reachable" |
| L2 | injectability proven, payload characters survive | "疑似漏洞 / likely vulnerable" |
| L3 | reproduced locally with negative control | "已复现 / confirmed" |

Rules:
- Publishing `L0`/`L1` as a vulnerability is a **process failure**, not a
  stylistic choice. Downgrade it in the report and say why.
- An LLM's own "this is exploitable" is not evidence. Neither is a scanner
  score, a CVE match from `govulncheck`, or a PoC repo existing.
- Deleting a finding is a decision that needs a reason recorded too
  (`FALSE-POSITIVE: sanitized at store.go:88`), because a silently dropped
  finding is indistinguishable from a missed one.

Full gate checklist: [verification gates](references/verification-gates.md).

## 5. Output contract

Full contract: **[report contract](references/report-template.md)** — two sizes
(short form for a single question, 8-section form for a project audit), ID
conventions, coverage matrix, and the pre-delivery checklist.

Four non-negotiables, whatever the size:

1. **Counts must sum.** candidates = `GO` + `HOLD-evidence` + `HOLD-capability`
   + `KILL` + `unjudged`. A gap in the arithmetic is a missed candidate.
2. **Every `path:line` is read back individually before it goes in the report.**
3. **`L3-repro` and `L3-lit` never mix.** The reproduce count only comes from
   `EV-###` records with a negative control.
4. **Every dynamic action states who authorized it.** No authorization, no L3 —
   log it as a coverage gap instead.

Minimum shape (short form):

```markdown
**结论**：<one line, with evidence level>
**目标**：<repo> @ <commit or "tarball, commit unknown">，扫描 <N> 个 .go 文件
**候选**：C-001 `<rule_id>` `path:line` — <verdict + the gate that decides it>
**覆盖边界**：<skipped trees, unjudged tiers, unchecked gates>
```

Keep rejected candidates in the report. A review that only lists confirmed bugs
cannot be audited, and the user cannot tell "checked and clean" from "did not
look".

## 6. References

Load order matters. Do **not** read all of these — run
`python3 scripts/kb_retrieve.py --project <path>` to get the 3–5 chunks that
actually apply (see [RAG index](references/rag-index.md); the full corpus is
~20k tokens across 114 chunks, a run should touch 2.5–3.5k).

- [execution flow](references/execution-flow.md) — six mermaid diagrams: OODA × Phase 0–5, the phase decision tree, the nine gates on the L0→L3 chain, tier ranking, retrieval wake-up, write-back loop. **Open this first to see the whole shape at once**
- [OODA workflow](references/ooda-workflow.md) — per-step completion criteria, three task types with different success standards, loop failure modes
- [RAG index](references/rag-index.md) — chunk definitions, two retrieval channels, four-layer loading budget
- [route map](references/route-map.md) — **read this before Phase 1 when the project is bigger than ~200 Go files**: framework fingerprint → route file → which phase to check first → 35-minute standard route → gate checklist
- [six paths](references/six-paths.md) — per-path mechanics and falsifiers
- [sink catalog](references/sink-catalog.md) — rule-by-rule: what it matches, when it is real, when it is noise
- [case library](references/case-library.md) — public Go RCE cases with evidence level, plus the known-fake CVE list
- [reachability](references/reachability.md) — Go idioms that break taint tracking
- [verification gates](references/verification-gates.md) — L0→L3 checklist, negative controls, report redaction
- [report contract](references/report-template.md) — **required before Phase 5**: two report sizes, ID conventions, coverage matrix, pre-delivery count check
- [prior art](references/prior-art.md) — comparable open-source projects and the four design rules worth stealing
- `prompts/architect-intake.md` — **paste-ready intake prompts**: directed hunt / capability calibration / CVE reproduction, plus the reverse-hypothesis notation
- `scripts/scan_sinks.py` — candidate scanner (stdlib only)
- `scripts/kb_retrieve.py` — reference retriever: fingerprint channel + BM25, returns `file:line` ranges you can feed straight to `Read`
- `scripts/test_scan_sinks.py` — falsifiable self-test (positive, negative, mutation)
- `assets/selftest/` — the samples those assertions run against

## 7. Maintaining this skill

Adding a rule? Three edits, in this order:
1. append to `RULES` in `scripts/scan_sinks.py` (stable `id`, choose `kind`)
2. add a sample under `assets/selftest/` and wire it into `test_scan_sinks.py`
   — a **positive** sample that must hit and, if possible, a **negative** near-miss
   that must not
3. run `python3 scripts/test_scan_sinks.py`; the mutation test will fail if your
   new assertion is vacuous
4. **regression on real code**: run the rule over a real tree (the 10-project
   corpus is ~760k LOC) and check the Tier A/B count did not jump. Synthetic
   samples pass while real code drowns in noise — two shipped rules were only
   caught this way: `E2-TPL-USERSTRING` matched `time.Parse`, `E4-ZIPSLIP`
   matched every `driver.Extract` method. Both looked fine on hand-written
   samples and were 82% noise on real projects.

A rule shipped without a failing-on-purpose assertion is a rule nobody can
maintain. Same standard applies to the case library: an entry without a
`primary source` line does not belong in it.
