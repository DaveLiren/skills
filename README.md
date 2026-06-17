# 🛡️ NVIDIA NIM Rate Limiter — No More 429s

<p align="center">
  <img src="https://img.shields.io/badge/NVIDIA_NIM-free_tier-76b900?style=for-the-badge&logo=nvidia&logoColor=white" alt="NVIDIA NIM"/>
  <img src="https://img.shields.io/badge/Python-3.8+-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/HTTP_429-solved-success?style=for-the-badge" alt="429 solved"/>
  <img src="https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20%7C%20Windows-lightgrey?style=for-the-badge" alt="Platform"/>
</p>

<p align="center">
  <strong>A clean, dependency-light Python wrapper that prevents HTTP 429 bans when using NVIDIA NIM's free tier API.</strong><br/>
  No tokens to count. No guessing limits. Just reliable, automatic rate control.
</p>

---

## 🤔 The Problem

[NVIDIA NIM](https://build.nvidia.com) gives free access to powerful models like `llama-3.3-nemotron-super-49b-v1` via a REST API. But if you make too many requests too quickly, you get:

```
HTTP 429 Too Many Requests
```

The tricky part: **NVIDIA does not publish the exact limits for the free tier.** This causes:

- 🚫 Agents and scripts breaking without warning
- 🔄 Hours wasted guessing retry intervals
- 🔑 API keys getting temporarily suspended

**The key insight** that makes the solution simple:

> NIM free tier limits by **number of HTTP requests** — NOT by token count.
> You don't need to measure input/output text. Just count calls.

---

## ✅ The Solution

A single Python module (`rate_limiter_nim.py`) that wraps every NIM call with automatic rate control:

| Behavior | Value |
|----------|-------|
| Max requests per minute | **20** (sliding window) |
| Minimum interval between calls | **3 seconds** |
| Backoff when HTTP 429 received | **60 seconds** (or `Retry-After` header) |
| Preventive pause threshold | `X-RateLimit-Remaining < 5` → pause 10s |
| Auto-retries on 429 | Up to **5 times** before raising an error |

---

## 🚀 Quick Start

### 1. Install the dependency

```bash
pip install requests
```

### 2. Set your NVIDIA API Key

Get your key at [build.nvidia.com](https://build.nvidia.com), then:

```bash
# Linux / macOS
export NVIDIA_API_KEY="nvapi-your-key-here"

# Windows PowerShell
$env:NVIDIA_API_KEY = "nvapi-your-key-here"

# Windows CMD
set NVIDIA_API_KEY=nvapi-your-key-here
```

### 3. Copy the module to your project

Download `rate_limiter_nim.py` and place it next to your script.

### 4. Use it

```python
from rate_limiter_nim import get_nim_text

# That's it. The rate limiter handles everything automatically.
response = get_nim_text("Explain what a neural network is in 2 sentences.")
print(response)
```

---

## 📦 Files in This Repository

```
📁 rate-limiter-nim/
│
├── rate_limiter_nim.py              ← Main module (the actual solution)
│
├── README.md                        ← This file
│
├── RateLimiter_NIM_Guia_Universal.md  ← Full technical guide (ES/EN)
│
├── SKILL_Hermes_RateLimiterNIM.md   ← Ready-to-install skill for Hermes agent
│
└── SKILL_OpenClaw_RateLimiterNIM.md ← Ready-to-install skill for OpenClaw agent
```

---

## 🧑‍💻 Usage Examples

### Single prompt

```python
from rate_limiter_nim import get_nim_text

text = get_nim_text("What is machine learning? Answer in one sentence.")
print(text)
```

### Full JSON response

```python
from rate_limiter_nim import rate_limited_nim_call

result = rate_limited_nim_call(
    prompt="List 3 benefits of neural networks.",
    model="nvidia/llama-3.3-nemotron-super-49b-v1",
    max_tokens=200,
    temperature=0.5
)
print(result["choices"][0]["message"]["content"])
```

### Processing multiple prompts (batch)

```python
from rate_limiter_nim import get_nim_text

prompts = [
    "What is a transformer? (1 sentence)",
    "What is RLHF? (1 sentence)",
    "What is fine-tuning? (1 sentence)",
    "What is RAG? (1 sentence)",
    "What is a vector database? (1 sentence)",
]

for i, prompt in enumerate(prompts):
    print(f"[{i+1}/{len(prompts)}]", end=" ")
    # No need to add time.sleep() — the rate limiter handles all waits
    print(get_nim_text(prompt, max_tokens=80))
```

> ⚠️ **Do NOT add `time.sleep()` calls** around your requests. The module manages all timing internally. Adding extra sleeps just makes it slower.

---

## 🔧 Configuration

All limits are defined as constants at the top of `rate_limiter_nim.py`. Adjust them if NVIDIA changes their limits:

```python
MAX_REQ_PER_MIN      = 20    # Max requests per 60-second window
MIN_INTERVAL_SEC     = 3     # Seconds to wait between consecutive calls
BACKOFF_ON_429       = 60    # Seconds to wait after a 429 response
RATE_LIMIT_THRESHOLD = 5     # Pause if X-RateLimit-Remaining drops below this
MAX_RETRIES          = 5     # Max auto-retries on 429 before raising an error
DEFAULT_MODEL        = "nvidia/llama-3.3-nemotron-super-49b-v1"
```

---

## 🤖 Agent Integrations

### Hermes

Copy `SKILL_Hermes_RateLimiterNIM.md` to:

```
~/.hermes/skills/mlops-inference/rate-limiter-nim/SKILL.md
```

The skill reads `NVIDIA_API_KEY` from `~/.hermes/.env` automatically.

### OpenClaw

Copy `SKILL_OpenClaw_RateLimiterNIM.md` to:

```
~/.openclaw/skills/rate-limiter-nim/SKILL.md
```

Then verify it loaded:

```bash
openclaw skills list | grep rate-limiter-nim
```

---

## 🐛 Troubleshooting

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `EnvironmentError: NVIDIA_API_KEY is not set` | Key not in environment | Set the env variable (see Quick Start §2) |
| `HTTP 429` persists after retries | Key temporarily suspended by NVIDIA | Wait 5–10 min and try again |
| `ModuleNotFoundError: No module named 'requests'` | Missing dependency | Run `pip install requests` |
| `requests.exceptions.Timeout` | NIM took > 120s to respond | Try lower `max_tokens` or retry later |
| `RuntimeError: 5 consecutive 429 errors` | Sustained rate limit | Wait a few minutes; reduce `MAX_REQ_PER_MIN` to 15 |
| `KeyError: 'choices'` | Unexpected API response format | Print the full `result` dict to see the actual error |

---

## 📊 How It Works (Flow Diagram)

```
[Incoming request]
        │
        ▼
┌───────────────────────────────┐
│ Are there 20+ calls in the   │  YES → Wait until the oldest one expires
│ last 60 seconds?             │        (sliding window)
└──────────────┬────────────────┘
               │ NO
               ▼
┌───────────────────────────────┐
│ Was the last call less than  │  YES → Wait the remaining seconds
│ 3 seconds ago?               │
└──────────────┬────────────────┘
               │ NO
               ▼
        [Send HTTP POST to NIM]
               │
               ▼
┌───────────────────────────────┐
│ Response = HTTP 429?         │  YES → Sleep 60s (or Retry-After), retry
└──────────────┬────────────────┘
               │ NO
               ▼
┌───────────────────────────────┐
│ X-RateLimit-Remaining < 5?  │  YES → Sleep 10s (preventive)
└──────────────┬────────────────┘
               │ NO
               ▼
        [Return response ✅]
```

---

## 📋 Requirements

- Python **3.8+**
- [`requests`](https://pypi.org/project/requests/) library
- A valid NVIDIA NIM API Key (free at [build.nvidia.com](https://build.nvidia.com))

---

## 🌐 Platform Support

| Platform | Supported | Notes |
|----------|-----------|-------|
| Linux | ✅ | Full support |
| macOS | ✅ | Full support |
| Windows (native) | ✅ | Use PowerShell or CMD to set env variables |
| Windows (WSL) | ✅ | Use `export` like Linux |
| Windows (Git Bash) | ✅ | Use `export` like Linux |

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

<p align="center">
  Made with 🧠 + ☕ to stop losing time to HTTP 429 errors.<br/>
  If this helped you, ⭐ the repo!
</p>
