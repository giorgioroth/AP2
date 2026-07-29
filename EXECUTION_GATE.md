# Execution Gate — Research Notes

*An experimental fork of the AP2 reference implementation. Not affiliated with Google. Not part of the official AP2 project.*

This document records what was observed in the AP2 reference implementation, what those observations appear to mean architecturally, and what this fork proposes to test. It is not a critique of AP2. The observations concern where an existing constraint-evaluation mechanism becomes binding, not whether that mechanism exists.

**Scope of verification.** Code observations are from commit `e1ea56d` (2026-04-29). Read: `code/sdk/python/ap2/sdk/constraints.py`, `payment_mandate_chain.py`, `checkout_mandate_chain.py`; the `shopping_agent_v2` mandate tools; and the counterparty roles under `code/samples/python/src/roles/` in both their ADK and MCP forms. The remainder of `code/sdk/` was searched but not read in full. No test runs were performed. Specification observations are from [ap2-protocol.org/specification](https://ap2-protocol.org/specification/), read in full. Nothing below rests on a summary. Where a claim is an inference rather than a reading, it is marked as one.

An earlier draft of this document stated that AP2 counterparties verify credentials without re-evaluating commercial constraints. That was wrong. It was written after reading the ADK role files and before reading the MCP ones, and it generalized from half the roles to all of them. The correction is in Section 1, and it narrows the thesis of this fork rather than supporting it.

---

## 1. Observed Architecture

Facts, each locatable in the repository.

**Constraint evaluation is real SDK logic.** `code/sdk/python/ap2/sdk/constraints.py` defines evaluators for `AmountRange`, `AllowedPayees`, `AllowedPaymentInstruments`, `AllowedPisps`, `Budget`, `AgentRecurrence`, `ExecutionDate` and `PaymentReference`, together with checkout evaluators for `AllowedMerchants` and `LineItems`. `check_payment_constraints` and `check_checkout_constraints` return lists of violation strings; an empty list means no violation was found. `PaymentMandateChain.verify()` and `CheckoutMandateChain.verify()` call them and return the resulting list. The SDK evaluates and reports. It does not decide whether execution continues; each caller does.

**Counterparties call that logic and refuse on violations.** The `shopping_agent_v2` flow connects to MCP server roles. Four of them convert a non-empty violation list into a refusal, unconditionally in the paths read:

- `merchant_agent_mcp/server.py` calls `check_checkout_constraints` (line 745) and returns `constraint_violation` (line 749) before creating the checkout JWT; at completion it calls `CheckoutMandateChain.verify()` and returns `verification_failed`.
- `credentials_provider_mcp/server.py` calls `PaymentMandateChain.verify()` and does not issue a payment token when violations are returned.
- `x402_credentials_provider_mcp/server.py` follows the same pattern before generating the EIP-3009 authorization bundle.
- `merchant_payment_processor_mcp/server.py` calls `PaymentMandateChain.verify()` and returns `chain_verification_failed` before creating a payment receipt.

**The missing-key condition is handled inconsistently across components.** All of these roles need the agent-provider public key to verify a mandate chain. They do not agree on what to do when it is unavailable.

`merchant_agent_mcp/server.py` refuses twice: `create_checkout` returns `no_public_key`, and `complete_checkout` returns `agent_provider_key_missing` with the message that the mandate chain cannot be verified. `merchant_payment_processor_mcp/server.py` follows the same pattern.

`x402_psp_mcp/server.py` does not. Its entire verification step — `MandateClient().verify()`, `PaymentMandateChain.parse()` and `parsed_chain.verify()` — sits inside `if agent_provider_pub:`. If `AGENT_PROVIDER_PUB_PATH` does not exist, or reading it raises `OSError`, `ValueError` or `JSONDecodeError`, the variable stays `None`, a warning is logged, and execution proceeds to binding, `ecrecover`, routing and the optional on-chain broadcast. There is no `else` branch returning an error, and no error return anywhere in the file for this condition. **The signature check and the constraint evaluation are skipped together**, not the signature check alone.

In the x402 flow, `complete_checkout` in `merchant_agent_mcp` calls this component at `/settle-payment` after having refused to proceed itself under the same condition. The component that refuses when it cannot verify calls the component that broadcasts when it cannot verify.

Whether that branch is reachable in a provisioned deployment has not been tested here. It is reported as an implementation observation and an internal inconsistency, not as a demonstrated vulnerability.

**Agent-side, the same verdict is advisory.** `check_constraints_against_mandate`, in `shopping_agent_v2/shopping_agent/mandate_tools.py`, resolves the open mandates from session state, builds a candidate `PaymentMandate`, calls `check_payment_constraints`, and returns `meets_constraints`, `violations`, the checked price and availability, and extracted constraints such as `price_cap` and `line_items`. The prompts instruct the model not to proceed unless `available` and `meets_constraints` are both true. That obligation lives in the prompt and in tool ordering. It is not encoded in the code: `create_checkout_presentation` (70 lines) and `create_payment_presentation` (128 lines) are separately exposed tools, and neither contains any call to the constraint check before constructing and signing a mandate.

**The specification contains both boundaries.** Section 6 sets out dispute resolution, including *Mispick, Unapproved by User*, where a signed Intent Mandate is compared against transaction details after a violation. Section 7.4 states the protocol does not aim to prescribe risk or fraud handling. Yet the sample implementation contains preventive refusals at several counterparties. Evidence after the fact and refusal before it are both present in the reviewed artifacts, and neither should be used to erase the other.

**The specification names this area as open.** Section 9, *A Call for Ecosystem Collaboration*, lists Delegated Authorization among the problems left to the ecosystem, with a stated long-term goal of more granular, time-bound access controls for agents.

---

## 2. Architectural Observation

Prevention exists in AP2. Four counterparty components refuse violating mandates before issuing tokens, completing checkout, or creating receipts. Any claim that AP2 supplies evidence without enforcement is refuted by its own samples.

What remains is narrower, and it concerns binding rather than existence:

> The same SDK evaluator runs on both sides of the boundary. At the counterparties it is a code-level precondition: violations produce a refusal on the return path. At the agent it is a prompt-level instruction: the tools that construct and sign mandates contain no call to it, so an agent that skips the check can still produce a well-formed mandate and send it.

This fork tests whether making the agent-side evaluation a code-level precondition changes any outcome that matters.

It may not. Downstream refusal already prevents settlement in the four fail-closed paths, and duplicating a decision earlier may buy only latency and maintenance cost. It may matter if earlier refusal avoids constructing and signing an invalid mandate at all, reduces what is disclosed to counterparties, contains an agent failure before it crosses a trust boundary, or covers a path where downstream verification is absent or skipped. Those are hypotheses.

*The following is an inference, not a reading.* Final settlement raises the cost of any gap in enforcement, because no adjudicator can reverse the transfer afterwards. The inconsistency described above is the one place in the reviewed code where a settlement component can proceed with neither signature nor constraint verification, on a rail without chargeback, having been called by a component that refuses under the same condition. That does not establish that redundant enforcement near the action source is necessary; a correctly provisioned deployment may never reach it. It does show that the reliability of downstream refusal is a property of each component's configuration rather than of the protocol, which is a weaker claim and the only one the code supports.

---

## 3. Experimental Integration Point

The integration point requires no change to the protocol, the message formats, or the delegation model.

```
Shopping Agent
      │
      ├── check_constraints_against_mandate()    →  verdict
      │
      ▼
  Execution Gate                                 →  ALLOW / DENY
      │
      ▼
  Checkout / Payment Mandate construction
```

The gate sits between constraint evaluation and mandate construction. Its input is the verdict the SDK already produces; its difference is that the mandate-construction tools cannot be reached around it. The agent-side check stops being a prompt-governed instruction and becomes a code-level precondition.

Everything downstream is unchanged: the same mandates, signatures, delegation chain, and counterparty verification. A merchant or processor receiving a mandate produced through the gate cannot tell the difference, which is the point — interoperability is preserved by construction.

**Proprietary components.** The gate demonstrated here is deliberately minimal and exists to show where such a boundary attaches. The production execution kernel, the admissibility model and the enforcement mechanisms belong to the Regen Engine project and are not included in this repository.

**Modified files.** The AP2 code is licensed under Apache 2.0. Files modified by this fork will carry a prominent notice that they were changed, as Section 4(b) of that licence requires. This fork will additionally record what changed and when, which the licence does not require but provenance does. Unmodified files remain untouched.

---

## 4. Open Research Questions

None of these is answered here. They are what the experiment is for.

**Does earlier binding change any outcome that matters?** If every action the gate blocks is one a fail-closed counterparty would have refused anyway, the gate is overhead. Demonstrating a difference is the first thing that must be shown, and it has not been shown.

**Is the agent the right place for it?** A gate inside the agent's own process is reachable by the agent's own failure modes. A gate outside costs a boundary crossing per action. The samples already evaluate constraints in both places; this fork binds the earlier one, which may still be the wrong placement.

**Where does this responsibility properly belong?** Protocol, wallet, agent framework, application runtime, middleware, counterparty, or some combination. The reviewed samples already distribute enforcement across several of these. This fork demonstrates one additional binding point and does not argue that it is the correct one.

**Does irreversibility actually change the requirement?** The inference in Section 2 is that it raises the cost of a gap. It could be wrong. If downstream checks are reliably provisioned and compensating transactions are cheap enough in practice, the earlier gate may change nothing worth having, and this fork explores a problem that does not need solving.

**Has this already been addressed?** Section 9 of the specification names delegated authorization as an open direction. If a proposal exists — within AP2, in the wider agent-systems literature, or in a design this fork has not found — a reference would be more useful than this repository.

---

*No results yet. This document will carry them when there are any.*
