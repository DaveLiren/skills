---
name: rate-limiter-nim
title: "Rate Limiter for NVIDIA NIM — Anti-429"
author: "Iasimov Technologies"
version: "2.0"
description: "Wraps any NVIDIA NIM API call to automatically enforce rate limits and prevent HTTP 429 bans. Reads NVIDIA_API_KEY from ~/.hermes/.env. Works on Linux, macOS, and Windows (WSL). Zero extra configuration needed."
---

## 🎯 Purpose

Prevent HTTP 429 bans from NVIDIA NIM (free tier) by enforcing:

- **Max 20 requests per minute** (sliding window of 60 seconds)
- **Minimum 3 seconds between consecutive requests**
- **60s automatic backoff** when a 429 is received
- **Preventive 10s pause** when `X-RateLimit-Remaining < 5`
- **Auto-retry** up to 5 times on 429 before raising an exception

> **Key insight**: NVIDIA NIM free tier limits by **HTTP requests**, NOT by tokens.
> You only need to count calls — not measure text size.

---

## 📂 Installation (Hermes)

Place this skill in your Hermes skills directory. The `~` symbol expands to your
home directory automatically on **Linux, macOS, and Windows (WSL)**.

```
~/.hermes/skills/mlops-inference/rate-limiter-nim/
├── SKILL.md                       ← this file
├── scripts/
│   └── rate_limiter_nim.py        ← the Python module (see below)
└── references/
    └── requirements.txt           ← pip dependencies
```

Commands to set up (Linux / macOS / WSL):

```bash
# Create directories
mkdir -p ~/.hermes/skills/mlops-inference/rate-limiter-nim/scripts
mkdir -p ~/.hermes/skills/mlops-inference/rate-limiter-nim/references

# Create requirements file
echo "requests" > ~/.hermes/skills/mlops-inference/rate-limiter-nim/references/requirements.txt

# Install dependency
pip install requests
```

---

## 🔑 API Key Setup

This skill reads `NVIDIA_API_KEY` from `~/.hermes/.env` automatically.

Add this line to your `~/.hermes/.env` file:

```bash
NVIDIA_API_KEY=nvapi-your-key-here
```

> Verify it's there:
> ```bash
> grep NVIDIA_API_KEY ~/.hermes/.env
> ```

You can also override the API endpoint (optional):

```bash
# Only needed if you use a custom NIM endpoint
NVIDIA_BASE_URL=https://integrate.api.nvidia.com/v1
```

---

## 🛠️ How to Use

Import and call from any Hermes skill or script:

```python
# Option A: Full JSON response
from rate_limiter_nim import rate_limited_nim_call

result = rate_limited_nim_call("Explain what a transformer is in 2 sentences.")
print(result)

# Option B: Text-only response (recommended for most use cases)
from rate_limiter_nim import get_nim_text

text = get_nim_text("Explain what a transformer is in 2 sentences.")
print(text)
```

---

## 📜 Full Python Module

Save this as `scripts/rate_limiter_nim.py` inside this skill's directory:

