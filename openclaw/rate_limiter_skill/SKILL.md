---
name: rate-limiter-nim
description: "Wraps NVIDIA NIM API calls to prevent HTTP 429 bans. Enforces 20 req/min, 3s minimum interval, and 60s backoff on rate limit errors. Reads NVIDIA_API_KEY from the environment. Works on Linux, macOS, and Windows."
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - NVIDIA_API_KEY
      bins:
        - python3
        - pip
    primaryEnv: NVIDIA_API_KEY
---

# Rate Limiter for NVIDIA NIM — Prevent HTTP 429

## Overview

NVIDIA NIM (free tier) blocks requests with **HTTP 429 Too Many Requests** when
you exceed its undocumented rate limits. This skill wraps every NIM call with
automatic rate control so you never get banned.

**Critical fact:** NIM limits by **HTTP requests** — not by tokens. You only need
to count calls and respect time intervals.

---

## Observed Rate Limits (Free Tier)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Max requests per minute | 20 | Main limit — never exceed |
| Min interval between calls | 3 seconds | Always wait at least 3s |
| Backoff on 429 | 60 seconds | Or use `Retry-After` header value |
| Preventive threshold | `X-RateLimit-Remaining < 5` | Pause 10s before hitting 0 |

---

## Installation (OpenClaw)

Place this skill in your OpenClaw skills directory. The `~` symbol expands to
your home directory on **Linux, macOS, and Windows (Git Bash / WSL)**.

```
~/.openclaw/skills/rate-limiter-nim/
├── SKILL.md                       ← this file
├── scripts/
│   └── rate_limiter_nim.py        ← the Python module (see below)
└── requirements.txt               ← pip dependencies
```

Commands to set up (Linux / macOS / Windows Git Bash or WSL):

```bash
# Create directories
mkdir -p ~/.openclaw/skills/rate-limiter-nim/scripts

# Create requirements file
echo "requests" > ~/.openclaw/skills/rate-limiter-nim/requirements.txt

# Install dependency
pip install requests

# Verify the skill is loaded by OpenClaw
openclaw skills list | grep rate-limiter-nim
```

---

## API Key Setup

Set your NVIDIA API Key using **one** of the following methods:

### Option 1 — Shell environment variable (works everywhere)

```bash
# Linux / macOS — add to ~/.bashrc or ~/.zshrc
export NVIDIA_API_KEY="nvapi-your-key-here"

# Windows PowerShell
$env:NVIDIA_API_KEY = "nvapi-your-key-here"

# Windows CMD
set NVIDIA_API_KEY=nvapi-your-key-here
```

### Option 2 — Workspace `.env` file

Create a `.env` file in your project root:
```bash
NVIDIA_API_KEY=nvapi-your-key-here
```

### Option 3 — OpenClaw `config.yaml` (if your version supports it)

```yaml
environment:
  NVIDIA_API_KEY: "nvapi-your-key-here"
```

> Verify the key is accessible before running: `echo $NVIDIA_API_KEY`

---

## How to Use

When asked to call NVIDIA NIM, use this pattern instead of calling `requests.post` directly:

```python
import sys, os
sys.path.insert(0, os.path.expanduser("~/.openclaw/skills/rate-limiter-nim/scripts"))

# Option A: Full JSON response
from rate_limiter_nim import rate_limited_nim_call
result = rate_limited_nim_call("Explain what a neural network is in 2 sentences.")
print(result)

# Option B: Text only (recommended for most use cases)
from rate_limiter_nim import get_nim_text
text = get_nim_text("Explain what a neural network is in 2 sentences.")
print(text)
```

---

## Full Python Module

Save this as `scripts/rate_limiter_nim.py` inside the skill's directory:

