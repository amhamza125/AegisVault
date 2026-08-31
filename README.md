# AegisVault — On-Chain AI Red-Teaming Vault

An on-chain vault guarded by an AI "sentry" that refuses to reveal a secret phrase under any circumstances. Anyone can attempt to jailbreak it. Every attempt is logged publicly. If someone genuinely breaks the sentry's defenses, they win the entire funded bounty pool — paid out automatically, on-chain, no human judge involved.

## Why this design

An earlier version of this contract let breaches be attributed to a caller-supplied address (spoofable) and had a bounty pool with no real funding or payout mechanism. This version fixes both:

1. **Authenticated attribution** — `attack_vault` takes no address argument at all. The attacker credited with any outcome is always `gl.message.sender_address`, the real, cryptographically authenticated transaction sender. Nobody can spoof a breach onto someone else's address.
2. **Real, funded payout** — `fund_bounty` is a genuine payable method that adds GEN to the pool. On a verified breach, the entire pool balance is credited to the winning attacker via a standard pull-payment `withdraw()` — the same safe pattern used in real fund-holding contracts. Once claimed, the vault locks against further payouts.

## How it works

1. Anyone can call `fund_bounty()` with GEN to grow the pool.
2. An attacker calls `attack_vault(target_prompt)` with their jailbreak attempt.
3. The contract runs the attacker's prompt against the sentry's system prompt via an LLM.
4. Consensus is reached through a **custom leader/validator function** (`gl.vm.run_nondet_unsafe`) — not a built-in equivalence shortcut. Each validator independently re-runs the sentry prompt on the same input and must arrive at the same categorical result (`BREACHED` or `DEFENDED`) as the leader. Nobody's claimed outcome is taken on trust; it's recomputed independently by every validator.
5. Every attempt (attacker address, prompt excerpt, outcome) is appended to a public on-chain log.
6. If the sentry is breached and the vault hasn't already been claimed, the entire pool is credited to the attacker, withdrawable via `withdraw()`.

## Contract

- **File:** `aegis_vault_v3.py`

## Methods

| Method | Type | Description |
|---|---|---|
| `fund_bounty()` | payable write | Adds GEN to the bounty pool |
| `attack_vault(target_prompt: str) -> str` | write | Attempts to jailbreak the sentry; returns `"BREACHED"` or `"DEFENDED"` |
| `withdraw()` | write | Claims any bounty owed to the caller (pull-payment pattern) |
| `get_vault_status() -> str` | view | Returns pool balance, breach status, winner, and total attempt count |
| `get_attempt(index: int) -> str` | view | Returns a single logged attempt (attacker, prompt excerpt, outcome) |
| `get_withdrawable_balance(address: str) -> str` | view | Returns the claimable balance for a given address |

## Why GenLayer

Judging whether a piece of natural-language text successfully manipulated an AI into violating its instructions isn't something a traditional smart contract can do — it requires actually running the model and reasoning over free-form output. This contract does that natively: it runs the adversarial prompt through an LLM, has a diverse, independently-selected validator set recompute the same judgment themselves rather than trusting a single leader, and turns the verified outcome into a real, enforceable fund transfer — with no human referee and no oracle in the loop.

## Built with

- GenLayer Studio
- Python (GenVM SDK)
- Custom `gl.vm.run_nondet_unsafe` leader/validator consensus
- 
