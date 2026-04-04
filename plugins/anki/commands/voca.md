Create Anki vocabulary cards from English words provided by the user.

# Context

- User: Korean learner of English
- Deck: **Vocabulary**
- Note Type: **Vocabulary** (standard, non-cloze, 1 card template)
- Rule: **1 word = 1 note = 1 card. No exceptions.**

# Decision Tree

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
├─ Deck "Vocabulary" exists? (deckActions → listDecks)
│  ├─ YES → next check
│  └─ NO → [A2] → next check
│
├─ Note type "Vocabulary" exists? (modelNames)
│  ├─ YES → verify 12 fields (modelFieldNames) → begin words
│  └─ NO → [A3] → begin words
│
├─ For each word:
│  │
│  ├─ [A4] Duplicate?
│  │  ├─ YES → skip, mark "duplicate" → next word
│  │  └─ NO → continue
│  │
│  ├─ [A5] Create note
│  │  ├─ OK → save note ID → continue
│  │  └─ FAIL → mark "error" → next word
│  │
│  ├─ [A6] Generate & attach audio
│  │  ├─ Word audio failed → mark "no word audio", continue
│  │  ├─ Sentence audio failed → mark "no sentence audio", continue
│  │  └─ OK → continue
│  │
│  └─ next word
│
└─ [A7] Report
```

# Actions

## A0. Anki MCP Not Configured

The **`anki`** MCP server is not installed. Run the following via Bash tool:

```bash
claude mcp add anki -- npx -y @alexanderadam/mcp-ankiconnect
```

Then tell the user: **"Please type `/reload-plugins` to activate the MCP, then retry."** **STOP — cannot proceed until MCP is loaded.**

## A1. Anki MCP Connection Failed

The `anki` MCP server is registered but cannot reach Anki. Guide the user through the following:

1. **Verify Anki is running** — The Anki app must be open.
2. **Install the AnkiConnect add-on** — An add-on that connects Anki with external programs.
   - Open Anki → **Tools → Add-ons → Get Add-ons...**
   - Enter code: `2055492159`
   - Click **OK** → **Restart Anki**
3. **Confirm AnkiConnect is active** — visit `http://localhost:8765` in a browser; it should return a response.

Guide the user to retry after completing the steps above. **STOP — cannot proceed without MCP.**

## A2. Create Deck

```
deckActions → createDeck, deckName: "Vocabulary"
```

## A3. Create Note Type

Create via `createModel` with the specification below.

### Fields (12, in order)

| # | Field | Description | Example |
|---|---|---|---|
| 1 | `Sentence` | Natural sentence with target word replaced by `<b>__________</b>`. Follows 1T principle: fully understandable except for the blank. | `She made a <b>__________</b> choice to leave her corporate job.` |
| 2 | `TargetWord` | The word being learned. | `deliberate` |
| 3 | `POS` | Part of speech (noun, verb, adj, adv, prep, etc.) | `adj` |
| 4 | `Pronunciation` | IPA transcription with slashes. | `/dɪˈlɪb.ər.ət/` |
| 5 | `KoreanMeaning` | Concise Korean translation of the word. | `deliberate, intentional` |
| 6 | `EnglishDefinition` | Monolingual English definition. | `done consciously and intentionally` |
| 7 | `Confusable` | Commonly confused word. Format: `≠ word (brief distinction)`. Empty if none. | `≠ careful (cautious, not intentional)` |
| 8 | `Collocations` | 2-3 common collocations separated by ` · `. | `deliberate decision · deliberate attempt` |
| 9 | `WordFamily` | Related word forms. Empty if no useful forms exist. | `n. deliberation · adv. deliberately` |
| 10 | `Examples` | Exactly 3 example sentences. Wrap target word in `<b>` tags. Separate with `<br>`. | `The damage was not <b>deliberate</b>.<br>...` |
| 11 | `Audio` | Word pronunciation. Left empty at creation; set in A6. | |
| 12 | `SentenceAudio` | Full sentence audio. Left empty at creation; set in A6. | |

### Card Template Structure

