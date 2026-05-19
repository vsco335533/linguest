 — Self-Hosted Human-Like Voice (Dev Spec)

## 🎯 Objective

Replace current robotic voice with a **natural, human-like voice system** that:

* Works **without third-party APIs**
* Runs on **GPU machine**
* Supports **Hindi + English mixed speech**
* Adds **pauses, tone, and clarity**

---

# 🚨 Scope

We are NOT just integrating TTS.

We are building:

> **TTS + Prosody Engine (voice modulation layer)**

---

# 1️⃣ Architecture

```text
LLM Response
   ↓
Text Normalization (Hindi script)
   ↓
Prosody Engine (pauses + tone)
   ↓
TTS Model (Coqui)
   ↓
Audio Output (.wav)
   ↓
Frontend Playback
```

---

# 2️⃣ Tech Stack

### TTS Engine

* Use: Coqui TTS
* Runs locally (GPU enabled)

---

### Audio Processing

* Python: `pydub`
* Format: WAV (recommended)

---

# 3️⃣ Backend Implementation

## 🔹 Install

```bash
pip install TTS pydub
```

---

## 🔹 Initialize Model

```python
from TTS.api import TTS

tts = TTS(
    model_name="tts_models/multilingual/multi-dataset/xtts_v2",
    gpu=True
)
```

---

# 4️⃣ Text Processing Layer (MANDATORY)

## 🔹 Step 1: Clean + Normalize Text

* Input comes from LLM
* Ensure Hindi is in **Devanagari script**

---

## 🔹 Step 2: Split Sentences

```python
import re

def split_sentences(text):
    return re.split(r'[.!?]', text)
```

---

## 🔹 Step 3: Add Pauses

```python
def add_pauses(sentences):
    result = []
    for s in sentences:
        if s.strip():
            result.append(s.strip())
            result.append("<break time='300ms'/>")
    return " ".join(result)
```

---

## 🔹 Step 4: Generate SSML

```python
def generate_ssml(text):
    sentences = split_sentences(text)
    paused_text = add_pauses(sentences)

    return f"""
    <speak>
        <prosody rate="90%" pitch="+2%">
            {paused_text}
        </prosody>
    </speak>
    """
```

---

# 5️⃣ Mixed Language Handling (CRITICAL)

## 🔹 Problem

Single-pass TTS fails on Hindi + English mix

---

## 🔹 Solution

Split into chunks:

```python
def split_mixed_text(text):
    return [
        "बहुत अच्छा",
        "The fox is clever",
        "तुम क्या सोचते हो"
    ]
```

---

## 🔹 Generate Audio Per Chunk

```python
def generate_audio_chunks(parts):
    audio_files = []

    for i, part in enumerate(parts):
        file_path = f"chunk_{i}.wav"
        tts.tts_to_file(text=part, file_path=file_path)
        audio_files.append(file_path)

    return audio_files
```

---

# 6️⃣ Audio Stitching

```python
from pydub import AudioSegment

def merge_audio(files):
    final_audio = AudioSegment.empty()

    for f in files:
        segment = AudioSegment.from_wav(f)
        final_audio += segment + AudioSegment.silent(duration=250)

    final_audio.export("final.wav", format="wav")
```

---

# 7️⃣ Voice Modulation Rules

## 🔹 Apply Consistently

| Setting     | Value                  |
| ----------- | ---------------------- |
| Speech Rate | 0.85 – 0.95            |
| Pitch       | +1% to +3%             |
| Pause       | 200–400 ms             |
| Tone        | Friendly / encouraging |

---

## 🔹 Example Input

```text
बहुत अच्छा! The fox बहुत clever है। तुम क्या सोचते हो?
```

---

## 🔹 Expected Output Behavior

* Clear Hindi pronunciation
* English words spoken naturally
* Pauses between sentences
* No robotic speed

---

# 8️⃣ API Endpoint

## 🔹 Create New Endpoint

```http
POST /api/v1/tts/generate
```

---

## 🔹 Request

```json
{
  "text": "बहुत अच्छा! The fox बहुत clever है।"
}
```

---

## 🔹 Response

```json
{
  "audio_url": "/audio/final.wav"
}
```

---

# 9️⃣ Frontend Integration

## 🔹 Add Controls

* 🔊 Play
* ⏸ Pause
* 🔁 Replay

---

## 🔹 Behavior

* Auto-play after AI response (optional)
* Preload audio before playback

---

# 🔟 Performance Optimization

* Cache repeated phrases
* Use async processing for TTS
* Store generated audio temporarily

---

# 1️⃣1️⃣ QA Checklist

### Must Pass

* Hindi sentences clear ✅
* Mixed sentences understandable ✅
* Pauses present ✅
* No robotic tone ✅
* Audio latency < 1s (target) ✅

---

# 1️⃣2️⃣ What NOT to Do

❌ Don’t use browser TTS
❌ Don’t send raw text directly to TTS
❌ Don’t skip sentence splitting
❌ Don’t generate one long audio

---

# 🧠 Final Outcome

After implementation:

* 🎧 Voice sounds natural
* 🧑‍🏫 Feels like human tutor
* 🗣️ Clear Hindi + English mix
* 💰 No third-party dependency

---

# ⚡ Priority

### Phase 1 (Must)

* Coqui TTS setup
* Text processing layer
* Audio generation

### Phase 2

* Prosody tuning
* Voice personalization

model running on run pod 

these are curl commands

curl -X POST https://dwnnwwce5ygx3v-11434.proxy.runpod.net/api/generate \
  -H 'content-type: application/json' \
  -d '{"model":"mistral:7b","prompt":"Say hello in one sentence."}'


curl -X POST https://dwnnwwce5ygx3v-8002.proxy.runpod.net/tts \
  -H 'content-type: application/json' \
  -d '{"text":"Hello, how are you?","language":"en"}'
