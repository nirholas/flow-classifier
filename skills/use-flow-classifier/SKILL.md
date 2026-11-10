---
name: use-flow-classifier
description: Build, test, integrate and deploy the FlowClassifierHook Uniswap v4 hook from the flow-classifier repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# FlowClassifierHook

Publishes, on-chain, how much of a pool's order flow arrives first in the block and how far it moves the price when it does. It changes nothing about the pool it measures.

Family: Order flow and MEV. Tags: mev, public-good, analytics, order-flow, oracle-free. License: Apache-2.0. Source: `src/hooks/FlowClassifierHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/flow-classifier && cd flow-classifier
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/FlowClassifierHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `afterSwap`, `beforeSwap`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring

This hook has no `configure` call. Its terms are constructor arguments, fixed at deployment:

```solidity
constructor(IPoolManager)
```

Edit the arguments in `script/Deploy.s.sol` before deploying, since the deployed address depends on them.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`InvalidPool()`.

## Limitation

First-in-block is a proxy, not a fact. A private-mempool arbitrage that lands second still leads economically, and an ordinary swap that happens to be first is counted as leading. The measure is meaningful in aggregate over many blocks and says nothing reliable about any single swap. On a chain with sub-second blocks where most blocks hold one swap, nearly everything leads and the ratio degenerates.

More: `docs/integrating.md`, `docs/deploying.md`, and https://flow-classifier.pages.dev.
