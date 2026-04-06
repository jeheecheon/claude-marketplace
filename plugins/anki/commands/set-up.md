Set up Anki for use with all `/anki:*` commands. Configures the MCP server, verifies AnkiConnect, creates required decks and note types, and recommends optimal Anki study settings.

# Context

- Purpose: One-time setup that prepares Anki so that `/anki:cs`, `/anki:voca`, `/anki:deck`, and any deck-based commands work immediately after.
- Rule: **Follow every step of the decision tree in order. Do not skip or reorder steps unless the user explicitly instructs otherwise.**

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
├─ Deck "Computer Science" exists? (deckActions → listDecks)
│  ├─ YES → next check
│  └─ NO → [A2a] create it → next check
│
├─ Deck "Vocabulary" exists? (check listDecks result)
│  ├─ YES → next check
│  └─ NO → [A2b] create it → next check
│
├─ Note type "Concept" exists? (modelNames)
│  ├─ YES → verify 9 fields (modelFieldNames)
│  │  ├─ Fields match → next check
│  │  └─ Fields mismatch → [A4] warn user → next check
│  └─ NO → [A3a] create it → next check
│
├─ Note type "Vocabulary" exists? (modelNames)
│  ├─ YES → verify 12 fields (modelFieldNames)
│  │  ├─ Fields match → next check
│  │  └─ Fields mismatch → [A4] warn user → next check
│  └─ NO → [A3b] create it → next check
│
├─ Note type "StudyCard" exists? (modelNames)
│  ├─ YES → verify 7 fields (modelFieldNames)
│  │  ├─ Fields match → next check
│  │  └─ Fields mismatch → [A4] warn user → next check
│  └─ NO → [A3c] create it → next check
│
├─ edge-tts installed? (python3 -m edge_tts --help)
│  ├─ YES → next
│  └─ NO → [A5] install it
│     ├─ OK → next
│     └─ FAIL → warn user, mark ❌ → next
│
└─ [A6] Report — show setup summary and optimal study settings
```

# Actions

## A0. Anki MCP Not Configured

The **`anki`** MCP server is not installed. Run the following via Bash tool:

```bash
claude mcp add anki -- npx -y @alexanderadam/mcp-ankiconnect
```

Then tell the user:

> **Anki MCP 서버를 설치했습니다.**
> **`/reload-plugins`를 입력하여 MCP를 활성화한 뒤, `/anki:set-up`을 다시 실행해주세요.**

**STOP — cannot proceed until MCP is loaded.**

## A1. Anki MCP Connection Failed

The `anki` MCP server is registered but cannot reach Anki. Guide the user:

> **Anki에 연결할 수 없습니다.** 아래 단계를 따라주세요:
>
> 1. **Anki 앱이 실행 중인지 확인하세요.**
> 2. **AnkiConnect 애드온을 설치하세요:**
>    - Anki 실행 → **도구(Tools) → 부가기능(Add-ons) → 부가기능 받기(Get Add-ons...)**
>    - 코드 입력: `2055492159`
>    - **확인(OK)** 클릭 → **Anki 재시작**
> 3. **AnkiConnect 활성화 확인:** 브라우저에서 `http://localhost:8765` 접속 시 응답이 있어야 합니다.
>
> 위 단계 완료 후 `/anki:set-up`을 다시 실행해주세요.

**STOP — cannot proceed without connection.**

## A2a. Create Deck "Computer Science"

```
deckActions → createDeck, deckName: "Computer Science"
```

Used by `/anki:cs`.

## A2b. Create Deck "Vocabulary"

```
deckActions → createDeck, deckName: "Vocabulary"
```

Used by `/anki:voca`.

## A3a. Create Note Type "Concept"

Create via `createModel` with the specification below. Used by `/anki:cs`.

**Fields (9, in order):**

