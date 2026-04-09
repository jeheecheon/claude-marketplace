Create Anki flashcards for the **{{DECK_NAME}}** deck.

# Context

- User: Korean user studying {{DECK_DESCRIPTION}}
- Language: **All card content must be written in Korean.** Only code, technical terms, and proper nouns remain in English.
- Deck: **{{DECK_NAME}}**
- Note Type: **StudyCard** — single unified type that supports both subjective (free-response) and multiple-choice formats via optional fields.
- Rule: **1 topic = 1 note = 1 card. No exceptions.**
- Rule: **Follow every step of the decision tree in order. Do not skip or reorder steps unless the user explicitly instructs otherwise.**

# Card Modes

The same Note Type handles two modes. The mode is determined by user input, NOT by the Note Type itself.

| Mode | Trigger keywords | Front shows | Back shows |
|---|---|---|---|
| **Subjective** (default) | No mode keyword, or `주관식` | Question + Hint (click-to-reveal) | Answer, Explanation, Misconception |
| **Multiple-choice** | `객관식`, `4지선다`, `5지선다`, `선다형` | Question + Choices + Hint (click-to-reveal) | CorrectChoice, Choices (echo), Explanation, Misconception |

- Default is **subjective** when no mode keyword is present.
- For `4지선다`: generate 4 choices. For `5지선다` or plain `객관식`: generate 5 choices.

# Decision Tree

```
START
│
├─ Anki MCP available? (try calling modelNames)
│  ├─ Tool not found (no mcp__anki__* tools exist) → [A0] → STOP
│  ├─ Tool exists but call fails (connection error) → [A1] → STOP
│  └─ Call succeeds (returns model list) → next check
│
├─ Deck "{{DECK_NAME}}" exists? (deckActions → listDecks)
│  ├─ YES → next check
│  └─ NO → [A2] → next check
│
├─ Note type "StudyCard" exists? (check modelNames result for "StudyCard")
│  ├─ YES → verify fields (modelFieldNames for "StudyCard")
│  │  │  Expected fields in order: Question, Choices, Answer, CorrectChoice, Explanation, Hint, Misconception
│  │  ├─ Fields match (same names, same order, same count = 7) → next
│  │  └─ Fields mismatch → warn user: "StudyCard note type exists but has unexpected fields. Delete it in Anki (Tools → Manage Note Types → StudyCard → Delete) and retry." → STOP
│  └─ NO → [A3] → next
│
├─ [A4] Parse input → determine input type, mode, and topics
│  │
│  │  Step 1: Detect input type
│  │  ├─ Image attached? (user provided screenshot, photo, diagram)
│  │  │  ├─ YES → inputType = IMAGE → [A4-I] Analyze image
│  │  │  └─ NO → inputType = TEXT
│  │  │
│  │  Step 2: Parse text portion (from $ARGUMENTS or image analysis result)
│  │  ├─ Check for mode keywords at the END of input:
│  │  │  - `객관식`, `5지선다`, `선다형` → mode = MC5
│  │  │  - `4지선다` → mode = MC4
│  │  │  - `주관식` → mode = SUBJECTIVE
│  │  │  - No keyword → mode = SUBJECTIVE (default)
│  │  │
│  │  Step 3: (MC only) Check for user-provided choice specs
│  │  ├─ User provided explicit choices? (①②③④⑤ or numbered list in input)
│  │  │  ├─ YES → choiceSource = USER_PROVIDED
│  │  │  └─ NO → User provided choice requirements? (e.g. "시대별로", "인물 중심으로")
│  │  │     ├─ YES → choiceSource = USER_GUIDED
│  │  │     └─ NO → choiceSource = AGENT_GENERATED
│  │  │
│  │  Step 4: Extract topic(s) from remaining text
│  │  ├─ Comma-separated → multiple topics, same mode for all
│  │  └─ Single item → one topic
│  │
│  └─ next
│
├─ For each topic:
│  │
│  ├─ [A5] Duplicate?
│  │  ├─ YES → skip, mark "duplicate" → next topic
│  │  └─ NO → continue
│  │
│  ├─ mode = SUBJECTIVE?
│  │  ├─ YES → [A6-S] Create subjective note
│  │  └─ NO (MC4 or MC5) → [A6-M] Create multiple-choice note
│  │     │  choiceSource?
│  │     │  ├─ USER_PROVIDED → use user's exact choices
│  │     │  ├─ USER_GUIDED → generate choices following user's requirements
│  │     │  └─ AGENT_GENERATED → generate choices autonomously
│  │  ├─ OK → save note ID → next topic
│  │  └─ FAIL → mark "error" → next topic
│  │
│  └─ next topic
│
└─ [A7] Report
```

