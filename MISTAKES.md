# MISTAKES.md

**Agent errors only. Newest first. Append, never rewrite history.**

Every time the agent breaks something, or a human corrects it, append an entry:

```
## YYYY-MM-DD — one-line title
**What happened:** the observable, not the intention.
**Root cause:** the mechanism, not "was careless".
**Consequence:** what it cost — wrong claim shipped, outage, wasted session.
**Rule:** the specific thing that prevents a repeat. Testable, not "be careful".
**Repeat:** Nth occurrence of <pattern>.
```

## The point is graduation, not confession

An entry is evidence. **When the same pattern reaches 3 occurrences it stops being a
mistake and becomes a law in `CLAUDE.md`** — where it is enforced on every session
rather than remembered by whoever read this file. This file is where evidence
accumulates; `CLAUDE.md` is where it gets applied. Always record the `**Repeat:**`
count: without it you get a vague sense that "that area is flaky" instead of a
countable pattern with a fix.

The best outcome is an entry **leaving this file** — because the rule moved to
`CLAUDE.md` and the failure stopped happening.

No tooling, no plugin, no vector store. It is a markdown file you grep.

⚠ Fleet-wide laws already graduated (see `ecp-standards:CLAUDE.md` and
`ecp-standards:MISTAKES.md` for the evidence behind each):
1. A probe must be shown to produce a NEGATIVE before its positives are believed.
2. Assert the effect, never the declaration.
3. `ssh` + stdin: work list needs `-n`; a piped payload dies from it. Never both.
4. Every anchored edit asserts its anchor and reads the result back.

---

*No entries yet for this repo. That means none have been recorded — not that none
have happened.*