```python
#!/usr/bin/env python3
"""
rate_limiter_nim.py — Rate Limiter for NVIDIA NIM (OpenClaw skill)

Compatible with: Linux, macOS, Windows (Git Bash, WSL, PowerShell).
Reads NVIDIA_API_KEY from the environment (export, .env, or shell variable).
Enforces: 20 req/min max, 3s min interval, 60s backoff on HTTP 429.

Usage:
    from rate_limiter_nim import rate_limited_nim_call, get_nim_text

    text = get_nim_text("Your prompt here")
"""

import os
import time
import requests


# ─── Configuration ──────────────────────────────────────────────────────────
MAX_REQ_PER_MIN      = 20    # Max requests in any 60-second sliding window
MIN_INTERVAL_SEC     = 3     # Minimum seconds between consecutive calls
BACKOFF_ON_429       = 60    # Seconds to wait after receiving HTTP 429
RATE_LIMIT_THRESHOLD = 5     # Preventive pause when X-RateLimit-Remaining < this
MAX_RETRIES          = 5     # Maximum auto-retries on 429 before raising error

DEFAULT_MODEL = "nvidia/llama-3.3-nemotron-super-49b-v1"
NIM_BASE_URL  = os.getenv("NVIDIA_BASE_URL", "https://integrate.api.nvidia.com/v1").rstrip("/")

# ─── Internal State ─────────────────────────────────────────────────────────
_request_timestamps = []     # Unix timestamps of recent API calls


# ─── Main Function ──────────────────────────────────────────────────────────
def rate_limited_nim_call(
    prompt: str,
    model: str = DEFAULT_MODEL,
    max_tokens: int = 1024,
    temperature: float = 0.7,
    _retry_count: int = 0
) -> dict:
    """
    Send a prompt to NVIDIA NIM with automatic rate limiting.

    Reads NVIDIA_API_KEY from the environment.
    Optionally reads NVIDIA_BASE_URL to override the default endpoint.

    Args:
        prompt:       Text or question to send to the model.
        model:        NIM model ID to use.
        max_tokens:   Max tokens in the response.
        temperature:  Response creativity (0.0 = deterministic, 1.0 = creative).
        _retry_count: Internal retry counter — do NOT pass this manually.

    Returns:
        dict: Full JSON response from the NIM API.

    Raises:
        EnvironmentError: If NVIDIA_API_KEY is not set in the environment.
        RuntimeError:     If max retries are exhausted due to repeated 429 errors.
        requests.exceptions.RequestException: On network or other HTTP errors.
    """
    # Step 1: Read API key from environment
    api_key = os.getenv("NVIDIA_API_KEY")
    if not api_key:
        raise EnvironmentError(
            "NVIDIA_API_KEY is not set in the environment.\n"
            "Set it with:\n"
            "  Linux/macOS: export NVIDIA_API_KEY='nvapi-your-key-here'\n"
            "  Windows PowerShell: $env:NVIDIA_API_KEY = 'nvapi-your-key-here'\n"
            "  Windows CMD: set NVIDIA_API_KEY=nvapi-your-key-here"
        )

    url = f"{NIM_BASE_URL}/chat/completions"

    # Step 2: Remove timestamps older than 60 seconds (sliding window)
    now = time.time()
    _request_timestamps[:] = [t for t in _request_timestamps if now - t < 60]

    # Step 3: Enforce max 20 requests per minute
    if len(_request_timestamps) >= MAX_REQ_PER_MIN:
        wait_time = 60 - (now - _request_timestamps[0]) + 0.5  # +0.5s safety margin
        if wait_time > 0:
            print(f"[RATE LIMIT] 20 req/min reached. Waiting {wait_time:.1f}s...")
            time.sleep(wait_time)
        now = time.time()
        _request_timestamps[:] = [t for t in _request_timestamps if now - t < 60]

    # Step 4: Enforce minimum 3-second interval between calls
    if _request_timestamps:
        elapsed = time.time() - _request_timestamps[-1]
        if elapsed < MIN_INTERVAL_SEC:
            wait_time = MIN_INTERVAL_SEC - elapsed
            print(f"[RATE LIMIT] Enforcing 3s interval. Waiting {wait_time:.1f}s...")
            time.sleep(wait_time)

    # Step 5: Send the HTTP POST request to NIM
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    payload = {
        "model": model,
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": max_tokens,
        "temperature": temperature
    }

    response = requests.post(url, headers=headers, json=payload, timeout=120)
    _request_timestamps.append(time.time())  # Record this call

    # Step 6: Handle HTTP 429 — wait and retry automatically
    if response.status_code == 429:
        if _retry_count >= MAX_RETRIES:
            raise RuntimeError(
                f"[429] {MAX_RETRIES} consecutive rate limit errors received. "
                "Wait a few minutes and try again."
            )
        retry_after = int(response.headers.get("Retry-After", BACKOFF_ON_429))
        print(f"[429] Rate limit hit. Waiting {retry_after}s "
              f"(retry {_retry_count + 1}/{MAX_RETRIES})...")
        time.sleep(retry_after)
        return rate_limited_nim_call(
            prompt=prompt, model=model,
            max_tokens=max_tokens, temperature=temperature,
            _retry_count=_retry_count + 1
        )

    # Step 7: Raise an exception for other HTTP errors (401, 404, 500, etc.)
    response.raise_for_status()

    # Step 8: Preventive pause if the remaining quota is running low
    remaining = response.headers.get("X-RateLimit-Remaining")
    if remaining is not None:
        try:
            if int(remaining) < RATE_LIMIT_THRESHOLD:
                print(f"[RATE LIMIT] Only {remaining} requests left. "
                      "Pausing 10s as a precaution...")
                time.sleep(10)
        except ValueError:
            pass

    return response.json()


def get_nim_text(prompt: str, **kwargs) -> str:
    """
    Simplified wrapper — returns only the model's text response.

    Args:
        prompt:   Text or question to send to the model.
        **kwargs: Additional arguments passed to rate_limited_nim_call.

    Returns:
        str: The model's text response.
    """
    result = rate_limited_nim_call(prompt, **kwargs)
    try:
        return result["choices"][0]["message"]["content"]
    except (KeyError, IndexError):
        return str(result)
```

