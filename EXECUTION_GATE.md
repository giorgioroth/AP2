# Execution Gate — Research Notes

*An experimental fork of the AP2 reference implementation. Not affiliated with Google. Not part of the official AP2 project.*

This document records what was observed in the AP2 reference implementation, what those observations appear to mean architecturally, and what this fork proposes to test. It is not a critique of AP2. The observations concern where an existing constraint-evaluation mechanism becomes binding, not whether that mechanism exists.

**Scope of verification.** Code observations are from commit `e1ea56d` (2026-04-29). Read: `code/sdk/python/ap2/sdk/constraints.py`, `payment_mandate_chain.py`, `checkout_mandate_chain.py`; the `shopping_agent_v2` mandate tools; and the counterparty roles under `code/samples/python/src/roles/` in both their ADK and MCP forms. The remainder of `code/sdk/` was searched but not read in full. The x402 sample flow was executed twice: once with all services reachable, once with the settlement endpoint unavailable. Both runs are recorded in [`REPRO_X402_SETTLEMENT_FAILURE.md`](REPRO_X402_SETTLEMENT_FAILURE.md) and summarised in Section 4. Specification observations are from `docs/ap2/specification.md` in the repository at that same commit, which is Agentic Payment Protocol v0.2. Nothing below rests on a summary. Where a claim is an inference rather than a reading, it is marked as one.

Two errors in earlier drafts are recorded here rather than removed.

*Corrected 2026-07-29, second error.* A previous version of this document cited three passages of the specification — a dispute table row named *Mispick, Unapproved by User*, a Section 7.4 disclaiming risk and fraud handling, and a Section 9 titled *A Call for Ecosystem Collaboration* listing delegated authorization as open ecosystem work. Those passages belong to AP2 v0.1, read from a documentation URL that has since been reorganised and now returns 404. The specification carried in the repository at commit `e1ea56d` is v0.2, and contains none of them: no Intent Mandate, no Cart Mandate, no such sections. The citations were to a superseded version of a document while the code was pinned to a later commit. All specification claims below have been rewritten against v0.2 as carried in the repository. The code observations were unaffected; they were read from the pinned commit throughout.

*Corrected earlier the same day, first error.* An earlier draft stated that AP2 counterparties verify credentials without re-evaluating commercial constraints. That was wrong. It was written after reading the ADK role files and before reading the MCP ones, and it generalized from half the roles to all of them. The correction is in Section 1, and it narrows the thesis of this fork rather than supporting it.

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

In the x402 flow, the merchant is fail-closed when its own verification key is unavailable, and then delegates settlement to a PSP whose corresponding missing-key branch is fail-open. The path only reaches the PSP after the merchant's own verification succeeded, so the two branches are not shown firing in a single run. Whether the two components can encounter different key availability in a supported deployment has not been tested.

Whether that branch is reachable in a provisioned deployment has not been tested here. It is reported as an implementation observation and an internal inconsistency, not as a demonstrated vulnerability.

