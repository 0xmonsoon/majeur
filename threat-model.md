# Threat Model Report: Moloch (Majeur) DAO Framework

## Roles

### Administrative Roles


| Role                   | Privileges                                                                      |
| ---------------------- | ------------------------------------------------------------------------------- |
| **DAO (self-call)**    | Controls governance, execution, permits, allowances, sale and transfer settings |
| **Summoner (Factory)** | Deploys DAO instances and calls `init()`                                        |
| **Ops Beneficiary**    | Receives configured DAICO tap payouts                                           |


### User Roles


| Role                     | Actions                                            |
| ------------------------ | -------------------------------------------------- |
| **Share Holder**         | Vote, delegate, ragequit, transfer shares, propose |
| **Loot Holder**          | Hold and transfer loot; ragequit                   |
| **Badge Holder**         | Post chat messages and hold badge identity         |
| **Proposer**             | Open proposals and manage own tribute offers       |
| **Buyer**                | Buy via `buyShares`, `buy`, `buyExactOut`          |
| **Delegate**             | Receive delegated voting power                     |
| **Futarchy Participant** | Fund pools, hold receipts, and cash out            |


### External Systems


| System                           | Integration Point                                                                  |
| -------------------------------- | ---------------------------------------------------------------------------------- |
| **ZAMM DEX**                     | `DAICO._initLP()` calls `ZAMM.addLiquidity()` at hardcoded singleton `0x000...0eD` |
| **Arbitrary ERC-20 Tokens**      | Payment, tribute, and sale token flows                                             |
| **Arbitrary External Contracts** | `executeByVotes` and `batchCalls` can call/delegatecall any address                |


---

## Assets


| Asset                     | Description                                                       |
| ------------------------- | ----------------------------------------------------------------- |
| **DAO Treasury**          | ETH and ERC-20 balances held by `Moloch`                          |
| **Share Tokens**          | Voting and membership token balances                              |
| **Loot Tokens**           | Non-voting token balances                                         |
| **Futarchy Pools**        | Proposal-linked reward pools                                      |
| **Permit Rights**         | ERC-6909 permit receipts used by `spendPermit`                    |
| **Treasury Allowances**   | `allowance[token][spender]` spending rights                       |
| **Tribute Escrow Funds**  | Funds held in `Tribute` pending claim/cancel                      |
| **DAICO Sale State**      | Sale, tap, and LP configuration plus sale flow balances           |
| **Governance Parameters** | Quorum, TTL, timelock, thresholds, ragequit, transfer lock config |


---

## Trust Assumptions

This threat model assumes governance is honest and non-malicious. Governance-dependent risks are intentionally moved out of `Security Threats` and listed here:

- **Execution powers are used safely**: DAO-controlled `call` / `delegatecall` paths (`executeByVotes`, `spendPermit`, `batchCalls`) are assumed to target safe contracts with safe calldata.
- **Governance parameters remain safety-oriented**: quorum, min-yes, TTL, and timelock are assumed to be configured for meaningful review and response windows.
- **Permit issuance is responsibly scoped**: permit counts, recipients, and intent definitions are assumed to be tightly managed by governance process.
- **Exit controls are used in good faith**: ragequit and transfer lock toggles are assumed not to be used to trap minority participants.
- **Economic knobs are prudently configured**: sale minting mode, auto-futarchy settings, tap rates, and treasury allowances are assumed to avoid self-harm.

---

## Security Threats

This section contains only protocol-level vulnerabilities under the trust assumptions above.


| Threat                                             | Description                                                                                                                   | Affected Surface                                             | Priority   |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | ---------- |
| **Permit/proposal ID namespace coupling**          | Proposals and permits share `_intentHashId` and `executed[id]`. Spending a permit can tombstone the same intent on vote path. | `_intentHashId`, `spendPermit`, `executeByVotes`, `executed` | **MEDIUM** |
| **Zero winning supply can strand futarchy pool**   | If resolved winner has zero receipt supply, payout per unit becomes zero and pool has no claim path.                          | `_finalizeFutarchy`, `cashOutFutarchy`                       | **MEDIUM** |
| **Fee-on-transfer/rebasing token incompatibility** | Core accounting assumes exact transfer amounts in multiple paths; non-standard token semantics can violate those assumptions. | Moloch/DAICO/Tribute token transfer paths                    | **HIGH**   |


---

## Recommendations

- **Harden permit/proposal namespace separation**: Split permit IDs from proposal IDs (domain-separate hash or separate `executed` latches) to avoid accidental or strategic tombstoning collisions.
- **Patch futarchy zero-winner handling**: Add reclaim/refund path for `winSupply == 0` pools to avoid stranded value.
- **Define token compatibility explicitly**: Either reject fee-on-transfer/rebasing tokens or add balance-delta accounting and tests across all transfer paths.

