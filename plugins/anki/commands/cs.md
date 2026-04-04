Create Anki computer science flashcards optimized for technical interview preparation.

# Context

- User: Korean software developer preparing for technical interviews
- Language: **All card content must be written in Korean.** Only code, technical terms, and proper nouns remain in English.
- Deck: **Computer Science**
- Note Type: **Concept** — single unified type for all CS topics (concepts, algorithms, system design, principles, data structures, etc.)
- Rule: **1 topic = 1 note = 1 card. No exceptions.**

# Decision Tree

> **Rule: Follow every step of the decision tree in order. Do not skip or reorder steps unless the user explicitly instructs otherwise.**

```
START
│
├─ Anki MCP configured? (check if anki MCP tools are present)
│  ├─ NO → [A0] → STOP
│  └─ YES → next check
│
├─ Anki MCP reachable? (try any Anki MCP call, e.g. modelNames)
│  ├─ YES → next check
│  └─ NO → [A1] → STOP
│
├─ Deck "Computer Science" exists? (deckActions → listDecks)
│  ├─ YES → next check
│  └─ NO → [A2] → next check
│
├─ Note type "Concept" exists? (modelNames)
│  ├─ YES → verify 9 fields (modelFieldNames) → begin topics
│  └─ NO → [A3] → begin topics
│
├─ For each topic:
│  │
│  ├─ [A4] Duplicate?
│  │  ├─ YES → skip, mark "duplicate" → next topic
│  │  └─ NO → continue
│  │
│  ├─ [A5] Create note
│  │  ├─ OK → save note ID → next topic
│  │  └─ FAIL → mark "error" → next topic
│  │
│  └─ next topic
│
└─ [A6] Report
```

# Actions

## A0. Anki MCP Not Configured

The **`anki`** MCP server is not installed. Run the following via Bash tool:

```bash
claude mcp add anki -- npx -y @alexanderadam/mcp-ankiconnect
```

Then tell the user: **"Please type `/reload-plugins` to activate the MCP, then retry."** **STOP — cannot proceed until MCP is loaded.**

## A1. Anki MCP Connection Failed

The `anki` MCP server is registered but cannot reach Anki. Instruct the user to:

1. **Verify Anki is running** — the Anki app must be open.
2. **Install AnkiConnect addon**:
   - Open Anki → **Tools → Add-ons → Get Add-ons...**
   - Enter code: `2055492159`
   - Click **OK** → **Restart Anki**
3. **Confirm AnkiConnect is active** — visit `http://localhost:8765` in a browser; it should return a response.

Ask the user to retry after completing the steps above. **STOP — cannot proceed without MCP.**

## A2. Create Deck

```
deckActions → createDeck, deckName: "Computer Science"
```

## A3. Create Note Type

If the note type does not exist, create it via `createModel` with the specification below.

---

### Concept (standard type)

**Fields (9, in order):**

| # | Field | Description |
|---|---|---|
| 1 | `Question` | A single interview-style question. e.g. "CAP 정리에 대해 설명해주세요." |
| 2 | `Interview Answer` | Model answer in conversational tone, as if speaking to an interviewer. Concise 3-5 sentences. |
| 3 | `Key Concept` | Precise technical definition in 1-2 sentences. Must NOT repeat Interview Answer. |
| 4 | `Pros` | Advantages only. Include specific benefits compared to alternative approaches. |
| 5 | `Cons` | Disadvantages only. Include specific limitations compared to alternatives. |
| 6 | `Keywords` | 3-5 essential keywords. Used as a self-test checklist. |
| 7 | `Misconception` | One common misconception. Leave empty if none exists. |
| 8 | `Real Example` | Real-world usage in specific systems/services/code. Concrete system names required. |
| 9 | `Follow Up` | 1-2 anticipated follow-up questions with brief answers. |

### Card Template Structure

**Front:**
- INTERVIEW badge + Question — displayed large and centered.

