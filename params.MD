# Universal Router V1 — Overview

## Address and Scope

- Mainnet address: `0x3fC91A3afd70395Cd496C647d5a6CC9D4B2b7FAD`
- Non-upgradable, unowned, immutable code. Deployed across multiple chains (same address where possible).

## Purpose

- Unify ERC-20 swap and payment operations into a single atomic transaction.
- Batch complex flows (e.g., Permit2 pull → swap → unwrap → sweep) to reduce gas and UX friction.
- Integrate Permit2 for safer approvals without granting allowances to the router.

## Capabilities

- Swaps: Uniswap v2 and v3 swaps; can split and interleave routes.
- Payments: ERC-20 pulls and transfers, native ETH ↔ WETH wrap/unwrap, split payments.
- Safety and cleanup: SWEEP for ERC-20, deadline expiry.
- Partial reverts: Per-command “allow revert” flag enables graceful partial fills.

Note: This repo removes all NFT marketplace and NFT transfer commands from Universal Router.

## Command Model

- Entry: `execute(commands, inputs, deadline)` processes a sequence of 1‑byte commands.
- Byte format: `f r cccccc` → 1 bit allow‑revert flag, 1 reserved bit, 6‑bit command ID (masked by `0x3f`).
- Inputs: Each command consumes one ABI‑encoded payload from `inputs`.
- Dispatch: A big if/else (jump table-like) calls the handler for each command ID.

Common input shapes:

- V3 swap: `(recipient, amountIn, minAmountOut, pathBytes, payerIsUser)`
- Permit2 transfer: `(token, recipient, amount)`
- Sweep: `(token, recipient, amount)`

## Token Flow

- Permit2: Router pulls tokens from `msg.sender` via signed permits or allowance transfers; users don’t set allowances on the router itself.
- Routing funds: Outputs accumulate in the router unless directed; final steps typically sweep/transfer to recipient.
- Placeholder recipients: Special markers (e.g., “msg.sender”) are mapped at runtime to the actual user or to the router for chaining.

## Control and Nesting

- Allow‑revert flag: If set, the router catches a failure for that command and continues execution (user must sweep leftovers as needed).
- Sub‑plans: `EXECUTE_SUB_PLAN (0x21)` executes a nested batch (internal self‑call bypasses the lock and doesn’t reenter).

## Security

- Reentrancy lock: Added in v1.1.0 after a reported issue; `execute` is locked per caller to prevent mid-sequence reentry.
- Reserved bit enforcement: This repo rejects commands with the reserved bit set (0x40), preventing malformed inputs.

## Parameters (Mainnet)

These are set immutably at deploy; see `params.json` for canonical values used in this repo’s build.

- Permit2: `0x000000000022D473030F116dDEE9F6B43aC78BA3`
- WETH9: `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`
- Uniswap v2 factory: `0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f`
- Uniswap v3 factory: `0x1F98431c8aD98523631AE4a59f267346ea31F984`
- v2 pair init code hash: `0x96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f`
- v3 pool init code hash: `0xe34f199b19b2b4f47f68442619d555527d244f78a3297ea89325f843f87b8b54`

## Usage Patterns

- Pull tokens via Permit2 → swap(s) → sweep leftovers to `msg.sender`.
- Use allow-revert on external calls you’re willing to skip, then sweep unused funds to avoid stranding assets.
- Prefer explicit recipients in swaps to keep end-state clean without extra sweeps.

## Security Considerations

- Use Deadline: Prefer `execute(..., deadline)` to bound price/MEV risk.
- Always Sweep: End scripts with `SWEEP` to avoid residual balances.
- Recipient Hygiene: Provide explicit recipients to minimize in-contract custody.

Hardening implemented in this repo:

- Param Validate: Constructor enforces non-zero critical addresses and code presence for key contracts; non-zero v2/v3 init code hashes.
- Reserved Bit Check: Dispatch rejects any command with the 0x40 reserved bit set.

## Notes and Limitations

- Reserved bits must be zero; unknown command IDs revert.
- Router code is immutable; adding new integrations requires a new deployment.
- Universal Router v1 does not route Uniswap v4 pools; v4 has dedicated routers.