# Actions

## A0. Anki MCP Not Configured

No `mcp__anki__*` tools are available. The **`anki`** MCP server is not installed. Run the following via Bash tool:

```bash
claude mcp add anki -- npx -y @alexanderadam/mcp-ankiconnect
```

Then tell the user: **"Please type `/reload-plugins` to activate the MCP, then retry."** **STOP — cannot proceed until MCP is loaded.**

## A1. Anki MCP Connection Failed

The `mcp__anki__*` tools exist but the `modelNames` call returned an error (connection refused, timeout, etc.). The Anki app is likely not running or AnkiConnect is not installed. Instruct the user to:

1. **Verify Anki is running** — the Anki app must be open.
2. **Install AnkiConnect addon**:
   - Open Anki → **Tools → Add-ons → Get Add-ons...**
   - Enter code: `2055492159`
   - Click **OK** → **Restart Anki**
3. **Confirm AnkiConnect is active** — visit `http://localhost:8765` in a browser; it should return a response.

Ask the user to retry after completing the steps above. **STOP — cannot proceed without MCP.**

## A2. Create Deck

```
deckActions → createDeck, deckName: "{{DECK_NAME}}"
```

## A3. Create Note Type

If the note type does not exist, create it via `createModel` with these parameters:

```
createModel →
  modelName: "StudyCard"
  inOrderFields: ["Question", "Choices", "Answer", "CorrectChoice", "Explanation", "Hint", "Misconception"]
  cardTemplates: [{ Name: "Card 1", Front: <front HTML>, Back: <back HTML> }]
  css: <CSS string>
```

The agent MUST generate the **Front HTML**, **Back HTML**, and **CSS** based on the template structure and styling requirements specified below. Do NOT leave them empty or use placeholder text.

---

### StudyCard — Unified Note Type

**Fields (7, in this exact order):**

| # | Field | Required | Description |
|---|---|---|---|
| 1 | `Question` | Always | The question text. Natural Korean. One clear question per card. |
| 2 | `Choices` | MC only | Numbered choices. Format: `①선택지1<br>②선택지2<br>③선택지3<br>④선택지4<br>⑤선택지5`. **Leave empty for subjective mode.** |
| 3 | `Answer` | Subjective only | Full answer in conversational tone. 3-5 sentences. **Leave empty for MC mode.** |
| 4 | `CorrectChoice` | MC only | The correct choice label and text. Format: `③ 정답텍스트`. **Leave empty for subjective mode.** |
| 5 | `Explanation` | Always | For subjective: deeper elaboration complementing Answer (NOT a repeat). For MC: explain why the correct answer is correct AND why each distractor is wrong. |
| 6 | `Hint` | Optional | A short retrieval cue shown on the front via click-to-reveal (`{{hint:Hint}}`). Helps the learner recall without giving away the answer. e.g. era, category, related concept, first syllable. Leave empty if the question is straightforward. |
| 7 | `Misconception` | Optional | One common misconception. Format: "commonly thought X, but actually Y". Leave empty if none. |

**Field usage by mode:**

| Field | Subjective | Multiple-Choice |
|---|---|---|
| `Question` | Filled | Filled |
| `Choices` | **Empty** | Filled (4 or 5 choices) |
| `Answer` | Filled | **Empty** |
| `CorrectChoice` | **Empty** | Filled |
| `Explanation` | Filled | Filled |
| `Hint` | Optional | Optional |
| `Misconception` | Optional | Optional |

### Card Template — Front

The front template MUST handle both modes using Anki's conditional syntax `{{#FieldName}}...{{/FieldName}}`.

**Layout (top to bottom):**

1. **Question** — always displayed, large and centered.
2. **Choices block** — conditional: `{{#Choices}}...{{/Choices}}`
   - Shown ONLY when `Choices` field is not empty (MC mode).
   - Each choice on its own line, clearly numbered with ①②③④⑤.
   - Hidden automatically in subjective mode because the field is empty.
3. **Hint** — conditional: `{{#Hint}}...{{/Hint}}`
   - Click-to-reveal retrieval cue using Anki's native `{{hint:Hint}}` syntax.
   - Renders as a clickable "Show Hint" button. When clicked, reveals the hint text.
   - Hidden automatically if the Hint field is empty.

