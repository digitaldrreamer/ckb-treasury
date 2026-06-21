# SPEC-TREASURY: Self-Enforcing Shared Pool

Status: **Draft**  
Version: **0.2.0**  
Depends on: [`SPEC-CORE.md`](./SPEC-CORE.md) (Structural Authorization Pattern)  
Related: [`structural-authorization-ckb.md`](./structural-authorization-ckb.md)

This document specifies the **treasury pattern** — a shared, donatable pool built on the structural authorization pattern defined in SPEC-CORE. The pool's capacity can only move when a valid condition cell of an authorized type appears in the same transaction.

Treasury-pattern applications (governance, pooled dev funds, …) are plug-ins that implement the condition script interface. Each is defined in a separate `SPEC-<APPLICATION>.md`. Applications that only need structural authorization (bounties, bonds, escrows, …) are specified against [`SPEC-CORE.md`](./SPEC-CORE.md) directly — see [`POSSIBILITIES.md`](./POSSIBILITIES.md).

---

## 1. Scope

### 1.1 In scope

- Treasury cell (guarded cell specialized as a shared pool)
- Donation mechanism
- Lifecycle: anchor, execute, abort transaction classes
- Capacity accounting for pooled funds
- Self-replenishment (for temporary-claim applications)
- Condition cell ("proof cell") requirements specific to treasury use
- Application plug-in checklist

### 1.2 Out of scope

- The structural authorization mechanism itself (see [`SPEC-CORE.md`](./SPEC-CORE.md))
- Application-specific condition logic (see `SPEC-<APPLICATION>.md`)
- Concrete cell data layouts for applications
- Frontend / mempool coordination
- Mainnet deployment checklist

---

## 2. Terminology

All terms from SPEC-CORE apply. Additional terms:

| Term | Definition |
|------|------------|
| **Treasury cell** | A guarded cell (per SPEC-CORE) used as a shared pool. Holds donatable, pooled capacity. |
| **Proof cell** | A condition cell (per SPEC-CORE) that represents an on-chain claim funded from the treasury. |
| **Anchor TX** | Transaction that spends the treasury to create a proof cell. |
| **Execute TX** | Transaction that consumes a proof cell after conditions are met, producing a result and (for temporary-claim applications) returning capacity to the treasury. |
| **Abort TX** | Transaction that consumes a proof cell without executing the success path, returning capacity to the treasury. |
| **Authorized proof type** | The authorized condition type (per SPEC-CORE) that a treasury instance accepts. Fixed in treasury `args`. |
| **Result** | Application-defined outputs produced on successful execute (e.g. updated registry, payout cell). |

---

## 3. Relationship to SPEC-CORE

The treasury pattern is one application of the structural authorization pattern:

```
┌──────────────────────────────────────────────────────────┐
│  CKB primitives: type scripts, cells, since, capacity    │
├──────────────────────────────────────────────────────────┤
│  SPEC-CORE: Structural Authorization Pattern             │
│  (guarded cell + condition cell + mutual validation)     │
├──────────────────────────────────────────────────────────┤
│  SPEC-TREASURY (this document): Shared Pool Pattern      │
│  (donation, lifecycle, replenishment, abort)              │
├──────────────────────────────────────────────────────────┤
│  Applications: governance, bounty, grants, …             │
├──────────────────────────────────────────────────────────┤
│  Implementations: concrete schemas, scripts, TX builders │
└──────────────────────────────────────────────────────────┘
```

The treasury pattern adds to SPEC-CORE:
- A **donation mechanism** (anyone can add capacity to the pool)
- A **multi-phase lifecycle** (anchor → delay → execute/abort)
- **Capacity replenishment** for temporary-claim applications
- An **abort path** for conditions that can't be fulfilled

SPEC-CORE's structural authorization mechanism also supports applications that don't need the treasury pattern — single-phase conditional payments, escrows, proof-of-work rewards, etc. Those applications reference SPEC-CORE directly.

---

## 4. Treasury-Specific Invariants

These extend SPEC-CORE's invariants (§4). All SPEC-CORE invariants remain in force.

### 4.1 Pool Safety

| ID | Invariant |
|----|-----------|
| **INV-TS1** | No transaction may reduce treasury capacity except via an anchor TX that creates one or more authorized proof outputs. |
| **INV-TS2** | A treasury input may appear only in anchor, execute, or abort transactions that satisfy this spec. |
| **INV-TS3** | A proof cell may be created only in an anchor TX that passes both the guard script and the condition script. |
| **INV-TS4** | A proof cell may be consumed only in an execute or abort TX that passes both scripts. |

