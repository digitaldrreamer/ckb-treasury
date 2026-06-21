# Gist Comments

Comments from [The Self-enforcing Treasury gist](https://gist.github.com/digitaldrreamer/382a99898a44f4b2db2266b5b5c1c9c6).

---

## @RobaireTH — June 6, 2026

This is a great initiative. So far, I have a few questions I think the read doesn't answer. Formal specification of financial conditions is hard, and the cost of getting it wrong is permanent and unrecoverable. The piece gestures at power without sitting with that weight. This sounds fun, yeah.

If the proof type script has a bug, there's no key to rotate and no owner to fix it. The piece's "no one controls it" property becomes a liability the moment there's a logic error. 🤔🤔

You should do some more good researches and get building. I'm rooting for you.

---

## @digitaldrreamer — June 12, 2026 (Research Notes)

## Research Notes — Comparative Landscape (June 2026)

These are notes on whether anything comparable to the Self-enforcing Treasury pattern described here has been built before. Short answer: **not in this combination, and not on a live system.**

---

### What was searched

- Autonomous / keyless treasury patterns across blockchain ecosystems
- Bitcoin covenant proposals (CTV, OP_VAULT, OP_CAT)
- Cardano eUTXO / Plutus validator treasury patterns
- Ergo box model / ErgoScript / ZK Treasury
- Ethereum DAO treasury (Governor + Timelock + Safe)
- Academic literature on UTXO-based condition spending and proof-conditional treasuries
- CKB `since` field as consensus-enforced time primitive

---

### The closest prior art

#### Bitcoin covenants (CTV / OP_VAULT / OP_CAT)

Closest in spirit. `OP_CHECKTEMPLATEVERIFY` lets a UTXO commit to a specific spending transaction structure without a signature. `OP_VAULT` adds a mandatory delay before funds can move — conceptually similar to the `since` field usage here.

**Gap**: Not deployed (still proposals as of mid-2026). More importantly, Bitcoin scripts cannot *read cell data* to construct dynamic spending conditions. CTV commits to a fixed transaction template; it cannot inspect a proof cell's payload, validate a receipt, or enforce arbitrary logical conditions. The expressiveness gap is large. Also, Bitcoin has no concept of recoverable storage costs — the self-replenishing treasury loop has no analogue.

References:

- https://bitcoinops.org/en/topics/covenants/
- https://bitcoinops.org/en/topics/op_checktemplateverify/
- https://bitcoinops.org/en/topics/vaults/
- https://blog.bitfinex.com/education/what-covenant-proposals-are-being-looked-at-for-bitcoin-in-2025/

#### Ergo (ErgoScript / eUTXO box model)

Technically the closest chain overall. ErgoScript is Turing-limited (by design) but can inspect the full spending transaction — inputs, outputs, data fields — which is the same property that makes the treasury/proof mutual-validation work on CKB. Ergo also has a **ZK Treasury** concept (announced 2020) where groups authorize spending using composable sigma protocols (ring signatures) without a single controlling key.

**Gap**: Ergo's ZK Treasury still requires a *cryptographically generated human artifact* (a ring signature or sigma proof) as the authorization mechanism. The CKB pattern eliminates that layer: the proof cell *is* the authorization, governed purely by a type script. No human needs to generate a cryptographic witness. Additionally, Ergo does not have CKB's explicit recoverable storage costs — the self-replenishing loop (capacity out on proof creation → capacity in on proof consumption, net fees only) is a direct product of CKB's capacity model. Ergo has no equivalent economic mechanic.

References:

- https://ergoplatform.org/en/blog/2020-09-04-announcing-the-zk-treasury-on-ergo/
- https://docs.ergoplatform.com/dev/scs/ergoscript/
- https://docs.ergoplatform.com/dev/data-model/box/
- https://docs.ergoplatform.com/uses/zkt/

#### Cardano (Plutus / eUTXO)

The document already notes this. Plutus validators can inspect transaction inputs and outputs and enforce complex spending conditions. The execution model is semantically similar.

**Gap**: Cardano requires every UTXO being referenced to be pre-declared in the transaction. Wiring up mutual-reference patterns — where treasury validator and proof validator simultaneously validate each other against the same transaction — is architecturally more cumbersome. No native consensus-enforced time constraint equivalent to CKB's `since` field exists; time logic must go through validity intervals with off-chain coordination. No recoverable storage cost model.

References:

- https://developers.cardano.org/docs/learn/core-concepts/eutxo/
- https://docs.cardano.org/about-cardano/learn/eutxo-explainer
- https://arxiv.org/pdf/2406.07700 (Scalable UTXO Smart Contracts via Fine-Grained Distributed State)

#### Ethereum DAOs (Governor + Timelock + Safe)

Standard infrastructure for on-chain treasuries. Governor contract, Timelock, Safe multisig, module ecosystems (Allowance, Zodiac, etc.).

**Gap**: Always terminates in a human-controlled key or multisig. The Timelock queues transactions but requires a *proposer with authority* to queue them. The committee/key problem described in the opening of this document is precisely what Ethereum DAO tooling cannot escape. No UTXO model, no recoverable storage, no mutual transaction-level validation. Reentrancy is a live concern in inter-contract calls.

References:

- https://ethereum.org/dao/
- https://safe.global/
- https://www.blockchain-council.org/cryptocurrency/dao-treasury-audit-guide-multisig-spending-proposals-on-chain-governance/

#### Academic literature

A 2024 paper (arXiv:2406.07700) specifically addresses scalable UTXO smart contracts via fine-grained distributed state and evaluates crowdfund, registry, and multisig wallet contracts in UTXO-based models — noting that UTXO-based models face specific attack surfaces (adversarial output injection) not present in account-based models. No prior work found describing the specific treasury+proof mutual-validation pattern or the capacity-recovery loop.

---

### What is genuinely novel in the combination

Three properties combine that have not been combined on a live system:

**1. Condition-as-authorization with no cryptographic artifact.**
The proof cell is the key. Anyone who constructs a valid proof cell (per the type script rules) can trigger a treasury spend. Nobody can without one. There is no private key path to the funds at all. This is not "keyless" in the MPC/TSS wallet sense (which still uses cryptographic key shares). The authorization is purely structural/logical.

**2. The self-replenishing economic loop.**
On CKB, value and storage are the same concept. Capacity that leaves the treasury to create a proof cell returns when the proof cell is consumed. Net cost per cycle = transaction fees only. This isn't a design trick — it's what the model naturally produces. No other major chain has this property in the same form.

**3. Simultaneous mutual validation without inter-contract calls.**
The treasury type script and proof type script run against the same transaction in the same consensus evaluation — neither calls the other, neither trusts a return value. No reentrancy surface exists. The mutual guarantee emerges from the CKB execution model. This is structurally different from Ethereum (msg.call, delegatecall) and from Cardano (each validator is local to its UTxO, requiring explicit reference input wiring).

---

### Honest qualifications

- Ergo is the narrowest gap. If you have read this document and know ErgoScript well, Ergo is where you'd look for the closest prior art. The ZK Treasury is sophisticated; the main delta is that it still uses sigma-protocol authorization and lacks the capacity recovery loop.
- The pattern isn't deployed yet (as of the time of writing); it's been prototyped on CKB testnet within the Transaction Firewall governance system. The crowdfund implementation referenced at the end is in progress.
- The "no one controls it" property applies to the *operational* layer. Whoever deploys the type scripts sets the rules. Trustlessness is about ongoing operation, not inception.
- If Bitcoin gets OP_CAT + full transaction introspection, a significant subset of this pattern becomes expressible on Bitcoin — but the capacity recovery loop would still be absent.

---

*Notes compiled June 2026. Searches covered: Bitcoin covenant proposals, Ergo ZK Treasury, Cardano Plutus eUTXO, Ethereum DAO infrastructure, CKB type script documentation, UTXO smart contract academic literature.*

---

## @digitaldrreamer — June 12, 2026 (reply to @RobaireTH)

> This is a great initiative. So far, I have a few questions I think the read doesn't answer. Formal specification of financial conditions is hard, and the cost of getting it wrong is permanent and unrecoverable. The piece gestures at power without sitting with that weight. This sounds fun, yeah.
>
> If the proof type script has a bug, there's no key to rotate and no owner to fix it. The piece's "no one controls it" property becomes a liability the moment there's a logic error. 🤔🤔
>
> You should do some more good researches and get building. I'm rooting for you.

@RobaireTH, you have a point.

The answer is that the mitigation is the same as it is anywhere in smart contract development: you do formal specification, thorough testing, and testnet deployment before any mainnet funds go in. The governance system (and your Pckt project) has been running on testnet specifically for this same reason: to surface logic errors before you use mainnet funds.

You're right that "no one controls it", but it goes both ways. The same property that removes the trusted party also removes the escape hatch. On Ethereum you can build in an upgrade proxy or an owner pause, but here, any post-deployment recovery mechanism reintroduces a trusted party and undermines the point of using the pattern. So the risk isn't different in kind from any immutable contract, but it's that the pre-deployment diligence bar is higher in degree, because there's less you can do after the fact. In short, one needs to be careful before deploying to mainnet.

I suppose its a real point to address, and the document should probably say it more directly than it does. The permanence of bugs is the cost of the trustlessness. The response to that cost is to not deploy underbaked scripts, because the pattern is only worth building because the diligence goes one-time into the scripts, and there's no ongoing trust management/maintenance. At the very least the energy that would have been used in multisig and runtime maintenance can be applied to the scripts instead.