| # | Field | Description |
|---|---|---|
| 1 | `Question` | Interview-style question |
| 2 | `Interview Answer` | Conversational model answer, 3-5 sentences |
| 3 | `Key Concept` | Precise technical definition, 1-2 sentences |
| 4 | `Pros` | Advantages compared to alternatives |
| 5 | `Cons` | Disadvantages compared to alternatives |
| 6 | `Keywords` | 3-5 essential keywords |
| 7 | `Misconception` | Common misconception (empty if none) |
| 8 | `Real Example` | Real-world usage with specific system names |
| 9 | `Follow Up` | 1-2 follow-up questions with brief answers |

**Card Template — Front:**
- INTERVIEW badge + Question — large, centered.

**Card Template — Back (top to bottom):**
1. Question echo (small, muted)
2. Interview Answer (largest, most prominent)
3. Keywords checklist
4. Key Concept
5. Misconception (conditional: hidden if empty)
6. Pros
7. Cons
8. Real Example (conditional: hidden if empty)
9. Follow Up (conditional: hidden if empty)

Each section must display a visible header/title so the user can identify each section at a glance. Sections must be visually separated from each other (e.g. spacing, borders, or background color differences) so they don't blend together.

**Styling:**
- Dark theme (GitHub Dark / VS Code Dark+ style)
- WCAG AA+ contrast
- Color coding: Pros (green), Cons (red), Misconception (orange/amber), Follow Up (gold), Real Example (teal/cyan)
- Korean readability: `word-break: keep-all`, `line-height: 1.6+`, `white-space: pre-line`
- Fonts: Pretendard / Noto Sans KR for Korean; JetBrains Mono / Fira Code for code
- Code blocks: distinct background, monospace, `overflow-x: auto`

**Styling is NOT fully specified here.** The agent creating the note type should design clean, readable card styling with good visual hierarchy at its own discretion, as long as the constraints above are satisfied.

## A3b. Create Note Type "Vocabulary"

Create via `createModel` with the specification below. Used by `/anki:voca`.

**Fields (12, in order):**

| # | Field | Description |
|---|---|---|
| 1 | `Sentence` | Natural sentence with blank (`<b>__________</b>`) |
| 2 | `TargetWord` | The word being learned |
| 3 | `POS` | Part of speech |
| 4 | `Pronunciation` | IPA transcription |
| 5 | `KoreanMeaning` | Korean translation |
| 6 | `EnglishDefinition` | Monolingual English definition |
| 7 | `Confusable` | Commonly confused word (empty if none) |
| 8 | `Collocations` | 2-3 common collocations |
| 9 | `WordFamily` | Related word forms (empty if none) |
| 10 | `Examples` | 3 example sentences |
| 11 | `Audio` | Word pronunciation audio (left empty at creation) |
| 12 | `SentenceAudio` | Full sentence audio (left empty at creation) |

**Card Template — Front:**
- Sentence with blank
- EnglishDefinition — always visible (NOT click-to-reveal). Must be displayed directly below the sentence at all times.

**Card Template — Back (top to bottom):**
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

Each section (9–12) must display a visible header/title so the user can identify each section at a glance. Sections must be visually separated from each other.

**Styling is NOT fully specified here.** The agent creating the note type should design clean, readable card styling with good visual hierarchy at its own discretion. Key constraint: text must have sufficient contrast against the background.

## A3c. Create Note Type "StudyCard"

Create via `createModel` with the specification below. Used by `/anki:deck` and custom deck commands created via `/anki:deck`.

**Fields (7, in order):**

| # | Field | Description |
|---|---|---|
| 1 | `Question` | The question text |
| 2 | `Choices` | MC choices (empty for subjective mode) |
| 3 | `Answer` | Subjective answer (empty for MC mode) |
| 4 | `CorrectChoice` | Correct MC choice (empty for subjective mode) |
| 5 | `Explanation` | Deeper elaboration |
| 6 | `Hint` | Click-to-reveal retrieval cue (optional) |
| 7 | `Misconception` | Common misconception (optional) |

**Card Template — Front:**

