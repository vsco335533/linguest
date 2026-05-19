# Linguist v2 — L1/L2 Language Awareness: Developer Guide

## Overview

Linguist v2 adds **native language (L1) detection and adaptive AI behavior** to the existing chat and voice pipeline. The goal is to meet students where they are linguistically — accepting their mother tongue as a valid starting point — and progressively guide them toward English (L2) through contextual AI responses.

> **Key principle:** Treat typed input and speech input identically after transcription. Everything flows through one unified L1/L2 logic layer.

---

## What Changes vs. What Stays

### No Changes Required

| Component | Reason |
|---|---|
| Speech-to-text pipeline | Already produces text output — plug L1/L2 logic after it |
| `POST /chat/stream` API | Interface stays the same |
| SSE streaming | No change |
| Analytics engine (grammar, complexity, etc.) | Extended, not replaced |

### What Changes

| Component | Change |
|---|---|
| `chat_service.py` | Add language detection + pass `input_language` to prompt |
| Prompt template | New context fields: `language_mode`, `native_language`, `input_language` |
| Database | New `language_mode` field on `UserProfile` |
| `analytics_service.py` | New `language_usage` metric |
| `MetricDelta` table | Store `l1_percentage` / `english_percentage` per message |
| Frontend `ChatPage` | Input-type icons (🎤 / ⌨️), L1 hint banner, "Try in English" button |
| Progression logic | Language shift auto-trigger (L1 → MIXED → L2) |

---

## Phase 1 — Backend Core (Immediate Priority)

### 1.1 Database: Add `language_mode` to `UserProfile`

```sql
ALTER TABLE UserProfile
ADD COLUMN language_mode VARCHAR(10) DEFAULT 'L1'
CHECK (language_mode IN ('L1', 'MIXED', 'L2'));
```

| Value | Meaning |
|---|---|
| `L1` | Student primarily uses native language |
| `MIXED` | Student mixes L1 and English |
| `L2` | Student primarily uses English |

---

### 1.2 Backend: Language Detection

**File:** `linguist-backend/app/services/chat_service.py`

Install dependency:
```bash
pip install langdetect
```

Add to `requirements.txt`:
```
langdetect==1.0.9
```

Add the detection function:
```python
from langdetect import detect

def detect_input_language(text: str) -> str:
    """
    Detects the language of the input text.
    Returns ISO 639-1 language code (e.g. 'en', 'te', 'hi')
    or 'unknown' on failure.
    """
    try:
        return detect(text)
    except Exception:
        return "unknown"
```

Integrate into the chat handler:
```python
def handle_chat(user_message, cefr_level, native_language, language_mode, ...):
    input_language = detect_input_language(user_message)

    # Pass to prompt builder
    prompt = build_prompt(
        cefr_level=cefr_level,
        language_mode=language_mode,
        native_language=native_language,
        input_language=input_language,
        ...
    )
```

---

### 1.3 Backend: Updated Prompt Template

**File:** `linguist-backend/app/constants/prompts.py`

Replace/extend the existing system prompt to include language context:

```python
LINGUIST_SYSTEM_PROMPT = """
You are Linguist — a warm, encouraging English learning companion for young students.

User CEFR Level: {cefr_level}
Language Mode: {language_mode}
Native Language: {native_language}
User Input Language: {input_language}

Current Story Paragraph:
{paragraph_text}

Conversation History:
{history}

Mistake Summary:
{mistake_summary}

Response Rules:

1. If the user writes in their native language (L1):
   - Acknowledge briefly in their language to show you understood
   - Provide the English equivalent of what they said
   - Ask them to try saying it in English
   - Example: "చాలా బాగుంది! In English: 'The fox is clever.' Try saying it! 😊"

2. If the user writes in mixed language (L1 + English):
   - Respond primarily in English
   - Use native language only for a single word of support if needed
   - Gently reinforce the English portions they used correctly

3. If the user writes in English:
   - Respond fully in English
   - Correct mistakes naturally and encouragingly (do NOT lecture)
   - Increase conversational complexity gradually

4. General rules (always apply):
   - Keep responses under 3 sentences
   - Never translate everything directly — encourage production, not dependency
   - Stay on topic (the current paragraph/story)
   - Tone: warm, playful, never condescending
"""
```

---

### 1.4 API: Pass `language_mode` and `native_language` in Chat Request

**File:** `linguist-backend/app/chat/routes.py`

Update the chat endpoint to accept and forward these fields:

