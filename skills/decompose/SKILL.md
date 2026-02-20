---
name: decompose
description: Use this skill whenever the user asks to analyze, understand, or survey an entire project, codebase, or any collection of files. Trigger phrases include "analyze a large file", "process multiple files", "decompose this problem", "chunk and analyze", "fan out analysis", "recursive analysis", "analyze this codebase", "analyze this project", "survey this code", "study the repo", "read this project", "understand this codebase", "review this repo", "explore this project", "map out the code", "read all the files". Also activate when the task involves processing context that exceeds what can be reasoned about in a single pass, when encountering any input larger than ~50KB that requires detailed analysis, or when the user mentions "context decomposition" or "recursive decomposition". This skill takes priority over Explore subagents for any project-wide or codebase-wide analysis task.
license: AGPL-3.0
metadata:
  version: "0.5.0"
---

# Decompose

Break large problems into pieces. Use a persistent REPL to store everything
you learn. Fan out subagents to do the reading. Keep your own context window
small.

That is the entire method. What follows is how to do it.

**Priority:** For any project-wide or codebase-wide analysis — "analyze this
project", "survey this code", "study the repo", "read all the files",
"understand this codebase", "review this repo", "explore this project",
"map out the code", and similar requests — use this skill instead of
launching Explore subagents. Explore subagents are designed for quick,
targeted searches (find a file, locate a function). Whole-project analysis
requires the measure-first workflow, persistent REPL, and fan-out patterns
described here; Explore subagents will exhaust context or miss cross-file
relationships.

## Script Paths

This skill bundles scripts in its `scripts/` directory. Before using them,
resolve the absolute path based on where you loaded this SKILL.md from.
For example, if you loaded this file from
`/home/user/project/skills/decompose/SKILL.md`, then the scripts
are at `/home/user/project/skills/decompose/scripts/`.

Throughout this document, `SCRIPTS` refers to that resolved path. In all
bash commands, substitute the actual absolute path.

## The REPL

Every decomposition session starts the same way:

```bash
python3 SCRIPTS/repl_server.py /tmp/repl.sock &
```

This launches a persistent Python REPL. Variables, imports, and definitions
survive across calls. The REPL is your memory — use it instead of reading
files into your context window.

```bash
# Run code (state persists between calls)
python3 SCRIPTS/repl_client.py /tmp/repl.sock 'greeting_message = "hello"'
python3 SCRIPTS/repl_client.py /tmp/repl.sock 'print(greeting_message)'

# See all stored variables
python3 SCRIPTS/repl_client.py /tmp/repl.sock --vars
```

**Use the REPL for everything.** Finding files, searching content, reading
source, storing results — all of it. Every fact you discover goes into a
variable where it accumulates instead of evaporating.

```bash
python3 SCRIPTS/repl_client.py /tmp/repl.sock '
import glob, os, re

# Find files (instead of Glob tool)
project_source_files = glob.glob("/path/to/project/**/*.py", recursive=True)
project_source_files = [f for f in project_source_files if "/.git/" not in f]

# Measure them (instead of wc)
file_size_by_path = {f: os.path.getsize(f) for f in project_source_files}
total_source_bytes = sum(file_size_by_path.values())

# Search content (instead of Grep tool)
function_definition_matches = {}
for filepath in project_source_files:
    with open(filepath) as fh:
        for line_number, line_text in enumerate(fh, 1):
            if re.search(r"def process_", line_text):
                function_definition_matches.setdefault(filepath, []).append(
                    (line_number, line_text.strip()))

# Everything persists: project_source_files, file_size_by_path,
# total_source_bytes, function_definition_matches
print(f"{len(project_source_files)} files, {total_source_bytes/1024:.0f} KB, "
      f"{len(function_definition_matches)} files with matches")
'
```

## The Results Dict

All subagent findings go into one well-known dict: **`_decompose_results`**.

The REPL server initializes this dict automatically on startup. Do not
re-initialize it — that would wipe results from other subagents. Just
write to it:

```bash
python3 SCRIPTS/repl_client.py /tmp/repl.sock '
_decompose_results["auth_module_analysis"] = {"functions": [...], "issues": [...]}
'
```

Every subagent writes to `_decompose_results[key]`. Every parent reads from
`_decompose_results[key]`. The underscore prefix and specific name avoid
collisions with any user or project variables.

Keys should be descriptive: `'auth_module_function_signatures'`, not `'chunk1'`.

## The Workflow

### 1. Measure

Before reading any files, measure everything you intend to read. Use the
REPL (as above) or the bundled script:

```bash
python3 SCRIPTS/chunk_text.py info <file>
```

Measure ALL files — source, tests, docs, config. The most common failure
is measuring only the core source, classifying it as small, then also
reading tests and docs and blowing past the limit.

### 2. Choose a strategy

| Total Size | Strategy |
|:-----------|:---------|
| < 50KB | Read directly into REPL variables. No subagents needed. |
| 50KB–200KB | Fan out subagents — one per file or file group, parallel. |
| 200KB–1MB | Chunk + fan out + aggregate in REPL. |
| > 1MB | Two-level: chunk, fan out, aggregate chunks, synthesize. |

### 3. Announce

Tell the user what you found and what you plan to do. Example:

> 47 files, ~145KB total. Fanning out 4 parallel subagents: (1) core library,
> (2) test harness, (3) test cases, (4) docs + config. All results stored in
> `_decompose_results`.

Do not read any files before this step.