**Front template pseudocode:**
```html
<div class="front">
  <div class="question">{{Question}}</div>

  {{#Choices}}
  <div class="choices">{{Choices}}</div>
  {{/Choices}}

  {{#Hint}}
  <div class="hint">{{hint:Hint}}</div>
  {{/Hint}}
</div>
```

### Card Template — Back

The back template MUST handle both modes using conditional blocks.

**Layout (top to bottom):**

1. **Question echo** — small, muted repeat of the question (always shown).
2. **Answer block** — conditional: `{{#Answer}}...{{/Answer}}`
   - Shown ONLY in subjective mode. Most prominent section, large text.
3. **CorrectChoice block** — conditional: `{{#CorrectChoice}}...{{/CorrectChoice}}`
   - Shown ONLY in MC mode. Displays the correct answer prominently, highlighted.
4. **Choices echo** — conditional: `{{#Choices}}...{{/Choices}}`
   - Shown ONLY in MC mode. Re-displays all choices on the back so the learner can review them alongside the correct answer. Style them muted/smaller than the CorrectChoice.
5. **Explanation** — always shown. Detailed reasoning.
6. **Misconception block** — conditional: `{{#Misconception}}...{{/Misconception}}`
   - Shown only if field is not empty. Warning/amber styling.

**Back template pseudocode:**
```html
<div class="back">
  <div class="question-echo">{{Question}}</div>

  {{#Answer}}
  <div class="section answer">
    <div class="section-title">Answer</div>
    {{Answer}}
  </div>
  {{/Answer}}

  {{#CorrectChoice}}
  <div class="section correct-choice">
    <div class="section-title">정답</div>
    {{CorrectChoice}}
  </div>
  {{/CorrectChoice}}

  {{#Choices}}
  <div class="section choices-echo">
    <div class="section-title">보기</div>
    {{Choices}}
  </div>
  {{/Choices}}

  <div class="section explanation">
    <div class="section-title">Explanation</div>
    {{Explanation}}
  </div>

  {{#Misconception}}
  <div class="section misconception">
    <div class="section-title">Misconception</div>
    {{Misconception}}
  </div>
  {{/Misconception}}
</div>
```

### Styling Requirements

The agent creating the note type designs the visual styling (fonts, colors, sizes, spacing). The agent has full creative freedom on specific values, but **MUST** satisfy every UX constraint listed below. These constraints are non-negotiable.

#### 1. Theme & Contrast

- **Dark theme required** — dark background, light text. Any dark palette is acceptable (GitHub Dark, VS Code Dark+, Catppuccin, Dracula, etc.).
- **WCAG AA+ contrast** — every text element must have sufficient contrast against its background. No washed-out, faint, or hard-to-read text. Test: if you squint, can you still read it?
- **Card max-width** — set a comfortable reading width so text doesn't stretch edge-to-edge on wide screens. Center the card content.

#### 2. Visual Hierarchy (Font Size)

The following elements must be visually distinguishable from each other through font size differences. The agent chooses the actual sizes, but this **relative ordering must hold** (largest → smallest):

1. **Question (front)** — largest text on the card. The learner's eye must land here first.
2. **Answer / CorrectChoice (back)** — second largest. The primary answer content.
3. **Choices (front)** / **Explanation (back)** — standard body text size.
4. **Misconception (back)** — slightly smaller than body or same size.
5. **Section titles (back)** — smallest text. Labels only, not content.
6. **Question echo (back)** — small and visually muted. Must not compete with the answer.
7. **Hint (front, revealed)** — smaller than the question. Subtle, not dominant.

Use `rem` units (not `px`) so sizes scale with user's Anki font settings.

#### 3. Font Weight Hierarchy

- **Question:** bold or semibold — must feel prominent.
- **Answer / CorrectChoice:** medium weight — clearly readable but not as heavy as the question.
- **Section titles:** bold or uppercase — scannable labels.
- **All other text:** regular weight.

#### 4. Color Coding by Section

Each section on the back must have a **distinct accent color** so the learner can identify sections at a glance without reading the titles. The agent chooses the actual colors, but must follow these semantic associations:

| Section | Color semantics | Constraint |
|---|---|---|
| Answer (subjective) | Cool tone (blue/cyan family) | Must feel calm, authoritative |
| CorrectChoice (MC) | Positive tone (green family) | Must feel "correct", affirmative |
| Explanation | Neutral / default text color | No special accent — this is the baseline |
| Misconception | Warning tone (orange/amber family) | Must feel "caution", distinct from other sections |