**Front:**
- Sentence with blank
- EnglishDefinition — always visible (NOT click-to-reveal). Must be displayed directly below the sentence at all times.

**Back (top to bottom):**
1. TargetWord
2. POS + Pronunciation
3. Audio (word)
4. KoreanMeaning
5. **── divider ──** (must be clearly visible, not subtle)
6. Filled sentence (blank replaced with TargetWord, highlighted) — requires JS to replace `<b>__________</b>` with `<b class="filled">{{text:TargetWord}}</b>`
7. SentenceAudio
8. **── divider ──** (must be clearly visible, not subtle)
9. Confusable — with section header label. Conditional: only shown if field is not empty.
10. Collocations — with section header label.
11. WordFamily — with section header label. Conditional: only shown if field is not empty.
12. Examples — with section header label.

Each section (9–12) must display a visible header/title (e.g. "Confusable", "Collocations", "Word Family", "Examples") above or alongside its content so the user can identify each section at a glance. Sections must be visually separated from each other (e.g. spacing, borders, or cards) so they don't blend together.

**Styling is NOT specified here.** The agent creating the note type should design clean, readable card styling with good visual hierarchy at its own discretion. Key constraint: text must have sufficient contrast against the background — no text should blend into or be hard to read against its background color.

## A4. Check Duplicate

```
findNotes → query: "deck:Vocabulary TargetWord:{word}"
```

- Returns note IDs → count > 0 means duplicate.

## A5. Create Note

```
addNotes → deckName: "Vocabulary", modelName: "Vocabulary",
           allowDuplicate: false, duplicateScope: "deck"
```

- Fill all fields EXCEPT Audio, SentenceAudio (leave empty).
- Tags: `voca` + headword (lowercase, hyphenated if multi-word).
- Save returned note ID for A6.

### Quality guidelines for field content

**Sentence:**
- 1T principle: the blank aside, every word in the sentence must be understandable to the learner. No other difficult words.
- The sentence context must make the target word recoverable — but not too obvious. Avoid ambiguous blanks where multiple words could fit.
- Natural, authentic sentences (news, essays, everyday conversation level) — not textbook-style.
- Choose a situation that reveals the core meaning of the word. Prefer literal/basic usage over figurative/idiomatic.

**Examples:**
- The 3 example sentences must show different contexts — no repeating the same situation. Each should demonstrate a different scene, register, or collocation.

**Other fields:**
- Confusable: what a Korean learner would actually confuse. Empty if none — don't force it.
- Collocations: high-frequency natural combinations only.
- WordFamily: only genuinely different/useful forms.
- Pronunciation: IPA with slashes.

## A6. Generate & Attach Audio

**IMPORTANT:** `updateNoteFields` audio parameter only supports `"url"` (HTTP), NOT local file paths. Use `mediaActions → storeMediaFile` to store local files first.

### Step 1: Generate two mp3s via edge-tts

```bash
python3 -m edge_tts --voice en-US-AriaNeural --text "{word}" --write-media /tmp/pronunciation_{word}_en.mp3
python3 -m edge_tts --voice en-US-AriaNeural --text "{full_sentence_with_word_filled_in}" --write-media /tmp/sentence_{word}_en.mp3
```

### Step 2: Store both in Anki media

```json
{"action": "storeMediaFile", "filename": "pronunciation_{word}_en.mp3", "path": "/tmp/pronunciation_{word}_en.mp3"}
{"action": "storeMediaFile", "filename": "sentence_{word}_en.mp3", "path": "/tmp/sentence_{word}_en.mp3"}
```

### Step 3: Set fields via updateNoteFields

```json
{"note": {"id": <note_id>, "fields": {"Audio": "[sound:pronunciation_{word}_en.mp3]", "SentenceAudio": "[sound:sentence_{word}_en.mp3]"}}}
```

## A7. Report

Show summary table:

| Word | POS | Korean | Word Audio | Sentence Audio |
|---|---|---|---|---|
| {word} | {pos} | {korean} | ✓ / ✗ | ✓ / ✗ |

# Input

`$ARGUMENTS` — single word, comma-separated list, or short phrase. Create one card per word.