This was sent to the Google security team first, as issue 540649426, since the affected path can reach a broadcast. They replied that it does not meet their threshold for tracking as a security bug and invited public disclosure, and it was then filed as [google-agentic-commerce/AP2#309](https://github.com/google-agentic-commerce/AP2/issues/309) on 2026-07-31.

*Update, 2026-10-01.* After #309 was filed, [AP2#310](https://github.com/google-agentic-commerce/AP2/pull/310) by chopmob-cloud reported a direct reproduction against `settle_payment`: with the key absent, the pre-fix path skipped mandate verification and reached Step 1 binding; with the patch, it returns `agent_provider_key_missing` before binding. Silentpartnercoding then found an invalid-JWK case that escaped the original exception guard; the proposed fix now fails closed on any key-load failure, with regression tests for missing, invalid-JWK, non-object-JSON and empty key files, and a test showing that verification still runs and rejects a malformed mandate when a valid key is present. The reproduction and the local test results are reported by the PR author and were not rerun here. The fix is upstream work by others and is not part of this repository; this fork's contribution was the report and review comments. Workflow and Biome changes that had entered the PR were split out, the Biome configuration into [AP2#366](https://github.com/google-agentic-commerce/AP2/pull/366), and the PR branch is back at `0a5841e`. As of 2026-10-01, #310 is open and unmerged.

#309 is also listed, with issue 255 and PRs 300 and 310, among the public evidence sources for case AP2-05 in Table VI of *A Formal Analysis of Agent Payment Protocols* ([arXiv:2609.00060](https://arxiv.org/abs/2609.00060)), with match strength "Partial", meaning the cited evidence covers a strict subcase of the formal model. The paper analyses the upstream protocols and samples; it does not mention or evaluate the gate proposed here or the Regen Engine. None of this changes the status of the observation in this document: no unauthorized settlement and no broadcast were demonstrated here.

**Internal state is persisted before external confirmation, with no rollback on failure.** This is a separate property from constraint checking and is recorded here because it is visible in the same file. In `merchant_agent_mcp/server.py`, `complete_checkout` marks the payment token consumed at line 923 (`token_data['used'] = True`), assigns an `order_id`, records `amount_charged`, and persists all of it at line 946 (`_save_token_store`). The settlement call to the x402 PSP is issued afterwards, at line 984. On failure the function returns `PSP_call_failed` at line 1000. Between the persist and the failure return there is no restoration of the token state: no assignment of `used` back to `False`, no deletion from the store, no compensating write. The card flow at line 1013 has the same ordering and raises `ValueError` on processor error, also without restoration. This window was tested by forcing the x402 settlement call to fail at connection time and inspecting the persisted token store. The reproduced result is recorded in Section 4 and in [`REPRO_X402_SETTLEMENT_FAILURE.md`](REPRO_X402_SETTLEMENT_FAILURE.md).

**Agent-side, the same verdict is advisory.** `check_constraints_against_mandate`, in `shopping_agent_v2/shopping_agent/mandate_tools.py`, resolves the open mandates from session state, builds a candidate `PaymentMandate`, calls `check_payment_constraints`, and returns `meets_constraints`, `violations`, the checked price and availability, and extracted constraints such as `price_cap` and `line_items`. The prompts instruct the model not to proceed unless `available` and `meets_constraints` are both true. That obligation lives in the prompt and in tool ordering. It is not encoded in the code: `create_checkout_presentation` (70 lines) and `create_payment_presentation` (128 lines) are separately exposed tools, and neither contains any call to the constraint check before constructing and signing a mandate.

**The specification requires constraint verification at the counterparties.** Under *Verification*, v0.2 places a MUST on the Merchant: it must process and verify the Checkout Mandate, and, where open Checkout Mandates are included, verify that the closed Checkout conforms to all of the Constraints by evaluating each Constraint. The Credential Provider and Network carry the same obligation for the Payment Mandate. The Merchant Payment Processor must verify that the Payment Credential is appropriately scoped to the Checkout. The four fail-closed MCP roles above implement these duties. In `x402_psp_mcp`, the verification that would establish that scoping sits inside the key-conditional branch described above.

**The specification assigns the Shopping Agent no verification duty.** The *Verification* section lists rules for the Merchant, the Credential Provider and Network, the Merchant Payment Processor, and for dispute. The Shopping Agent is not among them; its obligations elsewhere concern assembling mandates, obtaining signatures, presenting only necessary disclosures, and not presenting a subsequent open Mandate before receiving a rejection receipt for the previous one. The agent-side constraint check in the samples is therefore a convenience of the reference implementation, not a duty the specification imposes.

**Validation is required to be deterministic.** v0.2 states that where the document refers to validation or processing for a particular role, it must happen in deterministic code, whether the role is agentic or not. The roles carrying verification duties are the counterparties, and in the reviewed MCP components those checks are in code rather than in prompts.

---

## 2. Architectural Observation

Prevention exists in AP2, and the specification requires it. v0.2 places verification duties on the Merchant, the Credential Provider and Network, and the Merchant Payment Processor, and four counterparty components in the samples implement them by refusing violating mandates before issuing tokens, completing checkout, or creating receipts. Any claim that AP2 supplies evidence without enforcement is refuted both by the specification and by its own samples.

What remains is narrower, and it concerns binding rather than existence:

> The same SDK evaluator runs on both sides of the boundary. At the counterparties it is a code-level precondition, and the specification requires it to be one: violations produce a refusal on the return path. At the agent it is a prompt-level instruction, and the specification asks for nothing more, since it assigns the Shopping Agent no verification duty. The tools that construct and sign mandates contain no call to it, so an agent that skips the check can still produce a well-formed mandate and send it.

This fork proposes to test whether making the agent-side evaluation a code-level precondition changes any outcome that matters. It is worth being explicit that this would add a check the specification does not ask for, at a role the specification does not charge with verification.

It may not. Downstream refusal already prevents settlement in the four fail-closed paths, and duplicating a decision earlier may buy only latency and maintenance cost. It may matter if earlier refusal avoids constructing and signing an invalid mandate at all, reduces what is disclosed to counterparties, contains an agent failure before it crosses a trust boundary, or covers a path where downstream verification is absent or skipped. Those are hypotheses.

*The following is an inference, not a reading.* Final settlement raises the cost of any gap in enforcement, because no adjudicator can reverse the transfer afterwards. The specification treats dispute evidence as a mechanism whose resolution, retention and retrieval it leaves out of scope; on a rail without chargeback, what an adjudicator could do with that evidence is a question the specification does not answer and this fork cannot answer either. The inconsistency described above is the one place in the reviewed code where a settlement component can proceed with neither signature nor constraint verification. That does not establish that redundant enforcement near the action source is necessary; a correctly provisioned deployment may never reach it. It does show that the reliability of downstream refusal is a property of each component's configuration rather than of the specification that mandates it, which is a weaker claim and the only one the code supports.

---

## 3. Proposed Integration Point

The proposed integration point would require no change to the protocol, the message formats, or the delegation model.

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

The gate would sit between constraint evaluation and mandate construction. Its input is the verdict the SDK already produces; the intended difference is that the mandate-construction tools could not be reached around it. Whether that holds has not been shown. If it did, the agent-side check would stop being a prompt-governed instruction and become a code-level precondition.

Everything downstream would remain unchanged: the same mandates, signatures, delegation chain, and counterparty verification. A merchant or processor receiving a mandate produced through the gate could not tell the difference; interoperability would be preserved by design.

**Proprietary components.** The gate proposed here is deliberately minimal and is described to show where such a boundary could attach. No gate implementation is included in this repository, and its effectiveness and resistance to bypass have not been demonstrated. The production execution kernel, the admissibility model and the enforcement mechanisms belong to the Regen Engine project and are not included in this repository.

**Modified files.** The AP2 code is licensed under Apache 2.0. Files modified by this fork will carry a prominent notice that they were changed, as Section 4(b) of that licence requires. This fork will additionally record what changed and when, which the licence does not require but provenance does. Unmodified files remain untouched.

---

## 4. Reproduced Result

**Internal state persists after a failed settlement call.** The x402 sample flow was run twice at commit `e1ea56d`: once with all services reachable, and once with the settlement endpoint stopped. In the second run `complete_checkout` returned `PSP_call_failed: All connection attempts failed`. No receipt and no transaction hash were produced. The merchant token store nevertheless recorded the payment token as `used: true` with an allocated `order_id`.

The ordering in the source matches: the token is marked used at line 923, the order state is persisted at line 946, the settlement call is issued at line 984, and the failure is returned at line 1000, with no restoration between them.

Full conditions, the baseline run, and the limits of the claim are in [`REPRO_X402_SETTLEMENT_FAILURE.md`](REPRO_X402_SETTLEMENT_FAILURE.md). In particular this establishes neither loss of funds, nor on-chain consumption of the EIP-3009 authorization, nor anything about deployments other than the one run.

The observation was reported upstream as [google-agentic-commerce/AP2#308](https://github.com/google-agentic-commerce/AP2/issues/308) on 2026-07-30, asking whether the ordering is intentional for the samples. Whatever the maintainers answer will be recorded here. As of 2026-10-01 the issue is open and no maintainer answer has been posted.

This answers one of the questions in Section 5 and leaves the rest open. It also shifts what this fork is about: the demonstrated result is a failure of atomicity across an external boundary, not a result about admissibility before it. Whether the agent-side gate described in Section 3 changes any outcome remains untested.

---

## 5. Open Research Questions

The questions below define the remaining work. One has been answered by the reproduction in Section 4 and is kept struck through rather than deleted, so that the record shows a debt closed by a result rather than a question invented after one. The others remain open.

**Does earlier binding change any outcome that matters?** If every action the gate blocks is one a fail-closed counterparty would have refused anyway, the gate is overhead. Demonstrating a difference is the first thing that must be shown, and it has not been shown.

**~~Does the reviewed flow leave partial internal state when settlement fails?~~** *Answered — see Section 4.* It does, for a settlement call that fails at connection time. The two harder cases remain open: a settlement that succeeds while the response is lost, and one that fails after partial processing. Neither is addressed by the reproduction, and the second is the case where restoration alone would not be a sufficient answer.

> *External design observation, 2026-07-30.* In [AP2#308](https://github.com/google-agentic-commerce/AP2/issues/308) an independent commenter proposed separating settlement attempt, settlement outcome and token consumption, using explicit pending and failed states together with an idempotency key. At the architectural level this bears on the lost-response case above: reconciliation could then key on the idempotency key rather than restore the token blindly. It is a structure capable of carrying that case, not a resolution of it — a pending state without a processor that recognises the key, a way to query the outcome, and defined semantics for leaving the pending state only names the uncertainty. Nothing here has been implemented or tested in this fork, and it is recorded as provenance for a hypothesis rather than as an answer.

**Is the agent the right place for it?** A gate inside the agent's own process is reachable by the agent's own failure modes. A gate outside costs a boundary crossing per action. The samples already evaluate constraints in both places; this fork proposes to bind the earlier one, which may still be the wrong placement.

**Where does this responsibility properly belong?** Protocol, wallet, agent framework, application runtime, middleware, counterparty, or some combination. The reviewed samples already distribute enforcement across several of these. This fork proposes one additional binding point and does not argue that it is the correct one.

**Does irreversibility actually change the requirement?** The inference in Section 2 is that it raises the cost of a gap. It could be wrong. If downstream checks are reliably provisioned and compensating transactions are cheap enough in practice, the earlier gate may change nothing worth having, and this fork explores a problem that does not need solving.

**Has this already been addressed?** v0.2 assigns verification duties to the counterparties and none to the Shopping Agent, so a code-level precondition at the agent is outside what the specification asks for rather than a gap it names. If a proposal exists for binding it there — within AP2, in the wider agent-systems literature, or in a design this fork has not found — a reference would be more useful than this repository.

---

*One result has been reproduced and is recorded in Section 4. Further results will be recorded with their execution conditions and their limits.*