The front template MUST handle both modes using Anki's conditional syntax `{{#FieldName}}...{{/FieldName}}`.

1. **Question** — always displayed, large and centered.
2. **Choices block** — conditional: `{{#Choices}}...{{/Choices}}`
   - Shown ONLY when `Choices` field is not empty (MC mode).
   - Each choice on its own line, clearly numbered with circled numbers.
   - Hidden automatically in subjective mode because the field is empty.
3. **Hint** — conditional: `{{#Hint}}...{{/Hint}}`
   - Click-to-reveal retrieval cue using Anki's native `{{hint:Hint}}` syntax.
   - Renders as a clickable "Show Hint" button. When clicked, reveals the hint text.
   - Hidden automatically if the Hint field is empty.

**Card Template — Back (top to bottom):**

The back template MUST handle both modes using conditional blocks.

1. Question echo (small, muted)
2. Answer block — conditional: `{{#Answer}}...{{/Answer}}` (subjective only, most prominent)
3. CorrectChoice block — conditional: `{{#CorrectChoice}}...{{/CorrectChoice}}` (MC only, highlighted)
4. Explanation (always shown)
5. Misconception block — conditional: `{{#Misconception}}...{{/Misconception}}` (orange/amber)

Each section must display a visible header/title. Sections must be visually separated.

**Styling constraints:**
- Dark theme, WCAG AA+ contrast, card max-width centered
- Visual hierarchy: Question > Answer/CorrectChoice > Choices/Explanation > Misconception > Section titles
- Font weight: Question bold, Answer medium, section titles bold/uppercase
- Color coding: Answer (blue/cyan), CorrectChoice (green), Explanation (neutral), Misconception (orange/amber)
- Korean readability: `word-break: keep-all`, `line-height: 1.6+`, `white-space: pre-line`
- Korean sans-serif font with `@import`, monospace for code
- Code blocks: distinct background, rounded corners, `overflow-x: auto`
- Mobile responsive: reduced padding <480px, full-width tappable choices (44px min height)

**Styling is NOT fully specified here.** The agent creating the note type has full creative freedom to design the visual appearance as long as all listed constraints are satisfied.

## A4. Field Mismatch Warning

When a note type already exists but its fields do not match the expected schema, warn the user:

**Expected fields per note type:**

| Note Type | Count | Expected Fields |
|---|---|---|
| Concept | 9 | Question, Interview Answer, Key Concept, Pros, Cons, Keywords, Misconception, Real Example, Follow Up |
| Vocabulary | 12 | Sentence, TargetWord, POS, Pronunciation, KoreanMeaning, EnglishDefinition, Confusable, Collocations, WordFamily, Examples, Audio, SentenceAudio |
| StudyCard | 7 | Question, Choices, Answer, CorrectChoice, Explanation, Hint, Misconception |

Display the warning:

> **⚠️ "{NoteType}" 노트 타입의 필드가 예상과 다릅니다.**
>
> - 예상 필드: {expected fields}
> - 실제 필드: {actual fields}
>
> Anki에서 해당 노트 타입을 삭제한 뒤 `/anki:set-up`을 다시 실행하면 올바른 구조로 재생성합니다.
> (주의: 삭제하면 해당 노트 타입의 기존 카드도 삭제됩니다.)

Do NOT auto-delete or auto-modify existing note types. Continue to the next step.

## A5. Install edge-tts

Required for `/anki:voca` audio generation.

Check if installed:

```bash
python3 -m edge_tts --help 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```

**If not installed:**

```bash
pip3 install edge-tts
```

After installation, verify it succeeded:

```bash
python3 -m edge_tts --help 2>/dev/null && echo "INSTALLED" || echo "NOT_INSTALLED"
```

**If installation fails:** Warn the user:

