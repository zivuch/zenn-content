---
title: "LLMのプロンプトからPIIを自動検出・保護するOSSミドルウェア（マイナンバー対応）"
emoji: "🛡️"
type: "tech"
topics: ["python", "javascript", "llm", "security", "openai"]
published: true
---

## The Problem

Every prompt you send to an LLM provider — OpenAI, Anthropic, Google — is visible in plaintext. Customer names, email addresses, and national IDs end up in provider logs.

If your application handles Japanese user data, that includes マイナンバー (My Number), Japanese phone numbers, and Japanese names.

## CloakLLM

CloakLLM is open-source middleware that detects PII, replaces it with reversible tokens, and restores originals in the response. One line to integrate.

```bash
pip install cloakllm
```

```python
from cloakllm import Shield, ShieldConfig

shield = Shield(ShieldConfig(locale="ja"))

text = "田中太郎のマイナンバーは123456789012です。電話は090-1234-5678。"
sanitized, token_map = shield.sanitize(text)
# → "[PERSON_0]のマイナンバーは[MY_NUMBER_JP_0]です。電話は[PHONE_JP_0]。"

# After LLM response, restore originals
restored = shield.desanitize(llm_response, token_map)
```

## Japanese-Specific Detection

v0.4.0 adds locale-aware detection for Japan:

| Category | Pattern | Example |
|----------|---------|---------|
| `MY_NUMBER_JP` | 12-digit individual number | 123456789012 |
| `PHONE_JP` | Japanese mobile/landline | 090-1234-5678, 03-1234-5678 |
| `PASSPORT_JP` | Japanese passport | TK1234567 |

The locale setting also selects the appropriate spaCy model (`ja_core_news_sm`) automatically and provides Japanese-specific hints to the optional Ollama LLM detection pass.

## How It Works

3-pass detection pipeline:

1. **Regex** — structured patterns (emails, credit cards, My Number, phone numbers)
2. **spaCy NER** — names, organizations, locations (model auto-selected per locale)
3. **Ollama LLM** (opt-in) — context-dependent PII (addresses, medical terms)

Every operation is logged to a tamper-evident hash-chained audit trail (SHA-256). Designed for compliance requirements.

## Integration

Works as middleware for existing LLM SDKs:

```python
# OpenAI SDK
from cloakllm import enable_openai
from openai import OpenAI

client = OpenAI()
enable_openai(client)  # all calls now protected
```

```python
# LiteLLM (100+ providers)
import cloakllm
cloakllm.enable()  # all calls now protected
```

```javascript
// Node.js — OpenAI SDK
const cloakllm = require('cloakllm');
cloakllm.enable(client);  // all calls now protected
```

Also available as an MCP server for Claude Desktop.

## Cryptographic Attestation (v0.3.2+)

Every sanitize() call can produce an Ed25519-signed certificate — cryptographic proof that PII was removed before inference. Batch operations use Merkle trees.

```python
from cloakllm import Shield, ShieldConfig, DeploymentKeyPair

keypair = DeploymentKeyPair.generate()
shield = Shield(ShieldConfig(locale="ja", attestation_key=keypair))

sanitized, token_map = shield.sanitize("田中太郎のマイナンバーは123456789012です。")
cert = token_map.certificate
assert cert.verify(keypair.public_key)  # cryptographic proof
```

## Supported Locales

v0.4.0 supports 13 locales: English, Japanese, Chinese, Korean, German, French, Spanish, Portuguese (BR), Russian, Polish, Dutch, Hindi, and Hebrew. 36 locale-specific patterns total.

## Numbers

- 527 tests (260 Python, 234 JS, 33 MCP)
- ~2,300 downloads/week
- MIT license
- Zero JS runtime dependencies

## Links

- GitHub: https://github.com/cloakllm/CloakLLM
- Docs: https://cloakllm.dev
- PyPI: `pip install cloakllm==0.4.0`
- npm: `npm install cloakllm@0.4.0`