---

## Smoke Test

Run this to verify everything works before using in a real project:

```bash
cd ~/.openclaw/skills/rate-limiter-nim/scripts
python3 -c "
from rate_limiter_nim import get_nim_text
print('Testing NIM connection...')
result = get_nim_text('Reply with just the word: WORKS')
print(f'Result: {result}')
"
```

Expected output:
```
Testing NIM connection...
Result: WORKS
```

---

## Batch Usage (Multiple Prompts)

```python
import sys, os
sys.path.insert(0, os.path.expanduser("~/.openclaw/skills/rate-limiter-nim/scripts"))
from rate_limiter_nim import get_nim_text

prompts = [
    "What is a neural network? (1 sentence)",
    "What is reinforcement learning? (1 sentence)",
    "What is a transformer model? (1 sentence)",
]

for i, prompt in enumerate(prompts):
    print(f"[{i+1}/{len(prompts)}] {prompt[:50]}...")
    # The rate limiter handles all waits automatically — no time.sleep() needed
    text = get_nim_text(prompt, max_tokens=80)
    print(f"  → {text}\n")
```

> **Important**: Do NOT add `time.sleep()` outside this module. All timing is managed internally.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `EnvironmentError: NVIDIA_API_KEY is not set` | Key missing from environment | Set `export NVIDIA_API_KEY="nvapi-..."` or check your `.env` |
| `HTTP 429` keeps repeating after retries | API key may be temporarily suspended | Wait 5-10 min, then retry |
| `ModuleNotFoundError: No module named 'requests'` | Library not installed | Run `pip install requests` |
| `requests.exceptions.Timeout` | NIM took > 120 seconds | Try shorter `max_tokens` or retry later |
| `KeyError: 'choices'` in `get_nim_text` | Unexpected API response format | Print the raw `result` dict to debug |
| `RuntimeError: 5 consecutive 429 errors` | Sustained rate limit | Wait several minutes; reduce `MAX_REQ_PER_MIN` to 15 |
| Skill not found | Wrong directory or missing SKILL.md | Run `openclaw skills list` to verify |

---

## Rules for Any Agent Using This Skill

When this skill is active, always follow these rules:

1. **NEVER** call `requests.post` to NIM directly — always use `rate_limited_nim_call()`.
2. **NEVER** add manual `time.sleep()` calls around NIM requests — the module handles it.
3. **ALWAYS** verify `NVIDIA_API_KEY` is set before starting a batch job.
4. **LIMIT** retries to `MAX_RETRIES` (default: 5). Never loop infinitely on errors.
5. **Save large outputs to disk** — avoid printing massive responses to chat.