**Back (top to bottom):**
1. Question — echo (small, muted)
2. Interview Answer — model answer (largest, most prominent section)
3. Keywords — keyword checklist
4. Key Concept — technical definition
5. Misconception — common misconception. Conditional: hidden if field is empty.
6. Pros — advantages
7. Cons — disadvantages
8. Real Example — real-world usage. Conditional: hidden if field is empty.
9. Follow Up — follow-up questions. Conditional: hidden if field is empty.

Each section must display a visible header/title so the user can identify each section at a glance. Sections must be visually separated from each other (e.g. spacing, borders, or background color differences) so they don't blend together.

**Styling is NOT fully specified here.** The agent creating the note type should design clean, readable card styling with good visual hierarchy at its own discretion. Key constraints:

- **Dark theme** — dark background with light text, GitHub Dark / VS Code Dark+ style.
- **Sufficient contrast** — text must never blend into or look washed out against its background (WCAG AA+).
- **Color coding** — Pros (green), Cons (red), Misconception (orange/amber), Follow Up (gold), Real Example (teal/cyan).
- **Korean readability** — `word-break: keep-all`, `line-height` 1.6+, `white-space: pre-line` on body text.
- **Fonts** — Korean text: Pretendard / Noto Sans KR. Code: JetBrains Mono / Fira Code.
- **Code blocks** — distinct background from body, monospace font, `overflow-x: auto`.

## A4. Check Duplicate

```
findNotes → query: "\"deck:Computer Science\" {topic-specific query}"
```

**NOTE:** Anki search syntax requires quotes (not backslash escaping) for deck names with spaces. Use `"deck:Computer Science"`, NOT `deck:Computer\ Science`.

## A5. Create Note

```
addNotes → deckName: "Computer Science", modelName: "Concept",
           allowDuplicate: false, duplicateScope: "deck"
```

### Tagging

- Always include `cs` as base tag.
- Add domain tags: `algorithm`, `data-structure`, `system-design`, `database`, `networking`, `os`, `python`, `go`, `java`, `docker`, `git`, `sql`, etc.
- Add the specific topic: `binary-search`, `cap-theorem`, `list-comprehension`, etc.

### Quality guidelines

**Interview Answer (most important field):**
- Write as if speaking to an interviewer face-to-face. Natural, conversational tone.
- Answer ONLY what was asked. No tangents, no filler.
- 3-5 sentences max. If it needs more, the question scope is too broad — split it.
- Must be self-contained: understandable without reading other fields.
- For algorithm topics: naturally weave in the core idea, when to apply, and complexity.

**Question:**
- Must sound like a real interviewer asking — natural, professional Korean.
- One sentence. Direct. e.g. "인덱스가 무엇이고 왜 사용하나요?", not "인덱스의 정의를 서술하시오."

**Pros:**
- Advantages only. Specific benefits compared to alternative approaches.
- For algorithms: compare against brute force or other alternatives.

**Cons:**
- Disadvantages only. Specific limitations compared to alternatives.
- Mention alternatives or mitigations when possible.

**Keywords:**
- 3-5 terms. Only domain-specific terms essential for the interview answer.
- No generic words (e.g. "성능", "효율"). Only terms unique to the topic.

**Misconception:**
- One genuinely common confusion or misunderstanding.
- Format: "commonly thought X, but actually Y" — include both the misconception and correction.
- Leave empty if no clear misconception exists.

**Key Concept:**
- Precise technical definition — NOT a repeat of Interview Answer. Complementary depth.
- For algorithms: specify pattern name, core mechanism, and time/space complexity.

**Real Example:**
- Concept topics: name specific systems/tools (e.g. "Redis cache eviction", "Kafka partitioning"). Vague examples like "used in large-scale systems" are not acceptable.
- Algorithm topics: minimal but complete code skeleton (core logic only, no boilerplate) or representative problem examples (e.g. LeetCode). Both may be included.

**Follow Up:**
- Anticipate what an interviewer would ask NEXT based on Interview Answer. Include a brief answer.
- For algorithms: edge cases (off-by-one, overflow, empty input), pattern variations, complexity proofs, etc.

## A6. Report

Show summary table:

| Topic | Cards Created |
|---|---|
| {topic} | {count} |

# Input

`$ARGUMENTS` — single topic, comma-separated list, specific request, or broad area. Parse input, create cards.
