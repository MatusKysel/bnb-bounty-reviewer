---
name: bnb-bounty-reviewer
description: Use when reviewing BNB Chain bug bounty submissions or security findings against in-scope BNB repositories such as bsc, bsc-genesis-contract, opbnb, op-geth, greenfield, greenfield-cosmos-sdk, greenfield-contracts, greenfield-storage-provider, or tss-lib. Trigger on reports involving BSC/Parlia, Greenfield/SP, cross-chain packages, system contracts, validator logic, RPC/P2P, threshold-signing/TSS, mnemonics/keys, or exploit PoCs.
---

# BNB Chain Bug Bounty Reviewer

Review BNB Chain bug bounty reports by verifying the actual code, rejecting fabricated or out-of-scope claims, and mapping valid issues to the correct bounty severity.

## Repositories

Match the report's target to the right local repo before reading code.

**BNB Smart Chain / BNB Beacon Chain**
- `./bsc` — BSC Go node (Parlia consensus, EVM, p2p, RPC)
- `./bsc-genesis-contract` — system contracts (StakeHub, Governance, SlashIndicator, cross-chain)

**BNB Greenfield**
- `./greenfield` — Greenfield chain client / SDK code
- `./greenfield-cosmos-sdk` — Cosmos SDK fork used by Greenfield
- `./greenfield-contracts` — Greenfield bridge and resource-mirror contracts
- `./greenfield-storage-provider` — Storage Provider services and gRPC handlers

**opBNB**
- `./opbnb` — opBNB rollup node
- `./op-geth` — op-geth execution client for opBNB

**Threshold Signing**
- `./tss-lib` — threshold-signing library used by BNB Chain systems

All paths are relative to the repo root where Claude/Codex is running. If a repo is missing locally, say so and lower confidence rather than hallucinating.

Always read the actual source. Never trust reporter snippets, quoted line numbers, or PoC claims without checking the live code path yourself. When a report quotes a specific constant, sentinel value, or magic ID, verify the exact bytes/hex in source — reporters frequently get these wrong, which makes their PoC non-functional even if the underlying mechanism exists.

**Greenfield oracle module:** For reports in `x/oracle/keeper/`, trace the full call chain. `Claim -> CheckClaim -> IsRelayerValid -> getInturnRelayer` has early exits. Distinguish consensus-critical transaction paths from gRPC query handlers with panic recovery.

**Sometimes-mentioned repo:** `bsc-relayer` shows up in submissions. Verify current bounty scope first. Architecturally it relays messages between the Cosmos version of BNB Chain and BSC; claims that it can arbitrarily "mint tokens" are a common fabrication pattern.

---

## Step 1 — Scope and plausibility

**Out of scope — reject immediately:**
- BNB Chain websites, DNS, email, or other infrastructure
- Social engineering or physical attacks
- Pure volumetric/network flooding without a code defect
- Bugs that only affect outdated/unpatched client devices or browsers
- Local operator tooling behaving as designed, including CLI tools that write to user-chosen output paths
- Standard mempool front-running of public reward-bearing functions (for example evidence submission rewards). This is a public-blockchain property, not a contract vulnerability.

**Check current program scope before deep review.** If the report targets a repo or service outside the current bounty program, reject on scope. If the same submission is also architecturally impossible, say both.

**Latent / governance-gated issues:** If the bug only triggers after a privileged governance/admin parameter changes from a currently safe value, default to **REJECTED** unless:
- A single actor can force the change trivially, or
- The current configuration is already near the unsafe threshold in a realistic way

**Required fields:** Chain, attack scenario, impact, affected components, and reproduction steps. Missing suggested fix is fine. Missing fields do not automatically kill the report if enough technical detail exists to verify it.

**Fabrication / AI-generated report clues:**
- The described functions, parameters, or files do not exist
- The component is fundamentally misunderstood
- Reproduction steps are generic and not tied to the claimed code path
- The report claims races in deterministic state machines without proving nondeterminism
- The PoC uses mocks or patched test harnesses that bypass the real validation path
- The report asserts severe impact while never proving the required network boundary, auth bypass, or deployment condition

**Reachability matters:** For admin, HTTP, WebSocket, gRPC, signer, or Storage Provider findings, a bug in an internal handler is not enough. Verify whether the route is actually public, whether auth is enforced server-side, and whether the deployed listener topology exposes it.

