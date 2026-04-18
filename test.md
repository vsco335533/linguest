

# 🧾 Linguist v2 — Updated Dev Changes (With Existing Voice Support)

## 🔁 Key Principle

> ❗ Treat **typed input and speech input the same after transcription**

Once speech → text is done:
👉 Everything flows through the **same L1/L2 logic**

---

# 1️⃣ No Change Needed (Important)

Keep as-is:

* 🎤 Speech-to-text pipeline
* 💬 Chat API (`/chat/stream`)
* 📡 SSE streaming
* 🧠 Analytics engine

👉 Just plug L1/L2 logic **after transcription**

---

# 2️⃣ Input Handling Update (CRITICAL)

## 🔹 Current Flow

```
User speaks → Speech-to-text → message → chat API
```

## 🔹 Updated Flow

```
User speaks/types → text input
→ detect language (L1 vs English)
→ apply language_mode logic
→ send to LLM
```

---

## 🔹 Add Language Detection (Backend)

### File: `chat_service.py`

Use lightweight detection (fast):

```python
from langdetect import detect

def detect_input_language(text):
    try:
        return detect(text)
    except:
        return "unknown"
```

---

## 🔹 Store in Context

```python
input_language = detect_input_language(user_message)
```

Pass into prompt:

```python
Input Language: Telugu / English
```

---

# 3️⃣ Smart AI Behavior (Voice + Chat Unified)

## 🔹 Case 1: User speaks in L1 (Telugu)

AI should:

* Acknowledge in L1
* Convert to English gently
* Ask user to repeat in English

Example:

> “చాలా బాగుంది! In English, you can say: *The fox is clever.* Try saying it 😊”

---

## 🔹 Case 2: User speaks broken English

AI should:

* Correct subtly
* Encourage improvement

Example:

> “Nice try! You can say: *The fox was very clever.*”

---

## 🔹 Case 3: User speaks good English

AI should:

* Continue fully in English
* Increase difficulty

---

# 4️⃣ Prompt Upgrade (FINAL VERSION)

Modify your prompt template:

```python
prompt = f"""
User CEFR Level: {cefr_level}
Language Mode: {language_mode}
Native Language: {native_language}
User Input Language: {input_language}

Rules:

1. If user uses native language:
   - Respond partially in native language
   - Convert key parts into English
   - Encourage English response

2. If user uses mixed language:
   - Prefer English
   - Use native language only for support

3. If user uses English:
   - Respond fully in English
   - Correct mistakes naturally

4. Keep tone friendly and conversational
5. Do NOT translate everything directly
6. Encourage learning, not dependency
"""
```

---

# 5️⃣ Frontend Update (Voice + Chat UX)

## 🔹 Show Input Type (Small but powerful)

When user speaks:

* Show 🎤 icon on message

When typed:

* Show ⌨️ icon

---

## 🔹 Add Real-Time Hint (Optional but high impact)

If user speaks in L1:
Show UI hint:

> 💡 “Try saying this in English”

---

# 6️⃣ Speech-Specific Enhancement (BIG WIN)

## 🔹 Add “Repeat in English” Flow

After AI response:

👉 Show button:

* “🎤 Try in English”

When clicked:

* Auto-enable mic
* Prompt user to repeat

---

# 7️⃣ Analytics Upgrade (Use What You Already Built)

You already track:

* Grammar
* Complexity
* etc.

Now add:

## 🔹 New Metric (lightweight)

```json
"language_usage": {
  "l1_percentage": 70,
  "english_percentage": 30
}
```

---

## 🔹 How to Calculate

* Based on detected language per message
* Store in `MetricDelta`

---

# 8️⃣ Progression Logic Upgrade

Currently:

* CEFR based on quiz

Add:

👉 Language shift trigger

### Example:

If:

* English usage > 60% for last 10 messages

Then:

```python
language_mode = "MIXED" → "L2"
```

---

# 9️⃣ What NOT to Change

❌ Don’t rebuild speech system
❌ Don’t create separate voice pipeline
❌ Don’t duplicate chat logic

👉 Keep **one unified pipeline**

---

# 🔟 Dev Priority (Updated)

### Phase 1 (Immediate)

1. Language mode (DB + API)
2. Prompt update
3. Input language detection
4. AI response logic

---

### Phase 2

5. UI indicators (🎤 / ⌨️)
6. “Try in English” feature
7. Language usage tracking

---

### Phase 3

8. Auto progression (L1 → L2)
9. Gamification

---

# 🧠 Final Outcome

After this change:

* 🎤 Voice becomes **learning tool**, not just input method
* 🧠 AI adapts based on how user speaks
* 🌐 L1 → L2 transition becomes natural
* 📈 Engagement + learning efficiency both increase

---