> **⚠️ edge-tts 설치에 실패했습니다.**
>
> `/anki:voca` 명령어의 음성 생성 기능이 작동하지 않습니다.
> 카드 자체는 정상적으로 생성되지만, 발음 및 문장 오디오가 첨부되지 않습니다.
>
> 수동으로 설치하려면:
> ```
> pip3 install edge-tts
> ```
> 또는 Python 환경 문제가 있다면:
> ```
> python3 -m pip install edge-tts
> ```
>
> 설치 완료 후 `/anki:voca`를 사용하면 오디오가 정상적으로 생성됩니다.

Mark as ❌ in the A6 summary and continue. Do NOT stop — the rest of the setup is still valid.

**If already installed:** Mark as ✅ and continue.

## A6. Report

Show a setup summary and optimal study settings.

### Setup Summary

```
┌─────────────────────────────────────────────────────┐
│              Anki 설정 완료 현황                       │
├─────────────────────────────────────────────────────┤
│  MCP 서버          ✅ / ❌                            │
│  AnkiConnect       ✅ / ❌                            │
│  edge-tts          ✅ / ❌                            │
├─────────────────────────────────────────────────────┤
│  덱 (Decks)                                          │
│    Computer Science   ✅ 생성됨 / 이미 존재            │
│    Vocabulary         ✅ 생성됨 / 이미 존재            │
├─────────────────────────────────────────────────────┤
│  노트 타입 (Note Types)                               │
│    Concept (9 fields)    ✅ 생성됨 / 확인됨 / ⚠️      │
│    Vocabulary (12 fields) ✅ 생성됨 / 확인됨 / ⚠️     │
│    StudyCard (7 fields)  ✅ 생성됨 / 확인됨 / ⚠️      │
└─────────────────────────────────────────────────────┘
```

### Available Commands

| 명령어 | 설명 |
|---|---|
| `/anki:cs {topic}` | Computer Science 면접 카드 생성 |
| `/anki:voca {word}` | 영어 단어 카드 생성 (음성 포함) |
| `/anki:deck {name}` | 새 덱 커맨드 생성 |

### Optimal Anki Study Settings

Present these recommended settings to the user:

> **📖 최적의 학습 설정 (권장)**
>
> Anki에서 아래 설정을 적용하면 효율적인 학습이 가능합니다.
>
> **덱 옵션 (Deck Options) — 각 덱에서 톱니바퀴 → 옵션:**
>
> | 설정 | 권장값 | 이유 |
> |---|---|---|
> | 하루 새 카드 수 (New cards/day) | `20` | 과부하 방지. 익숙해지면 점진적으로 증가 |
> | 하루 최대 복습 수 (Maximum reviews/day) | `200` | 복습 밀림 방지 |
> | 학습 단계 (Learning steps) | `1m 10m` | 단기 기억 강화 |
> | 졸업 간격 (Graduating interval) | `1` 일 | 첫 복습까지 하루 대기 |
> | 쉬운 간격 (Easy interval) | `4` 일 | "쉬움" 누르면 4일 뒤 복습 |
> | 최대 간격 (Maximum interval) | `180` 일 | 6개월 이상 간격 방지 |
> | FSRS | `켜기 (ON)` | 최신 스케줄링 알고리즘 — SM-2보다 효율적 |
>
> **일반 설정 (Preferences → Review):**
>
> | 설정 | 권장값 |
> |---|---|
> | 다음 복습 시간 표시 (Show next review time) | `켜기` |
> | 남은 카드 수 표시 (Show remaining card count) | `켜기` |
> | 오디오 자동 재생 (Automatically play audio) | `켜기` |
>
> **FSRS 최적화 (선택사항):**
>
> 복습 데이터가 1,000개 이상 쌓이면:
> - **덱 옵션 → FSRS → Optimize** 클릭
> - 개인 학습 패턴에 맞춰 파라미터가 자동 조정됩니다
>
> 💡 **팁:** 매일 같은 시간에 복습하는 습관이 가장 중요합니다. 설정보다 꾸준함이 핵심입니다.

# Input

`$ARGUMENTS` — ignored. This command takes no arguments.