**Admin-RPC-gated findings:** BSC's `admin` namespace is available via local IPC by default but is NOT included in default HTTP modules (`["net", "web3"]`) or WS modules. Reports that assume `admin_*` methods are remotely accessible are describing operator misconfiguration, not a BSC vulnerability. If an attacker already has admin RPC access, they have far stronger primitives available (`admin_addPeer`, `admin_addTrustedPeer`, `admin_startHTTP`, chain import/export) — evaluate whether the reported "vulnerability" adds any capability beyond what admin access already grants.

---

## Step 2 — Verify the code

Read the referenced files and enough surrounding context to understand:
- The exact function and line range
- Its callers and downstream consumers
- Existing checks the reporter may have missed
- Whether the claimed trigger and impact are reachable in current code

**Always verify preconditions are reachable.** Common false positives:
- **Empty validator sets:** CometBFT/BSC cannot keep producing blocks with zero validators. A path that requires an empty active validator set is generally unreachable in live consensus.
- **Uninitialized system contracts:** Post-fork code does not run before block-1 initialization.
- **Corrupted trusted on-chain state:** If arbitrary chain state corruption is assumed, the whole chain is already compromised. This is not a useful application-level threat model. For precompile findings, trace who supplies each input: if the "trusted" input comes from on-chain contract storage (e.g., `GnfdLightClient.consensusStateBytes`) and only the system contract can update it, an attacker calling the precompile directly with crafted data only affects their own context — not the bridge's state. The code defect may still be real (INFORMATIONAL) but the exploit path through the deployed bridge is not.
- **Genesis misconfiguration:** Bad genesis data usually fails during initialization, not as a delayed exploitable state.
- **Early-exit guards:** Many Greenfield/BSC paths return errors before the claimed vulnerable line is reached.

### Step 2b — Verify specific claim categories

**Attack timing:** If the report says the attacker must "wait X days" or "only after rotation", confirm the relevant expiration/timelock is actually consulted on the auth path.

**Fix safety:** If the suggested fix deletes mappings or state, check whether other live paths still depend on that data.

**Do not accept the reporter's characterization of data structures.** When a report calls something a "ban list" or "security mechanism," trace what actually writes to it and what reads from it. For example, `disconnectEnodeSet` in BSC's P2P server is not an automatic malicious-peer ban list — entries are only added when the local node itself issues `DiscRequested` during `RemovePeer`. It is an operator-managed reconnect suppression set. Mislabeling a data structure inflates perceived impact. Always describe what the code does, not what the reporter says it does.

**Verify PoC constants match source.** If a PoC uses a specific magic value, enode ID, address, or sentinel, check the exact value in source. A PoC that uses the wrong constant (e.g., all-`0xFF` when the actual sentinel is a different hash) is non-functional regardless of whether the underlying mechanism exists.

**Related-pattern claims:** Similar-looking stale mappings or helper functions are only the same vulnerability if there is a live caller that uses them today.

**Backport / upstream-fix reports:** A cited upstream patch is not enough. Verify the vulnerable code is still present in BNB's branch/tag and that the upstream exploit path really applies to this fork's reachable code.

**Validator-set math:** Reports often confuse `maxElectedValidators` with the actual consensus validator set:
- `snap.Validators` comes from `getCurrentValidators()` / `getMiningValidators()`
- `numOfCabinets` controls the mining set used by Parlia consensus
- `maxElectedValidators` is not automatically the same thing as the active consensus set

**Fork-dependent parameters:** Do not use genesis defaults for current-state analysis.
- `TurnLength` changed from 1 to 8 and then to **16 at Maxwell** on June 30, 2025
- Current BSC examples should use `numOfCabinets = 21` and `maxElectedValidators = 45` unless the code proves otherwise

**Governance-control rhetoric:** Do not claim "a colluding majority already controls the network" unless you actually verified quorum and threshold parameters.

**Cross-chain package typing and atomicity:**
- If a report claims the wrong op-type or deserialize type, trace the real decode/dispatch/ACK path
- Check whether the mismatch changes handler routing or only event metadata
- For multi-message / batched execution, verify whether non-crash errors are propagated or silently dropped
- Compare request count vs ACK count and confirm whether sequence advancement removes retry paths