```python
@chat_bp.route('/chat/stream', methods=['POST'])
@jwt_required()
def stream_chat():
    data = request.get_json()
    user_id          = data.get('user_id')
    paragraph_id     = data.get('paragraph_id')
    message          = data.get('message')
    conversation_set_id = data.get('conversation_set_id')
    language_mode    = data.get('language_mode', 'L1')   # NEW
    native_language  = data.get('native_language', 'English')  # NEW
    ...
```

**Updated request payload (frontend → backend):**
```json
{
  "user_id": 1,
  "paragraph_id": 5,
  "message": "నేను fox గురించి చదివాను",
  "conversation_set_id": 12,
  "language_mode": "L1",
  "native_language": "Telugu"
}
```

---

## Phase 2 — Frontend Updates

### 2.1 Input-Type Icon on Messages

**File:** `linguist-frontent/src/pages/chatPage/ChatPage.jsx`

Track whether each message was typed or spoken, then display an icon:

```jsx
// When sending a message, include input_type
const sendMessage = (text, inputType = 'typed') => {
  const message = {
    content: text,
    sender: 'student',
    input_type: inputType,   // 'typed' | 'voice'
    timestamp: Date.now()
  };
  setMessages(prev => [...prev, message]);
  // ... send to API
};

// In the message bubble render
<Box sx={{ display: 'flex', alignItems: 'center', gap: 0.5 }}>
  {msg.input_type === 'voice' ? (
    <MicIcon fontSize="small" sx={{ color: 'grey.500' }} />
  ) : (
    <KeyboardIcon fontSize="small" sx={{ color: 'grey.500' }} />
  )}
  <Typography>{msg.content}</Typography>
</Box>
```

---

### 2.2 L1 Hint Banner

**File:** `linguist-frontent/src/pages/chatPage/ChatPage.jsx`

Show a contextual hint when the AI detects the user spoke in L1:

```jsx
{showL1Hint && (
  <Alert
    severity="info"
    icon={<TipsAndUpdatesIcon />}
    onClose={() => setShowL1Hint(false)}
    sx={{ mb: 1, borderRadius: 2 }}
  >
    💡 Try saying this in English — you can do it!
  </Alert>
)}
```

Trigger: set `showL1Hint = true` when the API response includes `"input_language": "te"` (or any non-`en` code).

---

### 2.3 "Try in English" Button

**File:** `linguist-frontent/src/pages/chatPage/ChatPage.jsx`

After an AI response to an L1 message, show a prompt-to-speak button:

```jsx
{showTryInEnglish && (
  <Box sx={{ display: 'flex', justifyContent: 'center', my: 1 }}>
    <Button
      variant="outlined"
      startIcon={<MicIcon />}
      onClick={handleTryInEnglish}
      sx={{ borderRadius: 3 }}
    >
      🎤 Try in English
    </Button>
  </Box>
)}

const handleTryInEnglish = () => {
  setShowTryInEnglish(false);
  // Auto-activate mic / speech input
  startSpeechInput();
};
```

---

### 2.4 Send `language_mode` and `native_language` from Frontend

**File:** `linguist-frontent/src/services/moduleApi.js`

Read from user profile (fetched at login) and include in every chat request:

```js
export const sendChatMessage = async (payload) => {
  const language_mode   = sessionStorage.getItem('language_mode') || 'L1';
  const native_language = sessionStorage.getItem('native_language') || 'English';

  return axiosInstance.post('/api/v1/chat/stream', {
    ...payload,
    language_mode,
    native_language,
  });
};
```

Store `language_mode` and `native_language` in `sessionStorage` after login (alongside `user_id`).

---

## Phase 3 — Analytics & Auto-Progression

### 3.1 New Metric: `language_usage`

**File:** `linguist-backend/app/services/analytics_service.py`

Add language usage tracking to `TextAnalyzer`:

```python
def calculate_language_usage(self, messages: list[dict]) -> dict:
    """
    Given a list of message dicts with 'detected_language' field,
    returns the percentage split between L1 and English.
    """
    total = len(messages)
    if total == 0:
        return {"l1_percentage": 0, "english_percentage": 0}

    english_count = sum(1 for m in messages if m.get('detected_language') == 'en')
    l1_count = total - english_count

    return {
        "l1_percentage": round((l1_count / total) * 100),
        "english_percentage": round((english_count / total) * 100)
    }
```

**Store in `MetricDelta`** (extend existing JSON structure):
```json
{
  "grammar": 0.82,
  "complexity": 0.65,
  "discourse": 0.70,
  "politeness": 0.50,
  "idiomatic": 0.30,
  "cosine_similarity": 0.45,
  "language_usage": {
    "l1_percentage": 70,
    "english_percentage": 30
  }
}
```

