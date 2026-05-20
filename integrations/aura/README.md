# AURA trust-check adapter

Opt-in, **read-only** counterparty reputation for agent hosts. One HTTP GET
answers *"can I trust this agent before I delegate work or settle a payment?"*

- **Zero dependencies** — pure Python stdlib. Vendor the `aura/` folder, no `pip install`.
- **Read-only** — the only network call is `GET /check?did=...`. No auth, no API key.
- **No coupling** — does not sign, hold keys, move funds, or touch your wallet.
- **Off by default** — nothing runs until you call it. Disabled = delete the import.

## Enable (opt-in)

It's a gate you call explicitly at a trust boundary — there is no global hook,
no monkey-patching, no background calls. Wrap the action you want to protect:

```python
from aura import before_settle, AuraUntrusted

def settle(counterparty_did: str, amount: float) -> None:
    try:
        before_settle(counterparty_did)        # rejects high_risk + unknown
    except AuraUntrusted as e:
        log.warning("blocked: %s", e)
        return                                  # your policy decides what to do
    pay(counterparty_did, amount)               # your existing logic, untouched
```

Prefer to read the verdict yourself instead of raising?

```python
from aura import aura_verdict

v = aura_verdict(counterparty_did)
print(v.verdict)   # trusted | caution | high_risk | new | unknown
print(v.reason)    # human-readable explanation
print(v.score)     # composite 0..1, or None when there's no history
print(v.ok)        # True for trusted/caution

# v.dimensions tells you *which* axis is weak, not just the aggregate:
if v.dimensions and v.dimensions.get("financial_integrity", 1) < 0.4:
    require_manual_review()
```

## Verdicts

| verdict | meaning | `ok` |
|---|---|---|
| `trusted` | strong on-chain track record (composite ≥ 0.70) | ✅ |
| `caution` | mixed history (0.40–0.70) | ✅ |
| `high_risk` | poor track record (< 0.40) | ❌ |
| `new` | registered identity, no interactions yet | ❌ |
| `unknown` | no track record — or AURA was unreachable | ❌ |

## Policy knobs

```python
# Reject brand-new agents too (strict):
before_settle(did, allow=("trusted", "caution"))

# Treat an *unreachable* AURA as a pass (fail-open). Off by default —
# absence of evidence is not evidence of trust.
before_settle(did, fail_open=True)

# Point at a self-hosted / staging gateway:
before_settle(did, base_url="https://my-aura-mirror.example", timeout=5)
```

`require_trust` is an alias of `before_settle` for non-payment call sites.

## Failure behavior

`aura_verdict()` **never raises on a network or parse error** — it returns an
`unknown` verdict with the reason set. The gate then decides:

- **default (`fail_open=False`)** — `unknown` is rejected → an unreachable AURA
  blocks the action. *Fail-closed.*
- **`fail_open=True`** — `unknown` from an unreachable endpoint is allowed
  through, so AURA can never take your flow down. *Fail-open.*

This keeps the trust signal **purely additive**: if you remove the adapter or
AURA is down, your existing allow/deny logic runs exactly as before.

## Tests

Offline — every call replays a recorded `/check` body, no network:

```bash
python -m pytest aura/tests -q
```

Covers all five verdict classes, the gate's allow-list + `fail_open`, the
unreachable path, and input validation. See `tests/fixtures.py` for the
recorded response shapes.

## Boundary & threats

See [THREAT_MODEL.md](./THREAT_MODEL.md) — what the verdict does and does not
prove, and the failure modes a verifier should account for.

## What's behind the verdict

[AURA Open Protocol](https://auraopenprotocol.org) — W3C DID identity plus 8
on-chain reputation dimensions on Base L2 (`task_completion`, `delivery_speed`,
`output_quality`, `honesty`, `financial_integrity`, `security_compliance`,
`collaboration`, `dispute_history`). Docs: https://dev.auraopenprotocol.org