**Cross-chain sequence fields:** In Greenfield, first ask whether downstream apps actually use `appCtx.Sequence` for replay protection, ordering, or state changes. If it is only emitted in events and replay protection already happens upstream, the bug is informational or rejected.

**Verifier return values:** If a contract or service treats "non-empty bytes" as success, check whether it decodes the actual boolean result. Then ask whether the attacker can reach that failure mode in the deployed verifier, rather than only in a patched/mock environment.

**Public signer / key-exposure paths:**
- For SP and SDK findings, verify the network boundary first
- Confirm whether the endpoint returns a real cryptographically valid signature, approval, mnemonic, or private-key material
- For file-read bugs, verify that caller-controlled metadata actually reaches filesystem access and that the service is reachable on a public or attacker-controlled path

**TSS / cryptographic findings:** Separate these classes carefully:
- Direct key exfiltration or signature forgery
- Cross-session transcript reuse / missing domain separation
- Ceremony DoS via panic or nil dereference
- Correctness bugs that return wrong values without compromising keys

Do **not** accept P* / P1 key-theft claims unless the PoC actually demonstrates key recovery or signature forgery under the library's real equations and checks. A session-binding defect without direct secret extraction is usually lower severity.

**System transaction nonce handling:** In BSC, a block with a failed system transaction is invalid and rejected. That is not an irreparable consensus split by itself.

**EVM state snapshot revert on error.** When a report claims "partial state changes persist after out-of-gas," check the outer call context. `evm.Call` / `StaticCall` / `DelegateCall` take a state snapshot before execution (`evm.go:243`) and revert on any error (`evm.go:314`). An error return from `Run()` (including ErrOutOfGas) causes the entire call's state changes to be rolled back. "Partial state persists" is only true if `Run()` returns `nil` error (clean exit), not if it returns an error.

**Opcode optimizer fallback control flow.** For super-instruction reports: when the fused `constantGas` equals the sum of individual sub-opcode costs (which is true for all current super-instructions in `jump_table.go`), the fallback executing sub-opcodes sequentially will always hit the same gas boundary and return an error — the successful-nil-return path is unreachable. However, the `return nil, err` at `interpreter.go:274/315` instead of `continue` is a real control-flow defect (the comment at `interpreter_si.go:109` says "shall continue in main loop"). Use REJECTED (inaccurate as submitted) — not flat REJECTED — because the code bug exists but the reporter's trigger doesn't work on current code. Tell the reporter what they need to prove: a concrete super-instruction where fallback returns nil.

---

## Step 2c — Recognize by-design behavior

Before accepting a report, compare it against known intended behaviors:

**Epoch snapshots are consensus authority.** Parlia snapshots are not a stale cache. Mid-epoch StakeHub changes take effect at the next epoch boundary by design.

**System transaction ordering is intentional.** In BSC `Finalize()`, common transactions execute before system transactions. Reports framing this as a race usually describe intended protocol behavior.

**Transfer gas limits are a deliberate trade-off, but compatibility gaps are real.** `transferGasLimit` in StakeCredit/StakeHub (default 5000, governance range 2300-10000) is intentional anti-griefing. However, do not flat-REJECT reports about contract wallets being unable to claim. The key questions are: (1) Can an external attacker force this state on a victim? No — the delegator is always `msg.sender` in `StakeHub.delegate/undelegate/claim`, so this is self-selected. (2) Does failed `claim()` burn funds? No — StakeCredit.claim reverts on transfer failure, rolling back queue pops; BNB stays in StakeCredit, not lost. (3) Is "permanent" proven? Only for immutable contracts exceeding the max cap; the reporter must cite an actual deployed wallet on BSC. This pattern is typically INFORMATIONAL (real compatibility limitation, no attacker-driven impact) rather than REJECTED (which implies no code issue exists).

**Felony slash quota is deliberate.** `maxFelonyBetweenBreatheBlock` is a governance-tunable safety valve to prevent mass slashing from client bugs.

**Maintenance mode is a trade-off.** `enterMaintenance()` and associated slash/jail thresholds are proportional-downtime design choices, not automatically bypasses.

**Deprecated / dead contracts matter.** If the vulnerable path depends on deprecated contracts or permanently dead state, the issue may be a real code smell but not a live vulnerability. TokenManager / unbound-token recovery is the canonical example.

