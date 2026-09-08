---
name: technical-writer
description: Writes technical runbooks and lab guides in Vietnamese that record the steps to set up a system, configure a service, reproduce a failure, or debug and resolve an incident. Use when the user asks to "viết runbook", "viết lab", "document lại các bước setup", "ghi lại quá trình debug", "tái hiện lỗi", "viết hướng dẫn cài đặt", or asks to turn a terminal session, chat history, or set of config files into a document someone else can follow end to end. Produces a fixed structure: goal and context, prerequisites, resource planning table, mermaid diagram, executable steps with verification, rollback, and references.
---

# Technical Writer

Write technical documents that **someone else can execute without asking the author anything**.
That is the only quality bar. Every rule below serves it.

> [!IMPORTANT]
> **Output language is Vietnamese.** All instructions in this skill and its reference files are in
> English, but the runbook you produce is written in Vietnamese. Keep technical terms in English —
> do not translate them (persistent disk, golden image, service account, pipeline, runner, rollback...).
> Command names, parameter names, file paths, and log excerpts always stay verbatim.

## Pick the document type

| Situation                                                                       | Template                             |
| ------------------------------------------------------------------------------- | ------------------------------------ |
| Installing/configuring a new system, building a lab, integrating two systems     | `assets/runbook-template.md`         |
| Investigating and fixing an incident, reproducing a bug, root cause analysis     | `assets/troubleshooting-template.md` |

If the request is ambiguous ("viết lại cái vụ hôm qua"), ask one question to settle it — do not guess.
Never merge the two templates into one file. If a task involves both a setup and an incident, write the
setup runbook and link out to a separate troubleshooting runbook.

## Workflow

### Step 1 — Gather before writing

Read every available source first: conversation history, config files, terminal output, logs, source code
in the repo, screenshots the user provided. Most of what you need is usually already there.

Then check against the required list:

- **Goal**: what state the reader reaches, and how they measure that they got there
- **Versions**: OS, and the version of every major software/package/module involved
- **Resources**: IPs, hostnames, domains, ports, datastores, service account names
- **The actual commands that were run**, and their output
- **Config file paths** and their contents
- **Failure-prone spots**: dangerous flags, non-idempotent steps, irreversible operations
- (Troubleshooting) **symptoms, raw logs, reproduction steps, root cause, fix, prevention**

When something is missing, resolve it in this order:

1. Search the repo, logs, and conversation for it.
2. Not found, and **the document is wrong without it** (version, root cause, step ordering) → batch the
   gaps and ask the user **once**, at most 5 questions.
3. Not found, but it is only an environment value (IP, domain, password) → use a `<placeholder>` and
   record it in the Planning table.

> [!WARNING]
> **Never invent.** Do not fabricate IPs, version numbers, module names, command output, or file paths.
> One wrong command in a runbook costs the reader more time than no command at all.
> Mark anything uncertain explicitly: `> [!TODO] Cần xác nhận: ...`

### Step 2 — Outline first

List the main sections (`##`) and steps (`###`) before writing any prose. Check:

- Are steps in true dependency order (install the DB before the app that uses it)?
- Does any step bundle several unrelated actions and need splitting?
- Is any section background knowledge rather than an action → cut it or move it to Reference.

### Step 3 — Write against the template

Open the matching template and follow its section order exactly. Do not invent new sections unless the
content genuinely fits nowhere; do not drop required ones — if a section truly does not apply, write
"Không áp dụng - <lý do>" in one line.

For layout rules, mermaid syntax, callout usage, and how to write commands, read `references/writing-guide.md`.

### Step 4 — Self-check

Run the checklist at the end of `references/writing-guide.md` before returning the document. Fix everything
it flags first, then hand it over.

## Hard rules

Violating any of these means rewriting, not annotating:

1. **Vietnamese prose, English technical terms.** See the callout at the top.
2. **One action per step**, phrased as an imperative ("Tải file binary", "Thêm dòng sau vào..."), never
   as narration ("Chúng ta sẽ tiến hành...").
3. **Every code block carries a language tag** (`bash`, `yaml`, `ini`, `sql`, `json`, `text`...). Do not
   label non-shell content as `shell` — YAML config is `yaml`, INI is `ini`, SQL is `sql`, command output
   and logs are `text`.
4. **The file path sits immediately above its code block**, in backticks on its own line. The reader must
   know where to paste the content.
5. **Every major section ends with a way to verify it** — a check command with its expected result, or a
   description of the correct observable state. Without it the reader cannot tell whether they succeeded.
6. **No real secrets in the document.** Passwords, tokens, API keys, and private keys are placeholders or
   references to a secret store (`${{ secrets.X }}`, vault, CI variable). If the input contained a real
   secret, replace it and tell the user in one line when returning the document.
7. **Warnings sit next to the dangerous step**, never collected at the end. Use `> [!WARNING]` for data
   loss or downtime, `> [!NOTE]` for context, `> [!TIP]` for a better alternative.
8. **Explain the why behind every non-obvious value.** `destroy: false`, `no_log: false`,
   `password_encryption = scram-sha-256` — each needs one sentence of reasoning. This is the most valuable
   part of a runbook and the most frequently omitted.
9. **A diagram is mandatory**, near the top, as a mermaid `flowchart TD`. Drawing rules are in the writing guide.
10. **Never write "tương tự như trên"** or "làm giống bước 3". Repeat the full command. Runbook readers are
    usually in a hurry and skip around.