---

### 3.2 Auto-Progression: Language Mode Shift

**File:** `linguist-backend/app/services/chat_service.py` or a new `progression_service.py`

After each message, check the rolling window of the last 10 messages:

```python
def evaluate_language_progression(user_id: int, db_conn) -> str | None:
    """
    Checks the last 10 messages for a user.
    Returns the new language_mode if a transition is warranted, else None.
    """
    query = """
        SELECT detected_language
        FROM MetricDelta
        WHERE user_id = %s
        ORDER BY created_at DESC
        LIMIT 10
    """
    rows = db_conn.execute(query, (user_id,)).fetchall()

    if len(rows) < 10:
        return None  # Not enough data yet

    english_pct = sum(1 for r in rows if r['detected_language'] == 'en') / len(rows) * 100

    if english_pct >= 60:
        return 'MIXED'   # L1 → MIXED
    if english_pct >= 85:
        return 'L2'      # MIXED → L2

    return None


def apply_language_progression(user_id: int, new_mode: str, db_conn):
    db_conn.execute(
        "UPDATE UserProfile SET language_mode = %s WHERE user_id = %s",
        (new_mode, user_id)
    )
    db_conn.commit()
```

**Transition thresholds:**

| Condition | Transition |
|---|---|
| English usage ≥ 60% (last 10 msgs) | `L1` → `MIXED` |
| English usage ≥ 85% (last 10 msgs) | `MIXED` → `L2` |

---

## Updated Overall Chat Flow

```
Student types/speaks
        │
        ▼
[Speech-to-Text]  ←── (unchanged if voice)
        │
        ▼
   text input
        │
        ▼
detect_input_language(text)
        │
        ├── 'te' / 'hi' / other L1
        │       └── AI: acknowledge + model English + prompt repeat
        │
        ├── 'en' (broken / partial)
        │       └── AI: correct subtly + encourage
        │
        └── 'en' (fluent)
                └── AI: full English + increase challenge
        │
        ▼
build_prompt(cefr_level, language_mode, native_language, input_language, ...)
        │
        ▼
OllamaLLM → SSE stream → Frontend
        │
        ▼
analytics_service: compute metrics + language_usage
        │
        ▼
evaluate_language_progression → update language_mode if needed
        │
        ▼
Save to PostgreSQL (MetricDelta, Attempt, UserProfile)
```

---

## Implementation Checklist

### Phase 1 — Backend Core
- [ ] Add `language_mode` column to `UserProfile` (migration SQL)
- [ ] Install `langdetect`, add to `requirements.txt`
- [ ] Add `detect_input_language()` to `chat_service.py`
- [ ] Update prompt template in `prompts.py`
- [ ] Update `/chat/stream` route to accept `language_mode` + `native_language`
- [ ] Pass `input_language` into LLM context
- [ ] Return `input_language` in API response (for frontend hints)

### Phase 2 — Frontend
- [ ] Store `language_mode` + `native_language` in `sessionStorage` after login
- [ ] Send them with every chat request in `moduleApi.js`
- [ ] Add `input_type` tracking to messages (`'typed'` / `'voice'`)
- [ ] Render 🎤 / ⌨️ icon on message bubbles
- [ ] Show L1 hint banner when `input_language !== 'en'`
- [ ] Show "Try in English" button after AI response to L1 message
- [ ] Wire "Try in English" button to speech input activation

### Phase 3 — Analytics & Progression
- [ ] Extend `TextAnalyzer` with `calculate_language_usage()`
- [ ] Store `language_usage` in `MetricDelta` JSON
- [ ] Add `detected_language` column to `MetricDelta`
- [ ] Implement `evaluate_language_progression()`
- [ ] Call progression check after every chat turn
- [ ] Surface `language_usage` on `HomePage` progress chart

---

## What NOT to Change

| Do not touch | Reason |
|---|---|
| Speech-to-text pipeline | Already works; L1/L2 logic plugs in after it |
| `/chat/stream` SSE protocol | Only the payload fields expand |
| Analytics weights formula | Extend it, don't replace it |
| LangGraph memory store | No change needed |
| Quiz-based CEFR progression | Language-shift progression runs alongside it, not instead |

---

## Dependencies Added

| Package | Version | Purpose |
|---|---|---|
| `langdetect` | `1.0.9` | Lightweight language detection (ISO 639-1 codes) |

No new frontend npm packages required — MUI already has `Alert`, `Button`, `MicIcon`, `KeyboardIcon`.

---

*Specification source: `test.md` — Linguist v2 Dev Changes*
*Target branch: `feature/pradeep`*