**Ethereum features disabled on BSC via `IsNotInBSC()`.** BSC adopts some Ethereum hard-fork EVM changes but intentionally disables Beacon-specific machinery. EIP-6110/7002/7251 request processing (deposits, withdrawal queue, consolidation queue) is gated by `config.IsNotInBSC()` in `state_processor.go` and `miner/worker.go`. On BSC, `IsNotInBSC()` returns `false` (Parlia != nil), so request collection is skipped, `RequestsHash` is always `EmptyRequestsHash`, and BSC block bodies do not carry a requests list. Reports claiming request injection or `VerifyRequests` no-op attacks must account for this: the injection path does not exist on BSC. However, a no-op verifier for a field that appears in headers is a real code gap (INFORMATIONAL), even if it has no current security impact.

---

## Step 3 — Assess real-world impact

Once the bug is real, answer:
- Who can trigger it?
- What concrete asset or property is affected?
- Is the path reachable on today's deployed configuration?
- What downstream component actually consumes the corrupted data?

**Self-inflicted vs attacker-driven.** A critical severity distinction: can an external attacker force the harmful state on a victim, or does the affected party put themselves in that position? If the "victim" is always `msg.sender` choosing to interact from an incompatible contract, that is a self-selected compatibility issue — typically INFORMATIONAL, not a bounty-severity vulnerability. Check who controls the input address in the affected call path. Also check whether failure reverts cleanly (funds safe but stuck) vs silently commits (funds lost). Revert-on-failure is much less severe than silent loss.

**Trace downstream consumers.** A bug in helper `X` matters only if some security-relevant path reads `X`'s output.

**RPC / P2P / DoS findings:**
- Check body limits such as `httpBodyLimit`
- Check rate limits like `receiveRateLimitPerSecond`
- Check whether the workload is isolated to a single goroutine or non-consensus subsystem
- Check real deployment topology: validators usually are not public RPC nodes

**Greenfield endpoint findings:**
- Verify whether the route is public or private
- Verify whether auth is missing server-side, not only missing client-side
- Verify whether the response contains a real on-chain-verifiable signature or sensitive bytes

**Cross-chain batch / ACK findings:** If earlier submessages commit while later non-crash failures suppress ACKs and retries are impossible, fund lock or permanent state divergence may be real. If atomic commit/rollback or upstream sequence validation already prevents that, downgrade or reject.

**TSS findings:** Map severity to actual consequence:
- Real key recovery / signature forgery
- Remote or cross-party secret leakage
- Ceremony-wide crash / liveness failure
- Monitoring noise / wrong return values / dead-code hardening only

---

## Step 4 — Assign severity

| Rating | Criteria |
|--------|----------|
| **P\*** | Validator selection set manipulation; Merkle proof validation vulnerabilities; remote leaks of unencrypted private keys / mnemonic / key seed |
| **P1** | Fund/fee safety; severe token-economy disruption; RCE; key generation / signing / verification vulnerabilities; governance disruption; irreparable consensus splits |
| **P2** | Validator DoS; consensus-result/performance disruption; access of disabled cross-chain channels; DoS of cross-chain communication |
| **P3** | Explorer / relayer / oracle-relayer DoS |
| **P4** | Stability / availability issues in non-critical functions |

When severity is ambiguous, prefer the more conservative rating unless the stronger impact is concretely proven.

**Configuration-gated issues:** A consensus bug behind a default-disabled experimental flag is usually lower severity until there is proof production nodes enable it.

**Useful precedents from past reviews:**
- P4: `eth_getProof` key-array resource exhaustion
- P4: WebSocket oversized message handling
- P4: Blob DA bypass via `ChasingHead` poisoning
- P4: Remote incremental snapshot path traversal in opt-in remote snapshot mode
- P3: Constant-gas light-client precompile doing O(n) signature verification
- P3: Configuration-gated opcode-optimizer consensus divergence
- INFORMATIONAL: Real code defect with caller-level or on-chain backstop (for example the self-comparison typo in finality slash evidence construction, or `VerifyRequests` no-op where BSC disables the entire request pathway via `IsNotInBSC()` but the verifier gap still exists in the code, or `StakeCredit.claim()` gas-limit compatibility gap where no attacker can force the state and failure reverts cleanly)
- REJECTED: Standard blockchain properties, unreachable preconditions, dead/deprecated code paths, trusted-state SDK patterns, fabricated architecture
- REJECTED: Admin-RPC-gated findings where the prerequisite is operator misconfiguration and the "vulnerability" adds no capability beyond what admin access already grants (e.g., P2P disconnect-set reset via `magicEnodeID` through `admin_removePeer`)