```python
#!/usr/bin/env python3
"""
rate_limiter_nim.py — Rate Limiter for NVIDIA NIM (Hermes skill)

Compatible with: Linux, macOS, Windows (WSL).
Reads NVIDIA_API_KEY from ~/.hermes/.env automatically.
Enforces: 20 req/min max, 3s min interval, 60s backoff on HTTP 429.

Usage:
    from rate_limiter_nim import rate_limited_nim_call, get_nim_text

    text = get_nim_text("Your prompt here")
"""

import os
import time
import requests


# ─── Load ~/.hermes/.env (Hermes-specific, cross-platform) ─────────────────
def _load_hermes_env():
    """
    Load environment variables from ~/.hermes/.env if the file exists.
    Uses os.path.expanduser so it works on Linux, macOS, and Windows (WSL).
    """
    env_path = os.path.expanduser("~/.hermes/.env")
    if os.path.exists(env_path):
        with open(env_path) as f:
            for line in f:
                line = line.strip()
                if not line or line.startswith("#"):
                    continue
                key, _, value = line.partition("=")
                if key and value:
                    os.environ.setdefault(key.strip(), value.strip())

_load_hermes_env()


# ─── Configuration ──────────────────────────────────────────────────────────
MAX_REQ_PER_MIN      = 20    # Max requests in any 60-second sliding window
MIN_INTERVAL_SEC     = 3     # Minimum seconds between consecutive calls
BACKOFF_ON_429       = 60    # Seconds to wait after receiving HTTP 429
RATE_LIMIT_THRESHOLD = 5     # Preventive pause when X-RateLimit-Remaining < this
MAX_RETRIES          = 5     # Maximum auto-retries on 429 before raising error

DEFAULT_MODEL = "nvidia/llama-3.3-nemotron-super-49b-v1"

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

    Reads NVIDIA_API_KEY (and optionally NVIDIA_BASE_URL) from ~/.hermes/.env.

    Args:
        prompt:       Text or question to send to the model.
        model:        NIM model ID to use.
        max_tokens:   Max tokens in the response.
        temperature:  Response creativity (0.0 = deterministic, 1.0 = creative).
        _retry_count: Internal retry counter — do NOT pass this manually.

    Returns:
        dict: Full JSON response from the NIM API.

    Raises:
        EnvironmentError: If NVIDIA_API_KEY is not set in ~/.hermes/.env.
        RuntimeError:     If max retries are exhausted due to repeated 429s.
        requests.exceptions.RequestException: On network or other HTTP errors.
    """
    # Step 1: Read API key from environment
    api_key = os.getenv("NVIDIA_API_KEY")
    if not api_key:
        raise EnvironmentError(
            "NVIDIA_API_KEY not found.\n"
            "Add this line to your ~/.hermes/.env file:\n"
            "  NVIDIA_API_KEY=nvapi-your-key-here"
        )

    base_url = os.getenv("NVIDIA_BASE_URL", "https://integrate.api.nvidia.com/v1").rstrip("/")
    url = f"{base_url}/chat/completions"

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

    # Step 4: Enforce minimum 3-second interval between consecutive calls
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

    # Step 7: Raise an exception for other HTTP errors (401, 500, etc.)
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
    Simplified wrapper — returns only the model's text response (not the full JSON).

    Args:
        prompt:   Text or question to send.
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

## ✅ Verification

1. Confirm your API key is set:
   ```bash
   grep NVIDIA_API_KEY ~/.hermes/.env
   ```

2. Install the dependency:
   ```bash
   pip install requests
   ```

3. Run the smoke test:
   ```bash
   cd ~/.hermes/skills/mlops-inference/rate-limiter-nim/scripts
   python3 -c "
   from rate_limiter_nim import get_nim_text
   print(get_nim_text('Reply with just the word: WORKS'))
   "
   ```

   Expected output:
   ```
   WORKS
   ```

---

## ⚠️ Pitfalls & Notes

- **Do NOT add `time.sleep()` calls** outside this module — all waits are handled internally.
- **Restarting Hermes resets `_request_timestamps`** — safe, because the limits are conservative.
- **NIM limits by requests, NOT by tokens** — do not try to count tokens; it's irrelevant.
- **In-memory state**: do not use with `background=true` in Hermes cronjobs (state resets between executions).
- The `DEFAULT_MODEL` can be changed to any valid NIM model ID if needed.

---

## 💡 Pro Tips

- **To process multiple prompts**, just loop over them calling `get_nim_text()` — no manual sleeps needed.
- **To debug**, inspect the internal call window:
  ```python
  import rate_limiter_nim as rl, time
  print(f"Calls in last 60s: {len(rl._request_timestamps)}")
  ```
- **Save large outputs to disk** — never print massive responses to chat.
