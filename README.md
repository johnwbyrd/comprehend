# comprehend

Deep codebase understanding for AI coding agents.

## The Problem

When AI coding agents encounter a large codebase, they typically do one of two things: skim too quickly and miss critical details, or read too much and exhaust their context window. Either way, they start writing code with a shallow understanding, and the results show it.

## What comprehend Does

comprehend teaches AI agents to *systematically understand* a codebase before touching it. Instead of dumping files into the context window, it uses a measure-first protocol that produces **smaller, richer context** than brute-force approaches:

1. **Measure** the problem (file count, total size, structure)
2. **Plan** the right strategy based on actual measurements
3. **Fan out** parallel subagents to read and analyze different parts of the codebase
4. **Accumulate** findings in a persistent REPL — facts survive across tool calls instead of evaporating
5. **Synthesize** a deep understanding from structured results

The persistent REPL is the key insight. It acts as shared memory: subagents write their findings to named variables, and the parent agent reads and aggregates them with plain Python. Nothing gets lost. Nothing bloats the context window.

## Why It Works Better

| | Without comprehend | With comprehend |
|:---|:---|:---|
| **Context usage** | Reads files directly into the agent's window until it runs out of room | Subagents do the reading; main agent stays small and focused |
| **Cross-file reasoning** | Reads file A, then file B, then forgets half of file A | Builds an architecture map in the REPL, then traces specific paths |
| **Large files** | Truncates or skims | Chunks at natural boundaries (function defs, markdown headers, timestamps) and analyzes each chunk in parallel |
| **State between calls** | Every tool call starts fresh | REPL variables, imports, and functions persist for the entire session |

## How It Works

comprehend is a [skill](https://skills.sh/) for Claude Code, OpenAI, Gemini, or whatever your favorite LLM is. It provides:

- **A context assessment protocol** -- measure before analyzing, choose the right strategy for the size
- **Three analysis primitives** -- direct (small), recursive (medium), batched parallel (large)
- **A persistent REPL server** -- Python process over Unix socket that maintains state across shell calls
- **A text chunking utility** -- splits files at natural boundaries, not arbitrary character counts
- **Worked examples** -- five real patterns (log analysis, code review, document Q&A, data comparison, cross-file tracing)

### The 50KB Rule

If total context is under 50KB, just read it directly. Above that, fan out subagents:

| Total size | Strategy |
|:---|:---|
| < 50KB | Read directly, no subagents needed |
| 50KB–200KB | One subagent per file or module, parallel |
| 200KB–1MB | Chunk files first, then fan out subagents |
| > 1MB | Two-level: chunk, fan out, aggregate, synthesize |

## Installation

### Via Skills.sh

```bash
npx skills add johnwbyrd/comprehend
```

### Local install (single project)

```bash
git clone https://github.com/johnwbyrd/comprehend.git
mkdir -p .claude/skills
cp -r comprehend/skills/comprehend .claude/skills/
```

### Global install (all projects)

```bash
git clone https://github.com/johnwbyrd/comprehend.git
mkdir -p ~/.claude/skills
cp -r comprehend/skills/comprehend ~/.claude/skills/
```

Compatible agents auto-discover skills at `.claude/skills/*/SKILL.md`. Commit the directory to share with your team.

## Usage

Invoke with `/comprehend`, or let it activate automatically when the agent encounters large-context analysis tasks — anything involving files over 50KB, multi-file analysis, or codebase-wide understanding.

### Bundled Scripts

All scripts are pure Python 3 with no external dependencies.

**chunk_text.py** — Measure and split files at natural boundaries:
```bash
python3 .claude/skills/comprehend/scripts/chunk_text.py info large_file.txt
python3 .claude/skills/comprehend/scripts/chunk_text.py chunk large_file.txt --size 80000
python3 .claude/skills/comprehend/scripts/chunk_text.py boundaries source.py
```

**repl_server.py / repl_client.py** — Persistent Python REPL over Unix socket (TCP on Windows):
```bash
REPL_ADDR=$(python3 .claude/skills/comprehend/scripts/repl_server.py --make-addr)
python3 .claude/skills/comprehend/scripts/repl_server.py "$REPL_ADDR" &

python3 .claude/skills/comprehend/scripts/repl_client.py "$REPL_ADDR" 'x = 42'
python3 .claude/skills/comprehend/scripts/repl_client.py "$REPL_ADDR" 'print(x + 1)'  # 43
python3 .claude/skills/comprehend/scripts/repl_client.py "$REPL_ADDR" --vars
python3 .claude/skills/comprehend/scripts/repl_client.py "$REPL_ADDR" --shutdown
```

## Background

Inspired by the [RLM framework](https://github.com/alexzhang13/rlm) from MIT OASYS lab ([paper](https://arxiv.org/abs/2512.24601), [blog](https://alexzhang13.github.io/blog/2025/rlm/)). The core idea -- that language models should decompose large problems into smaller ones, using persistent state to accumulate findings -- maps naturally onto the subagent + REPL architecture that modern coding agents already have.

## License

[AGPL-3.0](LICENSE)
