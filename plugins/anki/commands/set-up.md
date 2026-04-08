Set up Anki for use with all `/anki:*` commands. Configures the MCP server, verifies AnkiConnect, installs dependencies, and recommends optimal Anki study settings.

# Context

- Purpose: One-time setup that prepares Anki's MCP connection and dependencies so that `/anki:cs`, `/anki:voca`, `/anki:deck`, and any deck-based commands work immediately after. Decks and note types are created on demand by each individual command.
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
├─ edge-tts installed? (python3 -m edge_tts --help)
│  ├─ YES → next
│  └─ NO → [A2] install it
│     ├─ OK → next
│     └─ FAIL → warn user, mark ❌ → next
│
└─ [A3] Report — show setup summary and optimal study settings
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

## A2. Install edge-tts

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

Mark as ❌ in the A3 summary and continue. Do NOT stop — the rest of the setup is still valid.

**If already installed:** Mark as ✅ and continue.

## A3. Report

Show a setup summary and optimal study settings.

### Setup Summary

```
┌─────────────────────────────────────────────────────┐
│              Anki 설정 완료 현황                       │
├─────────────────────────────────────────────────────┤
│  MCP 서버          ✅ / ❌                            │
│  AnkiConnect       ✅ / ❌                            │
│  edge-tts          ✅ / ❌                            │
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