### 4.2 Accounting

| ID | Invariant |
|----|-----------|
| **INV-TA1** | On anchor: `Σ(treasury_in.capacity) = treasury_out.capacity + Σ(proof_out.capacity) + fee`. Additional fee-paying inputs may contribute. |
| **INV-TA2** | On execute: `proof_in.capacity + treasury_in.capacity = treasury_out.capacity + Σ(result.capacity) + fee`. |
| **INV-TA3** | On abort: `proof_in.capacity + treasury_in.capacity = treasury_out.capacity + fee`. All proof capacity returns. |
| **INV-TA4** | Treasury and proof outputs MUST meet minimum CKB occupancy for their byte size. |
| **INV-TA5** | For **temporary-claim** applications (e.g., governance): net treasury loss per anchor→execute cycle equals transaction fees only. For **permanent-transfer** applications (e.g., bounty payout): proof capacity is consumed by result outputs. Application spec MUST declare which class it belongs to. |

### 4.3 Time (when used)

| ID | Invariant |
|----|-----------|
| **INV-TT1** | If an application requires a delay between anchor and execute, it MUST be enforced via the CKB `since` field on the proof input (relative or absolute MTP as defined by the application spec). |
| **INV-TT2** | Execute/abort MUST NOT be includable before `since` is satisfied (consensus-enforced, not application-enforced). |

### 4.4 Liveness

| ID | Property |
|----|----------|
| **INV-TL1** | Anyone may submit a valid anchor, execute, or abort TX (no privileged relayer). |
| **INV-TL2** | Donations MUST succeed by creating separate UTXOs at the treasury address, without triggering the guard script (per SPEC-CORE INV-L2). |
| **INV-TL3** | Treasury must have sufficient capacity (≥ minimum proof capacity + fees) to create a proof; otherwise, anchor creation halts until donations replenish the pool. |

---

## 5. Transaction Classes

### 5.1 Donate

Treasury is not spent. A user sends capacity to the treasury address, creating a new, separate UTXO.

```
inputs:  [ donor_cell ]
outputs: [ donation_cell (treasury lock + treasury type) ]
```

Donation UTXOs are later merged during an anchor transaction. The guard script does not run on donation (the treasury cell is not an input).

### 5.2 Anchor TX (create proof)

Creates one or more proof cells funded from the treasury, merging any donation UTXOs.

```
inputs:
  - treasury_cell(s)    (one or more existing treasury/donation cells)
  - (optional) fee_paying_cell

outputs:
  - proof_cell(s)       (authorized type, each ≥ application-defined minimum capacity)
  - treasury_cell       (single consolidated change output, reduced capacity)

witnesses:
  - treasury: "0x"      (no signature — per SPEC-CORE INV-S3)

cell_deps:
  - guard script binary
  - condition script binary
```

**Guard script (anchor mode):**  
- One or more treasury inputs (merging donations).
- Exactly one treasury output (change).
- One or more proof outputs matching `authorized_condition_script_hash`.
- Capacity conservation per INV-TA1.

**Condition script (creation mode):**  
- Validates proof output structure, data, and any creation conditions.
- Application-specific rules in `SPEC-<APPLICATION>.md`.

### 5.3 Execute TX (consume proof, success path)

Consumes proof after conditions are met; produces result; returns capacity to treasury (for temporary-claim applications) or sends it to result outputs (for permanent-transfer).

```
inputs:
  - treasury_cell       (consumed and recreated for replenishment)
  - proof_cell          (since: per application, if required)
  - ... application inputs (e.g. registry cell, vote cells)

outputs:
  - result_cell(s)      (application-defined)
  - treasury_cell       (replenished, or same capacity if permanent-transfer)

witnesses:
  - per condition / application requirements (signatures, preimages, etc.)

cell_deps:
  - guard script binary
  - condition script binary
  - application deps
```

**Guard script (execute mode):**  
- Detects authorized proof cell consumed as input.
- Verifies treasury output receives appropriate capacity (INV-TA2).
- Does not re-validate application logic beyond proof type identity and accounting.

**Condition script (consumption mode):**  
- Validates all application success conditions.
- Validates result outputs.
- Enforces `since` if applicable.

### 5.4 Abort TX (application-defined)

Returns proof capacity to treasury when success conditions will not be met (e.g. failed governance vote, expired deadline).