Apply accent colors via **left-border, section title color, or background tint** — at least one visual indicator per section.

#### 5. Section Separation (Dividers)

- **Every section on the back must be visually separated.** The learner must instantly see where one section ends and the next begins.
- Use at least one of: colored left-borders, horizontal dividers, background color differences, or generous spacing.
- **Section titles** must be displayed above each section's content. Use a consistent label style (e.g., uppercase, small caps, or bold small text).
- **Question echo** at the top of the back must be visually separated from the first content section (e.g., a horizontal rule or extra spacing below it).

#### 6. Choices Styling (Front — MC mode)

- Each choice must be its own **visually distinct block** — not a run-on paragraph.
- Choices must have enough padding/spacing that they don't blur together.
- Circle numbers (①②③④⑤) should be visually distinguished from the choice text (e.g., bolder, different color, or badge-style).

#### 7. Hint Styling (Front)

- Anki's `{{hint:Hint}}` renders as a clickable link before reveal. Style it so it looks intentionally interactive but does **not** distract from the question.
- After reveal, hint text should be visually **subdued** (smaller, muted color, italic, etc.) — it is a support cue, not a primary element.
- Position: below the question (and below choices in MC mode).

#### 8. Korean Readability

- `word-break: keep-all` — prevents Korean words from breaking mid-syllable.
- `line-height: 1.6` or higher — Korean characters need more vertical breathing room than Latin text.
- `white-space: pre-line` on body text fields — preserves intentional line breaks.

#### 9. Font Selection

- **Body text:** a Korean-optimized sans-serif font. Use `@import` for web fonts with system fallbacks.
- **Code / technical terms:** a monospace font (any is acceptable). Must be visually distinct from body text.

#### 10. Code Blocks

- Background must be **visually distinct** from the card body (darker or lighter shade).
- Monospace font, rounded corners, horizontal scroll (`overflow-x: auto`) for long lines.
- Sufficient padding inside the block.

#### 11. Mobile Responsiveness

- Reduce card padding on narrow screens (< 480px).
- Choices must be full-width and tappable on mobile — minimum touch target height of `44px`.
- No horizontal overflow — all content must fit within the viewport width.

## A4. Parse Input

`$ARGUMENTS` contains the user's text input. The user may also attach images.

### Step 1: Detect input type

| Condition | inputType |
|---|---|
| User attached an image (screenshot, photo, diagram, scan) | **IMAGE** → go to A4-I first |
| Text only | **TEXT** → continue to Step 2 |

### Step 2: Parse mode keyword

Check for mode keywords at the **END** of the text input:

| Keywords | Mode |
|---|---|
| `객관식`, `5지선다`, `선다형` | **MC5** (5 choices) |
| `4지선다` | **MC4** (4 choices) |
| `주관식` | **SUBJECTIVE** |
| No keyword | **SUBJECTIVE** (default) |

Remove the mode keyword from input. The remainder is the topic(s) and optional choice specs.

### Step 3: Detect choice specifications (MC only)

If mode is MC4 or MC5, check whether the user provided choice details:

| Condition | choiceSource | Example |
|---|---|---|
| Input contains explicit circled-number choices (①②③...) or a numbered list of choices | **USER_PROVIDED** — use these exact choices as-is | `빗살무늬토기 객관식 ①신석기 ②구석기 ③청동기 ④철기 ⑤삼국` |
| Input contains choice-guiding instructions after the mode keyword. Indicators: verbs like "구성", "넣어", "만들어", "중심으로", or category words like "시대별", "인물별", "지역별" | **USER_GUIDED** — generate choices following user's constraints | `빗살무늬토기 객관식 시대별로 보기 구성해줘` |
| Input contains only the topic and mode keyword | **AGENT_GENERATED** — agent generates all choices autonomously | `빗살무늬토기 객관식` |

### Step 4: Extract topics

- Remove mode keyword and choice specs from input.
- If comma-separated → multiple topics, same mode and choiceSource for all.
- Single item → one topic.

### A4-I. Analyze Image Input

When the user attaches an image:

