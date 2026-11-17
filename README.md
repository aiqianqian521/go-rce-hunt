# go-rce-hunt

**Not another grep for `exec.Command` — it is the part that turns a sink hit into a verdict you can defend in a report.**

A Go RCE hunting skill for LLM agents. The scanner produces ranked candidates; the skill
produces verdicts, and only after a named evidence gate. A sink hit, a grep match, a CVE
description, or a model's "exploitable" claim is never treated as a vulnerability.

## What it does

- **Six-path RCE model.** Go is memory-safe, so RCE only takes six shapes, and each one has
  its own falsifier:

| # | Path | What makes it true | Typical sink | Falsifier |
|---|------|--------------------|--------------|-----------|
| E1 | **Shell concatenation** | a shell binary parses the string | `exec.Command("sh","-c", …)` | args passed separately → false |
| E2 | **Template evaluation** | template text is attacker-controlled, **or** FuncMap exposes an exec/file function | `template.Parse(userStr)`, `.Funcs(...)` | literal template + safe FuncMap → false |
| E3 | **Deserialization** | decoded data is later used as a command/path/permission | `gob`/`yaml`/cookie decode | decode alone never executes → false |
| E4 | **Write-then-execute** | a write lands where something later executes it | write into `hooks/`, cron, `authorized_keys`, plugin dir, webroot | no downstream executor → false |
| E5 | **Argument injection** | a controllable argv token starts with `-` | `exec.Command("git", userArg)` | literal prefix (`"/reports/"+n`) → false |
| E6 | **Dynamic loading** | the load target is controllable | `plugin.Open(v)`, `syscall.Exec`, DB `LOAD` | fixed literal path → false |

- **Evidence gate L0 → L3.** Every claim carries a level, and the allowed wording is bound to it:

| Level | Meaning | Allowed wording |
|-------|---------|-----------------|
| L0 | sink hit only | candidate, unverified |
| L1 | source→sink path traced, entry named | suspected reachable |
| L2 | injectability proven, payload characters survive | likely vulnerable |
| L3 | reproduced locally with negative control | confirmed |

- **Phase workflow.** Scope → candidate generation → reachability → injectability →
  (authorized, isolated) local reproduction → report. Phase 0 alone records path, commit SHA,
  Go version, `vendor/` presence, entry surface and the deployment user — without the
  deployment user you cannot state impact.
- **Output contract.** Counts must sum (`candidates = GO + HOLD-evidence + HOLD-capability + KILL + unjudged`),
  every `path:line` is read back individually before it enters the report, `L3-repro` never mixes
  with `L3-lit`, and every dynamic action states who authorized it. Rejected candidates stay in
  the report, with the reason recorded — a silently dropped finding is indistinguishable from a missed one.
- **Retrieval-aware references.** `kb_retrieve.py` returns the 3–5 chunks that actually apply
  instead of loading the whole corpus (~20k tokens across 114 chunks; a run should touch 2.5–3.5k).

## Quick start

```bash
python scripts/scan_sinks.py <checkout>      # ranked candidates
python scripts/test_scan_sinks.py             # rule self-test (mutation testing)
python scripts/kb_retrieve.py --list          # reference chunks
```

Pure standard library, Python 3.9+, no third-party dependency.

The rule assertions are written so that they **fail if the corresponding rule is broken or
removed**: positive samples must produce the named rule id, negative samples must produce no
E1..E6 hit at all, and mutation tests prove each rule is genuinely required.

## Layout

```
SKILL.md                 # the skill body: non-negotiables, six paths, workflow, gates, contract
scripts/scan_sinks.py    # candidate generator (ranked rules, no LLM needed)
scripts/test_scan_sinks.py# falsifiable assertions + mutation tests
scripts/kb_retrieve.py   # reference retrieval over references/
references/              # six paths, sink catalog, case library, gates, report template, RAG index
prompts/                 # intake prompt
assets/selftest/         # positive and negative Go samples used by the assertions
```

## Related Projects

`go-rce-hunt` is a domain skill in the `aiqianqian521` agent-skill family. Its sibling
`go-sec-audit` covers Go security audit more broadly; the other repositories supply the
execution discipline, the report structure and the orchestration that drive them.

| Repository | What it is | Relation to `go-rce-hunt` |
| --- | --- | --- |
| [aiqianqian521/go-sec-audit](https://github.com/aiqianqian521/go-sec-audit) | Go source-code security audit skill (27 sink rules + 29 keyword channels, L0-L3 verdicts) | Sibling domain skill: wider surface, same evidence-gate discipline; use it when the question is not only RCE |
| [aiqianqian521/github-project-evaluation](https://github.com/aiqianqian521/github-project-evaluation) | Cross-source project discovery, frozen-repo static first pass, per-project test design | Its first pass can call this skill as the Go RCE dimension |
| [aiqianqian521/ai-skill-output-template](https://github.com/aiqianqian521/ai-skill-output-template) | Declarative output-template registry | Same contract idea: stable, verifiable, regression-checked report structure |
| [aiqianqian521/ooda_skill](https://github.com/aiqianqian521/ooda_skill) | OODA execution discipline (purpose, evidence, verify, close) | The round loop that drives a hunt; verdicts feed its per-round records |
| [aiqianqian521/sloth-loop](https://github.com/aiqianqian521/sloth-loop) | Zero-LLM task orchestration engine | Orchestrates repeated hunts; this skill is one domain capability inside a run |

## Scope

Only for **authorized** code audit and security research. Running a payload against a
third-party instance is out of scope unless the user owns it; authorization must be stated
for any dynamic action, and without it there is no L3 — log it as a coverage gap instead.

## License

MIT, see [LICENSE](LICENSE).