```
inputs:
  - treasury_cell       (consumed and recreated)
  - proof_cell          (since: per application abort rules)

outputs:
  - treasury_cell       (replenished with proof capacity)
  - (optional) tombstone / null result

witnesses:
  - per application abort rules
```

Whether abort exists and its conditions are **application-specific** but MUST be defined if proof cells can become stranded (no valid execute path). All proof capacity MUST return to the treasury on abort (INV-TA3).

> **Note:** Abort is strongly recommended for any application where proof cells persist on-chain before final approval (e.g. governance proposals awaiting votes).

---

## 6. Mode Detection (Treasury-Specific)

The guard script detects its execution context from the transaction structure:

| Mode | Condition |
|------|-----------|
| **ANCHOR** | Treasury cell in inputs AND authorized proof cell in outputs (but not in inputs) |
| **EXECUTE / ABORT** | Treasury cell in inputs AND authorized proof cell in inputs |
| **DONATE** | Treasury cell in outputs only (no treasury input) — permit without proof |

Distinguishing Execute from Abort is delegated to the condition script or handled via an explicit mode indicator in the proof cell's data. The guard script's only concern for Execute vs. Abort is capacity accounting: abort requires full capacity return (INV-TA3); execute allows capacity to flow to result outputs (INV-TA2).

---

## 7. When to Use This Pattern

This document specifies an **adaptation** of the structural authorization pattern ([`SPEC-CORE.md`](./SPEC-CORE.md)), not the pattern itself. Use SPEC-TREASURY only when all of the following apply:

- Multiple parties need to spend from a **common, donatable pool**
- No single party should be custodian of that pool
- Claims are **temporary** — capacity leaves the pool to fund proof cells and should return after execute or abort
- A **multi-phase lifecycle** (anchor → delay → execute/abort) is required
- Throughput requirements are low (~1 operation per CKB block)

If any of these do not apply, use SPEC-CORE directly.

### 7.1 Strong fit (treasury adaptation)

| Application | Why the treasury layer is needed |
|-------------|----------------------------------|
| **Governance treasury** | Shared pool funds proposal proof cells; capacity returns after execute or abort |
| **Pooled dev / grant funds** | Donations accumulate; milestone proofs trigger disbursement; pool replenishes or tracks balance across claims |
| **Insurance / mutual pools** | Shared premiums; claims consume proof cells; pool must persist across many policyholders |

### 7.2 Use SPEC-CORE directly instead

Most applications in [`POSSIBILITIES.md`](./POSSIBILITIES.md) use structural authorization only. They do not need donation, anchor/execute/abort, or replenishment machinery.

| Application | Why SPEC-TREASURY is unnecessary |
|-------------|----------------------------------|
| **Bug bounties** | One guarded cell per bounty; single-phase claim; capacity transfers to claimant permanently |
| **Accountability bonds** | 1:1 bond cell; referee attestation condition; no shared pool |
| **Dominant assurance / crowdfund** | Guarded pool cell with condition-script modes; no treasury proof lifecycle |
| **Bridge escrows** | 1:1 escrow cell; light-client proof condition; single-phase release |
| **Vesting / escrow** | 1:1 relationship; time-locked guarded cell suffices |
| **Salary / periodic payments** | 1:1 relationship; no multi-party pool |
| **Personal savings vault** | Single party; defeats the structural authorization premise |

> **Note:** A *program* that funds many individual bounties from one pool could use SPEC-TREASURY. An individual bounty cell does not — it is SPEC-CORE only.

---

## 8. Application Plug-in Checklist

Each `SPEC-<APPLICATION>.md` MUST define:

- [ ] Application class: **temporary-claim** or **permanent-transfer**
- [ ] Proof cell `data` schema (versioned)
- [ ] Proof `args` schema
- [ ] Creation rules (anchor — condition script `on_create`)
- [ ] Success rules (execute — condition script `on_consume`)
- [ ] Abort rules (if applicable — condition script abort path)
- [ ] Result output constraints
- [ ] `since` encoding (relative MTP seconds, absolute, units)
- [ ] Witness layout
- [ ] State machine diagram
- [ ] Mapping to transaction classes (§5)
- [ ] Edge-case / negative test matrix
- [ ] MIN_PROOF_CAPACITY for this application

### 8.1 Reference applications (treasury layer)

These applications use SPEC-CORE **and** the treasury adaptation defined in this document.