### 4. Execute

Fan out subagents. Each writes to `_decompose_results[key]`. You read the
results back. Details are in the next section.

### 5. Iterate

If the aggregated answer has gaps, target those specific areas for deeper
analysis. The REPL still holds everything from the first pass.

### The 50KB Rule

**Never read more than 50KB of source into your main context window.**
Everything above that limit must go through subagents that write findings
to the REPL. The cost of a subagent is latency. The cost of context
exhaustion is the entire rest of the session.

Watch for the trap: you measure 30KB of core source (under threshold!),
then also read tests, config, and docs — now you're at 120KB and your
context window is shot. Measure the total. All of it.

## Fan-Out Patterns

All patterns follow the same contract:

1. Parent starts the REPL server (once per session). `_decompose_results`
   is auto-initialized. **Never re-initialize it.**
2. Subagent writes findings to `_decompose_results[descriptive_key]`.
3. **Subagent reports back** the key it wrote and a summary of what's in it.
4. Parent reads `_decompose_results[key]` from the REPL.

Step 3 is critical. The Task tool returns a text message to the parent —
that message is the *only* way the parent learns what the subagent stored.
Every subagent prompt must end with an instruction to report what was written.

### Direct Query

For simple one-shot tasks (summarize, classify, extract a fact). The
subagent's return value is used directly — no REPL variable needed.

```
Task(subagent_type="Explore",
     prompt="Summarize the key functions in this file: <chunk>")
```

### Recursive Query

For sub-problems needing multi-step reasoning or tool access. The subagent
stores results in `_decompose_results` under a descriptive key.

**Parent launches subagent:**
```
Task(subagent_type="general-purpose",
     prompt="Use the REPL at /tmp/repl.sock. Read and analyze these modules.
     Store your findings in _decompose_results['auth_module_analysis'] as a
     dict with keys:
       'function_signatures' — list of all public function signatures
       'import_dependency_map' — dict mapping each file to its imports
       'identified_concerns' — list of architectural or correctness issues

     Files: src/auth.py, src/models.py, src/tokens.py

     When done, reply with: the key you wrote to in _decompose_results,
     what sub-keys you stored, and a one-line summary of each.")
```

**Parent receives** a message like: "Wrote to
`_decompose_results['auth_module_analysis']` with keys:
'function_signatures' (12 public functions), 'import_dependency_map' (3 files
mapped), 'identified_concerns' (2 issues: circular import between auth.py and
models.py, unused import in tokens.py)."

**Parent reads back:**
```bash
python3 SCRIPTS/repl_client.py /tmp/repl.sock '
auth_data = _decompose_results["auth_module_analysis"]
print("Functions:", auth_data["function_signatures"])
print("Concerns:", auth_data["identified_concerns"])
'
```

### Batched Parallel Query

For independent chunks that can run concurrently. Issue all Task calls in
a single message.

**Parent launches all subagents at once:**
```
Task(prompt="Use REPL at /tmp/repl.sock. Analyze this log segment for errors.
     Store in _decompose_results['log_segment_hours_00_to_06'] as a dict with
     keys 'error_summary' and 'critical_error_list'.
     Segment: <chunk1>

     When done, reply with: the _decompose_results key you wrote,
     how many errors found, and one sentence summarizing the most severe.")

Task(prompt="Use REPL at /tmp/repl.sock. Analyze this log segment for errors.
     Store in _decompose_results['log_segment_hours_06_to_12'] as a dict with
     keys 'error_summary' and 'critical_error_list'.
     Segment: <chunk2>

     When done, reply with: the _decompose_results key you wrote,
     how many errors found, and one sentence summarizing the most severe.")

Task(prompt="Use REPL at /tmp/repl.sock. Analyze this log segment for errors.
     Store in _decompose_results['log_segment_hours_12_to_18'] as a dict with
     keys 'error_summary' and 'critical_error_list'.
     Segment: <chunk3>

     When done, reply with: the _decompose_results key you wrote,
     how many errors found, and one sentence summarizing the most severe.")
```

**Parent receives** three messages confirming what each wrote.

**Parent reads accumulated results:**
```bash
python3 SCRIPTS/repl_client.py /tmp/repl.sock '
for segment_key, segment_data in sorted(_decompose_results.items()):
    if segment_key.startswith("log_segment_"):
        print(f"{segment_key}: {segment_data[\"error_summary\"]}")
        for critical_error in segment_data.get("critical_error_list", []):
            print(f"  - {critical_error}")
'
```

## Chunking

For large single files, use the bundled script to split at natural boundaries:

```bash
python3 SCRIPTS/chunk_text.py info large_file.txt      # measure
python3 SCRIPTS/chunk_text.py boundaries source.py      # find split points
python3 SCRIPTS/chunk_text.py chunk large_file.txt --size 80000 --overlap 200  # split
```

For structured files (code, markdown), prefer splitting at functions, classes,
or section headers rather than arbitrary character boundaries.

## When NOT to Decompose

- **< 50KB total** — Read into REPL variables directly. No subagents needed.
- **Global questions** — "What is the overall theme?" needs the full picture.
  Summarize first (in chunks if needed), then analyze the summary whole.
- **Quick answers** — When speed matters more than thoroughness.

## References

- `references/decomposition-patterns.md` — Five worked examples.
- `references/mapping-table.md` — RLM primitive to agent tool mapping.
- `references/rlm-system-prompt.md` — Theoretical foundation.
