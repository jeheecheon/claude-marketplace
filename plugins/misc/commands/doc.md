Generate a technical document on the topic in `$ARGUMENTS` for an LLM Agent reader with zero prior knowledge. Self-contained, unambiguous, exhaustive, yet token-lean.

# 1. Rules

## 1.1. Language
English only — headings, body, examples, comments. Translate non-English `$ARGUMENTS` internally.

## 1.2. Format
CommonMark. ATX headings (`#`..`######`), fenced code blocks, pipe tables, inline backticks for identifiers.

## 1.3. Numbering
Every heading carries a hierarchical decimal prefix matching depth: `# 1.`, `## 1.2.`, `### 1.2.3.`. Siblings monotonically increase from 1; no gaps, no reuse. Trailing dot required.

## 1.4. Deduplication
Each fact appears in exactly one section. Other sections cite it as `§ N.M.` (U+00A7 + space + number). Inline restatement forbidden. This is what makes § 1.3. load-bearing.

## 1.5. Detail
Reader has no domain knowledge. Therefore:
- Define every non-trivial term on first use (inline gloss or glossary section).
- State preconditions, postconditions, invariants, failure modes explicitly.
- Banned phrases: "obviously", "clearly", "as expected", "of course".
- Enumerate edge cases experts would treat as self-evident.
- When behavior varies by environment (OS, shell, runtime, locale), list each case.

## 1.6. Terminology
Use the precise term-of-art over colloquial paraphrase ("idempotent" not "safe to rerun"; "ABI" not "binary interface"). Gloss on first use so the reader resolves it without external lookup.

## 1.7. Token Economy
- Tables over repeated prose.
- Cross-references (§ 1.4.) over restatement.
- Short canonical names after first definition.
- Strip filler ("in order to"→"to", "due to the fact that"→"because").
- Never compress into ambiguity — clarity wins ties (§ 1.5.).

# 2. Procedure

1. Parse `$ARGUMENTS` as topic. If empty, ask and STOP.
2. Draft the section tree; validate against § 1.3. and § 1.4. before writing prose.
3. Write sections under § 1.1.–§ 1.7.
4. Self-audit: re-check every rule in § 1.
5. Emit Markdown directly — no outer code fence, no preamble, no closing remarks (§ 1.7.).

# 3. Input

`$ARGUMENTS` — topic. Required. May include scope hints (audience, exclusions); honor unless they conflict with § 1, in which case § 1 wins.