| Application | Spec document | Status |
|-------------|---------------|--------|
| Governance (Transaction Firewall) | `SPEC-GOVERNANCE.md` | TBD |

Applications that use structural authorization only (bounties, bonds, assurance contracts, …) are catalogued in [`POSSIBILITIES.md`](./POSSIBILITIES.md) and specified against [`SPEC-CORE.md`](./SPEC-CORE.md), not here.

---

## 9. Test Requirements (Treasury-Specific)

These extend SPEC-CORE's tests (§8).

### 9.1 Must-pass (positive)

| ID | Test |
|----|------|
| **TT+01** | Donate creates separate UTXO at treasury address |
| **TT+02** | Valid anchor creates proof + treasury change |
| **TT+03** | Anchor merges multiple donation UTXOs into one treasury output |
| **TT+04** | Valid execute consumes proof, replenishes treasury, produces result |
| **TT+05** | Valid abort returns all proof capacity to treasury |
| **TT+06** | Full cycle: donate → anchor → execute; net treasury delta = fees |
| **TT+07** | Empty witness on treasury input (anchor) |

### 9.2 Must-reject (negative)

| ID | Test |
|----|------|
| **TT-01** | Anchor without proof output |
| **TT-02** | Anchor with wrong proof `code_hash` |
| **TT-03** | Anchor with correct layout but wrong proof `args` |
| **TT-04** | Anchor violating capacity conservation |
| **TT-05** | Execute before `since` elapsed |
| **TT-06** | Execute without proof consumption |
| **TT-07** | Execute with insufficient treasury replenishment |
| **TT-08** | Extra attacker-controlled outputs failing condition script |
| **TT-09** | Proof from foreign deployment (wrong script hash) |
| **TT-10** | Direct treasury spend with signature witness (no proof) |
| **TT-11** | Abort that doesn't return full proof capacity |

### 9.3 Property tests

- Randomized output injection on anchor/execute
- Capacity sum invariant after every TX
- Fuzz proof `data` near boundary lengths
- Invariant: treasury capacity after full anchor→execute cycle = initial − fees

---

## 10. Open Questions (v0.2)

1. **Abort in guard vs condition script:** Should the guard script expose an explicit abort mode, or rely entirely on the condition script's branching?
2. **Outstanding proof cap:** Enforce on-chain in the guard script (count proof cells) or off-chain via coordinator?
3. **Proof capacity sizing:** Fixed per application or dynamic from proof data?
4. **Treasury UTXO set:** For higher throughput, should the spec define merge/split rules for multiple treasury cells (relaxing the singleton constraint)?

These SHOULD be resolved before first mainnet deployment.

---

## 11. References

- Core mechanism: [`SPEC-CORE.md`](./SPEC-CORE.md)
- Pattern overview: [`structural-authorization-ckb.md`](./structural-authorization-ckb.md)
- Discussion / prior art: [`structural-authorization-ckb-comments.md`](./structural-authorization-ckb-comments.md)
- CKB `since` / transaction structure: https://docs.nervos.org/docs/tech-explanation/since
- Live governance implementation: https://github.com/digitaldrreamer/ckb-transaction-firewall
- UTXO output injection: arXiv:2406.07700

---

## Appendix A: Pseudocode (treasury guard script)

This extends SPEC-CORE's guard pseudocode with treasury-specific lifecycle logic.

```
fn treasury_guard_main():
    authorized = load_args().authorized_condition_script_hash

    if treasury_in_inputs():
        mode = detect_mode()  // ANCHOR | EXECUTE | ABORT (see §6)

        match mode:
            ANCHOR =>
                assert at_least_one_treasury_input()
                assert single_treasury_output()
                proofs = find_proof_outputs(authorized)
                assert len(proofs) >= 1
                assert each proof.capacity >= MIN_PROOF_CAPACITY
                assert capacity_conserved(treasury_ins, treasury_out, proofs)

            EXECUTE =>
                proof_in = find_consumed_proof_input(authorized)
                treasury_out = find_treasury_output()
                // capacity accounting per INV-TA2
                // result outputs may consume some proof capacity
                assert capacity_balanced(proof_in, treasury_in, treasury_out, results)

            ABORT =>
                proof_in = find_consumed_proof_input(authorized)
                treasury_out = find_treasury_output()
                // all proof capacity returns — INV-TA3
                assert treasury_out.capacity >= treasury_in.capacity + proof_in.capacity - fee
    else:
        // Treasury in outputs only — donation / creation
        return OK
```
