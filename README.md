<div align="center">

```
    ███████╗ ██████╗  ██████╗ ██╗   ██╗██████╗
    ██╔════╝██╔═══██╗██╔═══██╗██║   ██║██╔══██╗
    █████╗  ██║   ██║██║   ██║██║   ██║██████╔╝
    ██╔══╝  ██║   ██║██║   ██║██║   ██║██╔══██╗
    ██║     ╚██████╔╝╚██████╔╝╚██████╔╝██████╔╝
    ╚═╝      ╚═════╝  ╚═════╝  ╚═════╝ ╚═════╝
```

### Fuck OpenClaw OAuth Usage Ban

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Exit Test](https://img.shields.io/badge/exit--test-afternoon--portable-brightgreen)]()
[![Runtime](https://img.shields.io/badge/runtime-liquid-blueviolet)]()
[![Built With](https://img.shields.io/badge/built%20with-os--moda-blue)](https://github.com/bolivian-peru/os-moda/)

**your moat is my afternoon.**

[philosophy](philosophy.md) · [skill](skill.md) · [website](index.html) · [os-moda](https://github.com/bolivian-peru/os-moda/)

</div>

---

### tl;dr

anthropic banned openclaw oauth on april 4. we swapped the entire runtime to claude code native in **4 hours**. zero skills broken. zero state lost. $40/day saved. the token is the receipt.

```
         ╔═══════════════╗                  ╔═══════════════╗
         ║   OPENCLAW    ║                  ║  CLAUDE CODE  ║
         ║               ║    ┌────────┐    ║               ║
         ║  api credits  ║───>│  SWAP  │───>║  oauth/max    ║
         ║  $40/day      ║    └────────┘    ║  $0 extra     ║
         ╚═══════╤═══════╝                  ╚═══════╤═══════╝
                 │                                  │
                 └──────────────┬───────────────────┘
                                │
                   ┌────────────▼────────────┐
                   │  ~/.openclaw/skills/*   │
                   │                         │
                   │  untouched. all of it.  │
                   └─────────────────────────┘
```

---

### the idea

in the age of agentic ai, any thin wrapper around an api can be replicated in one session. openclaw was a wrapper. anthropic changed the terms. we changed the `$PATH`. the skills, state, configs, secrets — all stayed. only the binary that parses the prompt changed.

> *"never change what the code does — only how it does it."*
> — [code-simplifier](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-simplifier/agents/code-simplifier.md)

**simplicity is portability. portability is sovereignty.**

---

### the proof

```
  time to swap .............. 4 hours
  skills broken ............. 0
  state files modified ...... 0
  glue code written ......... ~200 lines
  billing ................... $40/day → $0
  auth ...................... API key → OAuth file
  invocation ................ openclaw run → claude -p
```

---

### use it

```bash
mkdir -p ~/.claude/skills/clawswap
cp skill.md ~/.claude/skills/clawswap/SKILL.md
claude            # then type /clawswap
```

---

### the repo

```
  README.md ........... you are here
  philosophy.md ....... the movement — liquid software, exit test, brand
  skill.md ............ the tool — full 10-step migration, copy and run
  index.html .......... the website — single file, no deps, deploy anywhere
  LICENSE ............. MIT
```

---

### what this is not

not anti-anthropic — we pay for claude and use their binary as intended.
not a hack — `claude -p` with oauth is a supported, documented flow.
not anti-openclaw — solid product, the billing change is the problem.

---

### $FOOUB

not a financial instrument. a timestamp. on april 9, 2026, four hours after sitting down, the community shipped a full runtime migration using the platform's own tools. built on [os-moda](https://github.com/bolivian-peru/os-moda/). works on any openclaw. the token is the receipt.

```
  they sent an email. we sent a commit.
```

---

<div align="center">

[MIT](LICENSE) — fork it. ship it. keep your claws.

</div>
