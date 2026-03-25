---
title: "OpenAIにAPIを送るだけで個人情報保護法違反？LLM開発者が知るべきAPPIリスクと対策"
emoji: "⚖️"
type: "tech"
topics: ["python", "llm", "security", "privacy", "openai"]
published: true
---

## あなたのLLM APIコールは個人情報保護法に違反しているかもしれません

You might be violating Japan's privacy law every time you call an LLM API.

When your application sends a prompt to OpenAI, Anthropic, or Google, the entire prompt is transmitted in plaintext. If that prompt contains a customer's name, email, マイナンバー, or phone number, you've just transferred personal information to a foreign company's servers.

あなたのアプリケーションがOpenAI、Anthropic、GoogleにプロンプトをAPIで送信する際、プロンプト全体が平文で送信されます。そのプロンプトに顧客の名前、メール、マイナンバー、電話番号が含まれていれば、外国企業のサーバーに個人情報を転送したことになります。

## APPIが要求すること

Japan's Act on the Protection of Personal Information (APPI) applies to any organization handling personal information of individuals in Japan — regardless of where the organization is based.

Key requirements relevant to LLM API usage:

**越境データ移転 (Cross-border transfers):** APPI Article 28 requires prior consent from data subjects before transferring personal data to third parties in foreign countries, unless the destination country has an adequacy designation or equivalent safeguards are in place.

**利用目的の特定 (Purpose limitation):** Personal information must be used within the scope of the specified purpose. If you collected customer data for "order processing," sending it to an LLM for "summarization" may exceed the original purpose.

**安全管理措置 (Security measures):** Organizations must implement organizational, personnel, physical, and technical measures to protect personal data. Sending unprotected PII in API calls fails this requirement.

**2025-2026改正:** The latest APPI amendments introduce administrative surcharges for serious violations, expanding enforcement beyond the traditional guidance-based approach.

## 具体的なリスク (The Real Risk)

Consider this common pattern:

```python
import openai

response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": "田中太郎様（マイナンバー: 123456789012）の申請書を要約してください"
    }]
)
```

This single API call:

1. Sends マイナンバー to OpenAI's servers (likely in the US)
2. Creates a cross-border transfer without consent
3. May exceed the original purpose of data collection
4. Stores PII in OpenAI's logs

マイナンバー is classified as "特定個人情報" (specific personal information) under the My Number Act, which imposes even stricter handling requirements than standard APPI provisions.

## 解決策: プロンプトからPIIを除去する

The solution is to sanitize PII before it leaves your infrastructure. CloakLLM is open-source middleware that does exactly this:

```python
from cloakllm import Shield, ShieldConfig

shield = Shield(ShieldConfig(locale="ja"))

# Before
prompt = "田中太郎様（マイナンバー: 123456789012）の申請書を要約してください"

# Sanitize — PII replaced with reversible tokens
sanitized, token_map = shield.sanitize(prompt)
# → "[PERSON_0]様（マイナンバー: [MY_NUMBER_JP_0]）の申請書を要約してください"

# Send sanitized prompt to LLM — no PII leaves your infrastructure
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": sanitized}]
)

# Restore originals in the response
restored = shield.desanitize(response.choices[0].message.content, token_map)
```

The LLM never sees real data. It receives `[PERSON_0]` and `[MY_NUMBER_JP_0]` — enough context to understand the prompt, but no actual PII.

## ワンラインで統合 (One-Line Integration)

Even simpler — wrap your OpenAI client directly:

```python
from cloakllm import enable_openai
from openai import OpenAI

client = OpenAI()
enable_openai(client)  # done — all calls are now protected

# Use OpenAI normally — CloakLLM works transparently
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "田中太郎のマイナンバーは123456789012です"}]
)
# Provider never saw the real マイナンバー
# Response has original values restored automatically
```

Or with LiteLLM (100+ providers):

```python
import cloakllm
cloakllm.enable()  # all LiteLLM calls are now protected
```

## 日本語対応のPII検出 (Japanese-Specific Detection)

v0.4.0 adds locale-aware detection for Japan:

| カテゴリ | 検出内容 | 例 |
|---------|---------|-----|
| `MY_NUMBER_JP` | マイナンバー（12桁） | 123456789012 |
| `PHONE_JP` | 日本の電話番号 | 090-1234-5678, 03-1234-5678 |
| `PASSPORT_JP` | パスポート番号 | TK1234567 |
| `PERSON` | 人名 | 田中太郎 (spaCy NER) |
| `EMAIL` | メールアドレス | tanaka@example.co.jp |

`locale="ja"` を設定すると、spaCyモデル `ja_core_news_sm` が自動的に選択されます。

## 改ざん防止の監査ログ (Tamper-Evident Audit Logs)

Every sanitize/desanitize operation is logged to hash-chained JSONL files:

```json
{
  "seq": 1,
  "event_type": "sanitize",
  "entity_count": 2,
  "categories": {"PERSON": 1, "MY_NUMBER_JP": 1},
  "prompt_hash": "sha256:9f86d0...",
  "prev_hash": "sha256:000000...",
  "entry_hash": "sha256:b5e8f3..."
}
```

No PII is stored in the logs. Each entry links to the previous via SHA-256. Modify any entry and the chain breaks — providing tamper-evident proof for auditors and regulators.

```bash
python -m cloakllm verify ./cloakllm_audit/
# Audit chain integrity verified — no tampering detected.
```

## 暗号学的証明 (Cryptographic Attestation)

v0.3.2 added Ed25519-signed sanitization certificates — cryptographic proof that PII was removed before inference:

```python
from cloakllm import DeploymentKeyPair

keypair = DeploymentKeyPair.generate()
shield = Shield(ShieldConfig(locale="ja", attestation_key=keypair))

sanitized, token_map = shield.sanitize("田中太郎のマイナンバーは123456789012です")
cert = token_map.certificate
assert cert.verify(keypair.public_key)  # 暗号学的証明
```

## まとめ (Summary)

| リスク | CloakLLMの対策 |
|-------|---------------|
| PIIがLLMプロバイダーに送信される | プロンプトからPIIを除去してからAPIに送信 |
| 越境データ移転の同意なし | PIIがインフラを離れない |
| 監査証跡がない | SHA-256ハッシュチェーンの監査ログ |
| 改ざんの検出が不可能 | Ed25519署名付き証明書 |

## インストール

```bash
pip install cloakllm==0.4.0
npm install cloakllm@0.4.0
```

- GitHub: https://github.com/cloakllm/CloakLLM
- ドキュメント: https://cloakllm.dev
- 527テスト、MIT ライセンス、13ロケール対応
