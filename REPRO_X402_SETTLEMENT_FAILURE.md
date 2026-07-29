# Reproduction — Local Token Consumption After Failed x402 Settlement

*Recorded 2026-07-29. AP2 reference implementation at commit `e1ea56d` (2026-04-29).*

This document records one reproduced observation about the x402 sample flow. It is not a security report, and it makes no claim about production deployments, the AP2 protocol, or loss of funds. What it establishes is bounded and stated at the end.

---

## Environment

| | |
|---|---|
| Commit | `e1ea56d` |
| Scenario | `code/samples/python/scenarios/a2a/human-not-present/x402` |
| Flow | `FLOW=x402` |
| On-chain broadcast | `BROADCAST_ON_CHAIN=FALSE` (default) |
| Model | `gemini-3.1-flash-lite-preview`, free tier |
| Platform | Windows 11, Python 3.14, `uv` 0.11.32, Node 24.18 |

Services were started individually rather than through `run.sh`, because the script's terminating `wait` does not hold background processes under Git Bash on Windows. All roles were given a shared `TEMP_DB_DIR` so that the merchant trigger, the PSP and the agent read and write the same state directory; without this they use `.temp-db` relative to their own working directory and do not observe each other.

---

## Baseline — settlement reachable

With all four services running, the delegated-purchase flow completed. The transaction chain reported by the client was `check_product → cart → checkout → complete`, with `issue_payment_credential (verify + issue)`, and the interface displayed **Purchase Complete**, order `91f5df62-6add-446d-ac95-60ced132dcff`, charged `$199.00`.

The merchant token store after this run contained `"used": true` and the allocated `order_id`.

This establishes what success looks like in this configuration.

---

## Test — settlement unreachable

1. `ap2_token_store.json` was deleted so the run started from a clean store.
2. The x402 PSP process listening on `localhost:8084` was stopped. All other services remained running.
3. The flow was repeated from the beginning: product query, budget authorization of $200, mandate approval, and the price-drop trigger setting the item to $199 with stock 10.

The agent proceeded autonomously through `assemble_cart`, `create_checkout`, `create_payment_presentation`, `issue_payment_credential`, `create_checkout_presentation`, and `complete_checkout`.

### Result

`complete_checkout` returned:

```
Error
PSP_call_failed
All connection attempts failed
```

No payment receipt was produced. No transaction hash was returned. The interface showed an error rather than a completed purchase.

The merchant token store nevertheless contained:

```json
{
  "x402_tok_4703483ae1954bb5a4e97a4fa62deed7": {
    "used": true,
    "order_id": "cf8e19d1-1631-4abd-9035-eee632f2a3f3",
    "expires_at": 1785351577,
    "currency": "USD",
    "checkout_jwt_hash": "14UvHy8FJkNG98TQ…",
    "open_checkout_hash": "QjZheS5cty8Q8mfz…",
    "bundled_payload": { "eip_3009_payload": { "…": "signed authorization, truncated" } }
  }
}
```

The payment token remains marked `used: true` in the merchant's local token store, and an order identifier remains allocated, although no settlement occurred.

---

## Correspondence with the source

The ordering in `code/samples/python/src/roles/merchant_agent_mcp/server.py` matches the observation:

| Line | Action |
|---|---|
| 923 | `token_data['used'] = True` |
| 924–930 | `order_id` allocated, `amount_charged` and `currency` recorded |
| 946 | `_save_token_store(token_store)` — state persisted |
| 984 | `POST` to the x402 PSP at `/settle-payment` |
| 1000 | `return {'error': 'PSP_call_failed', …}` on exception |

Between the persist at 946 and the failure return at 1000 there is no reassignment of `used`, no deletion from the store, and no compensating write. The card flow beginning at line 1013 has the same ordering and raises `ValueError` on processor error, also without restoration.

`complete_checkout` begins by refusing a token already marked used, returning `token_already_used`.

---

## What this establishes

At commit `e1ea56d`, in the x402 sample configuration described above, failure of the settlement call leaves the merchant's local state recording token consumption and order allocation although settlement did not occur, and the reviewed failure branch contains no restoration of that state.

## What this does not establish

**Not loss of funds.** No transfer was attempted or completed.

**Not on-chain consumption.** A signed EIP-3009 authorization was produced and stored, but broadcast was disabled and no transaction was submitted. The authorization nonce is not shown to be consumed on-chain, and the reproduction says nothing about what would happen if it were.

**Not a blocked retry in practice.** `complete_checkout` refuses a token already marked used. Whether a retry would present the same token or obtain a new payment credential was not tested.

**Not a protocol defect.** This concerns the sample implementation at one commit, in one configuration, with the settlement endpoint deliberately unavailable. Nothing here describes AP2 as specified, or any deployment other than the one run.

**Not evidence from `amount_charged`.** That field records `0` in this run, but it also records `0` in the successful baseline run. It is an anomaly independent of the failure and is not offered as part of the result.

---

## Failure conditions tested and not tested

The settlement call failed at connection time: the endpoint was not listening. This is the cleanest of the three possible outcomes, and the only one reproduced here.

Two others were not tested and would behave differently:

- settlement succeeds and the response is lost, leaving the local state consistent with an outcome the merchant cannot confirm;
- settlement fails after partial processing on the PSP side.

Neither is addressed by this reproduction.