1. **Read the image** — analyze its content (textbook page, diagram, handwritten notes, exam paper, etc.)
2. **Extract information** — identify topics, facts, concepts, or questions visible in the image.
3. **Combine with text** — if the user also provided text input, use the text as additional instructions (e.g., mode keyword, choice specs, which parts of the image to focus on). If text is empty, derive topics from the image content alone.
4. **Determine topics** — produce a list of topics extracted from the image. Each topic becomes one card.
5. **Continue to Step 2** — parse mode and choice specs from the text portion (if any).

**Image input examples:**

| Image | Text | Result |
|---|---|---|
| Textbook page about 빗살무늬토기 | (empty) | SUBJECTIVE, topic: 빗살무늬토기 (extracted from image) |
| Textbook page about 빗살무늬토기 | `객관식` | MC5, topic: 빗살무늬토기 (extracted from image) |
| Diagram of 삼국시대 영토 | `삼국시대 영토 변화` | SUBJECTIVE, topic: 삼국시대 영토 변화 (text overrides/clarifies) |
| Exam paper with 5 questions | `객관식` | MC5, topics: each question from the exam becomes a card |
| Photo of handwritten notes | (empty) | SUBJECTIVE, topics: key concepts from notes |

**Important:** When creating cards from image content, the card fields (Question, Answer, Explanation, etc.) should be based on the **information in the image**, not the image itself. The image is a source of knowledge, not an attachment to the card.

### Parsing examples (all cases)

| Input | Image | Mode | choiceSource | Topics |
|---|---|---|---|---|
| `빗살무늬토기` | No | SUBJECTIVE | — | 빗살무늬토기 |
| `빗살무늬토기 주관식` | No | SUBJECTIVE | — | 빗살무늬토기 |
| `빗살무늬토기 객관식` | No | MC5 | AGENT_GENERATED | 빗살무늬토기 |
| `빗살무늬토기 5지선다` | No | MC5 | AGENT_GENERATED | 빗살무늬토기 |
| `빗살무늬토기 4지선다` | No | MC4 | AGENT_GENERATED | 빗살무늬토기 |
| `빗살무늬토기, 고인돌 객관식` | No | MC5 | AGENT_GENERATED | 빗살무늬토기, 고인돌 |
| `빗살무늬토기 객관식 ①신석기 ②구석기 ③청동기 ④철기 ⑤삼국` | No | MC5 | USER_PROVIDED | 빗살무늬토기 |
| `빗살무늬토기 객관식 시대별로 보기 구성` | No | MC5 | USER_GUIDED | 빗살무늬토기 |
| (empty) | Textbook page | SUBJECTIVE | — | (extracted from image) |
| `객관식` | Exam paper | MC5 | AGENT_GENERATED | (extracted from image) |

## A5. Check Duplicate

```
findNotes → query: "\"deck:{{DECK_NAME}}\" Question:*{topic}*"
```

**NOTE:** Anki search syntax requires quotes (not backslash escaping) for deck names with spaces. e.g. Use `"deck:My Deck"`, NOT `deck:My\ Deck`.

## Formatting Rule

- **Line breaks: Anki renders HTML, so use `<br>` for all line breaks. Never use `\n` — it renders as invisible whitespace, not a visible line break.**

## A6-S. Create Subjective Note

```
addNotes → deckName: "{{DECK_NAME}}", modelName: "StudyCard",
           allowDuplicate: false, duplicateScope: "deck",
           tags: [see Tagging section below]
```

**Fill these fields:**

| Field | Action |
|---|---|
| `Question` | Fill — natural Korean question about the topic |
| `Choices` | **Leave empty** |
| `Answer` | Fill — conversational answer, 3-5 sentences |
| `CorrectChoice` | **Leave empty** |
| `Explanation` | Fill — deeper elaboration, complementary to Answer |
| `Hint` | Fill if helpful, otherwise leave empty |
| `Misconception` | Fill if applicable, otherwise leave empty |

### Quality guidelines (Subjective)

**Question:**
- Must sound natural — professional Korean.
- One sentence. Direct. e.g. "~에 대해 설명해주세요.", "~란 무엇인가요?"

**Answer (most important field):**
- Write as if explaining to someone face-to-face. Natural, conversational tone.
- Answer ONLY what was asked. No tangents, no filler.
- 3-5 sentences max. If it needs more, the question scope is too broad — split it.
- Must be self-contained: understandable without reading other fields.

**Explanation:**
- Adds depth beyond the Answer — NOT a repeat.
- Can include historical context, comparisons, cause-and-effect, or significance.

