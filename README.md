# AgentBudget

**A spending limit the venue itself enforces: an autonomous agent may swap on this pool only within a budget its principal signed, and the pool refuses the swap that would exceed it.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://agent-budget.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/AgentBudgetHook.sol`](src/hooks/AgentBudgetHook.sol)
- **Licence:** Apache-2.0

## How it works

Handing a key to an autonomous agent means choosing between two bad options. Approve a router for a small amount and the agent stalls the moment it needs more, at which point somebody has to be awake to raise it. Approve it for the usual unbounded amount and the only thing standing between a bug and the whole balance is the agent's own code, which is the thing you were trying not to trust.

Smart accounts answer this with a policy engine, which works and costs you a smart account: a migration, a new address, a new set of integrations, and a policy layer that every venue has to be taught about. This hook puts the limit somewhere neither of those touch, in the pool, where it applies to any wallet, any router and any account type, because it is enforced by the venue rather than by the spender. A principal signs one `Delegation`, off-chain and once: an agent address, a per-epoch cap in each of the pool's two currencies, an epoch length, an expiry.

The agent then signs each swap it makes under that delegation. On `afterSwap` the hook checks both signatures, measures what the swap actually spent from the balance delta, and reverts if that would take the agent past its cap for the current epoch. A revert in `afterSwap` unwinds the swap with it, so an over-budget trade cannot land.

Measuring in `afterSwap` rather than `beforeSwap` is deliberate and is what makes the cap honest. Before the swap the only figure available is `amountSpecified`, which on an exact-output swap says nothing about how much the swapper will actually pay; a budget checked against it would be trivially evaded by asking for an exact output and letting the input land wherever the curve puts it. After the swap the true spend is in the delta.

What this does not do, stated plainly, because the distinction matters: it never custodies funds, never moves a token, and grants no allowance. The agent still needs its own ERC-20 approval to trade at all. The hook only refuses to let this pool be the venue for a swap outside the budget.

An agent with an unbounded approval can still spend elsewhere, so this is a limit on a venue, not on a key, and it is worth exactly as much as the set of venues that enforce it. Both signatures are checked with ERC-1271 as well as ECDSA, so a principal or an agent may itself be a contract.

## Prior art

Per-agent policy engines exist in smart-account land (session keys, ERC-7710 delegations, module-based spending limits), and hooks that gate swaps on an allowlist or a credential are common. Enforcing a signed, per-epoch, per-currency spending cap inside the AMM, on behalf of an EOA principal with no smart account anywhere in the path, is the contribution here.

## Where it does not help

The cap binds this pool only. An agent holding an unbounded ERC-20 approval can spend the same funds on any venue that does not enforce the delegation, so this raises the cost of a compromised agent rather than bounding it absolutely. It also requires the caller to pass hookData, so an aggregator that strips it will simply be unable to trade the pool.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AuthorizationRequired()` | The swap carried no `hookData`, so there was no delegation to check it against. |
| `BadAgentSignature()` | The swap signature does not recover to the delegation's agent. |
| `BadPrincipalSignature()` | The delegation signature does not recover to the named principal. |
| `BudgetExceeded(uint8,uint128,uint256)` | The swap would take the agent past its budget for the epoch in progress. |
| `Expired()` | The delegation has expired, or the swap authorisation has. |
| `InvalidEpochLength()` | `epochLength` of zero would make every swap its own epoch, which is no budget at all. |
| `NonceAlreadyUsed(uint256)` | This nonce has already been spent under this delegation. |
| `Revoked()` | The principal has revoked this delegation. |
| `WrongPool()` | The delegation is for a different pool than the one being swapped. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 1 of the fourteen:

- `afterSwap`

Mask: `0x40`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # AgentBudget
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # agent, delegation, spending-limit, eip712, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/agent-budget
cd agent-budget
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