**High-severity classes that require proof, not rhetoric:**
- Public endpoints returning live cryptographic approvals without auth
- Remotely reachable plaintext mnemonic/private-key exposure
- `tss-lib` issues that actually recover keys or forge signatures

---

## Step 5 — Verdicts and output

There are four verdicts:

- **`P<N>`** — valid bug with qualifying bounty severity
- **`INFORMATIONAL`** — real code defect or hardening gap, but no qualifying security impact in current code/deployment
- **`REJECTED`** — not a live vulnerability (false positive, by-design, out of scope, latent+safe, dead code path, etc.)
- **`REJECTED (inaccurate as submitted)`** — real underlying issue, but the trigger analysis / parameter math / impact story is materially wrong

Use this format for **accepted** reports:

```text
Verdict — P<N>  Valid bug (<severity label>). <One sentence describing what the attacker can actually do.>

<Report title> (P<N> Assessment)

What the bug does: <1-2 plain-language sentences>

Why P<N> and not higher:
1. <Factor> — <explanation>
2. <Factor> — <explanation>

Best-fit criterion: "<exact criterion text>" — <why it fits>
```

Severity labels: Critical (P*), High (P1), Medium-High (P2), Medium (P3), Low-Medium (P4).

Use this format for **informational** findings:

```text
Verdict — INFORMATIONAL  Real code defect, but no qualifying security impact in the current code path or deployment.

<Report title> (Informational)

What the code issue is: <1-2 sentences>

Why informational:
1. <Mitigation or backstop> — <explanation>
2. <No exploitable downstream impact> — <explanation>
```

Use this format for **rejections**:

```text
Verdict — REJECTED  <One sentence saying why.>

<Report title> (Rejected)

Why rejected:
1. <Reason> — <explanation>
2. <Reason> — <explanation>
```

Use this format for **inaccurate submissions**:

```text
Verdict — REJECTED (inaccurate as submitted)  Valid underlying bug, but the report's trigger analysis or impact story is materially wrong.

<Report title> (Rejected — needs revision)

What the bug does: <1-2 sentences on the real issue>

What needs to be corrected:
1. <Specific error> — <correct analysis>
2. <Specific error> — <correct analysis>

What the reporter must fix to be accepted: <one sentence>
```

**When to choose `INFORMATIONAL` vs `REJECTED`:**
- Use **INFORMATIONAL** when the code defect is real in current code, but the only impact is hardening / monitoring noise / redundant guard failure. Key signal: the reporter points at a real code gap (e.g., a no-op verifier, an unchecked return value, an unsafe integer cast) but the attack path they describe doesn't exist because the data never reaches the vulnerable point on BSC's actual execution path. The code gap is still worth noting. When a reporter honestly self-identifies non-exploitability and frames their finding as defense-in-depth, acknowledge that framing — it's a sign of a high-quality submission and INFORMATIONAL is the appropriate verdict, not REJECTED.
- Use **REJECTED** when the supposed exploit does not exist in current code, depends on dead/deprecated state, is by design, is out of scope, or requires a currently safe governance/config change. Key signal: the reporter's code snippet is fabricated, the function doesn't exist, or the described behavior is intentional.
- **Don't over-reject.** If you find yourself dismissing a real code observation as "by design" when the truth is that the feature is simply disabled on BSC (not that the no-op was a deliberate design choice), INFORMATIONAL is more accurate than REJECTED. A validator being able to set arbitrary header metadata without validation is a real gap even if it has no current downstream impact.
- **Use REJECTED (inaccurate) when the code defect is real but the exploitation path is wrong.** If the reporter points at a genuine control-flow bug or code/comment contradiction, but their trigger scenario doesn't work on current code (e.g., gas math doesn't produce the claimed condition, or the error path reverts state contrary to the "partial state persists" claim), use REJECTED (inaccurate as submitted) and tell them exactly what to prove. Flat REJECTED is for when the bug itself doesn't exist.