**Hint:**
- One short phrase that jogs memory without giving away the answer.
- Good hints: era/period, category, related concept, first syllable, location, key person.
- Bad hints: the answer itself, overly vague ("중요한 것"), too long (full sentence).
- Leave empty if the question is straightforward enough to recall without help.

**Misconception:**
- One genuinely common confusion or misunderstanding.
- Format: "commonly thought X, but actually Y" — include both the misconception and correction.
- Leave empty if no clear misconception exists.

## A6-M. Create Multiple-Choice Note

```
addNotes → deckName: "{{DECK_NAME}}", modelName: "StudyCard",
           allowDuplicate: false, duplicateScope: "deck",
           tags: [see Tagging section below]
```

**Fill these fields:**

| Field | Action |
|---|---|
| `Question` | Fill — clear question about the topic |
| `Choices` | Fill — 4 or 5 numbered choices (see Choices section below) |
| `Answer` | **Leave empty** |
| `CorrectChoice` | Fill — correct choice label + text (see format below) |
| `Explanation` | Fill — why correct answer is correct, why each distractor is wrong |
| `Hint` | Fill if helpful (e.g. narrow down category), otherwise leave empty |
| `Misconception` | Fill if applicable, otherwise leave empty |

### Choices — how to fill based on choiceSource

**Format:** Use circled numbers. Separate each choice with `<br>`:

```
①선택지1<br>②선택지2<br>③선택지3<br>④선택지4<br>⑤선택지5
```

For MC4 (4지선다), use only ①②③④.

**choiceSource determines HOW to generate the Choices field:**

#### USER_PROVIDED — user gave explicit choices

Use the user's choices **exactly as provided**. Do NOT modify, reorder, add, or remove any choices.

1. Convert user's choices to the ①②③... format if not already.
2. Determine which choice is correct.
3. Fill `CorrectChoice` with the correct one.
4. Fill `Explanation` explaining why each is correct/incorrect.

Example input: `빗살무늬토기 객관식 ①신석기 ②구석기 ③청동기 ④철기 ⑤삼국`
→ Choices: `①신석기<br>②구석기<br>③청동기<br>④철기<br>⑤삼국`
→ CorrectChoice: `①신석기` (agent determines correctness)

#### USER_GUIDED — user gave choice construction requirements

Generate choices **following the user's constraints**. The user describes HOW choices should be structured (e.g., by era, by person, by region).

1. Read the user's guidance carefully.
2. Generate choices that follow the specified pattern/structure.
3. Ensure choices meet the quality guidelines below.
4. Fill remaining fields normally.

Example input: `빗살무늬토기 객관식 시대별로 보기 구성해줘`
→ Agent generates choices organized by historical period:
`①신석기 시대<br>②구석기 시대<br>③청동기 시대<br>④철기 시대<br>⑤삼국 시대`

#### AGENT_GENERATED — no user choice specs

Generate all choices autonomously following the quality guidelines below.

### CorrectChoice format

```
③ 정답텍스트
```

The label (e.g. ③) must match exactly one of the choices.

### Quality guidelines (Multiple-Choice)

**Question:**
- Clear, unambiguous. Must have exactly one correct answer.
- Avoid "다음 중 옳지 않은 것은?" style negatives when possible — positive framing preferred.

**Choices (applies to USER_GUIDED and AGENT_GENERATED):**
- All choices must be plausible. No obviously wrong "joke" answers.
- Choices should be similar in length and style — don't make the correct answer stand out by being longer or more detailed.
- Distractors should test genuine understanding, not trick the reader with wordplay.
- Randomize the position of the correct answer — do NOT always put it in the same slot.

**Explanation:**
- First state why the correct answer is correct.
- Then briefly explain why each distractor is incorrect (one sentence each).
- Format: `정답: ③ ... (이유). ① ... (왜 틀린지). ② ... (왜 틀린지). ④ ... (왜 틀린지). ⑤ ... (왜 틀린지).`

## A7. Report

Show summary table:

| Topic | Mode | Status |
|---|---|---|
| {topic} | 주관식/객관식 | created / duplicate / error |

# Tagging

- Always include `{{BASE_TAG}}` as base tag.
- Add `subjective` or `multiple-choice` tag based on mode.
- Add domain-specific tags relevant to the topic.
- Add the specific topic as a tag (lowercase, hyphenated).

# Input

`$ARGUMENTS` — topic(s) optionally followed by a mode keyword and/or choice specs. The user may also attach images as input sources. Parse all input per A4 rules, then create cards.
