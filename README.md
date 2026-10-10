# FlowClassifier

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/flow-classifier?style=flat&logo=github)](https://github.com/nirholas/flow-classifier/stargazers) [![License](https://img.shields.io/github/license/nirholas/flow-classifier?style=flat)](https://github.com/nirholas/flow-classifier/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/flow-classifier?style=flat)](https://github.com/nirholas/flow-classifier/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/flow-classifier/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/flow-classifier/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Publishes, on-chain, how much of a pool's order flow arrives first in the block and how far it moves the price when it does. It changes nothing about the pool it measures.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://flow-classifier.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/FlowClassifierHook.sol`](src/hooks/FlowClassifierHook.sol)
- **Licence:** Apache-2.0

## How it works

Almost every fee mechanism in this catalogue is a guess about who is trading. The staleness tax guesses from elapsed time. The priority-fee tax guesses from what a trade bid for its position.

Each one re-derives the same hidden quantity privately, none of them publishes it, and no two of them agree. The quantity they are all reaching for has a clean on-chain signature. Arbitrage arrives first in its block, because being second is worthless when the whole trade is closing a gap somebody else can close instead.

Ordinary flow does not care where in a block it lands. So the split between swaps that led their block and swaps that followed one is a measurable proxy for the split between informed and uninformed flow, and the ratio of how far each population moves the price is a measure of how expensive the informed half is. This hook counts both and publishes them through {IFlowStats}: - `leadShareBps`: the fraction of swaps that were first in their block.

- `leadImpactRatioBps`: how much further the average leading swap moves the price than the average following one. `10_000` means they move it equally. A pool being arbitraged reads well above that.

The point is that it is a public good rather than a mechanism. It sets no fee, returns no delta, takes no payment and rejects nothing; a test asserts that a swap through a measured pool receives exactly what the same swap through an identical unhooked pool receives. Other hooks can price from it, routers can prefer pools whose flow is cheap to fill, providers can decide whether a pool is worth quoting, and indexers can rank pools by something more meaningful than volume.

Counters saturate rather than wrap. A wrapped counter reports a small number where a huge one belongs and every ratio derived from it becomes quietly wrong; a saturated one stops moving and keeps the last true value, which is a failure a reader can notice.

## Prior art

Off-chain, order-flow toxicity is standard: VPIN, markout, and the lead-lag analysis every market maker runs on its own fills. On-chain, hooks consume such signals privately to set a fee. Publishing the measurement itself, from a hook that deliberately does nothing else, so that every other contract can read one pool's flow quality instead of each guessing at it, is the contribution here.

## Where it does not help

First-in-block is a proxy, not a fact. A private-mempool arbitrage that lands second still leads economically, and an ordinary swap that happens to be first is counted as leading. The measure is meaningful in aggregate over many blocks and says nothing reliable about any single swap. On a chain with sub-second blocks where most blocks hold one swap, nearly everything leads and the ratio degenerates.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.


## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `beforeSwap`
- `afterSwap`

Mask: `0xc0`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # FlowClassifier
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, public-good, analytics, order-flow, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/flow-classifier
cd flow-classifier
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

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/flow-classifier&type=Date)](https://www.star-history.com/#nirholas/flow-classifier&Date)

<!-- three.ws:growth -->
## Support the project

If flow-classifier saves you time, **[star it on GitHub](https://github.com/nirholas/flow-classifier)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=flow-classifier%3A%20Publishes%2C%20on-chain%2C%20how%20much%20of%20a%20pool's%20order%20flow%20arrives%20first%20in%20the%20block%20and%20how%20far%20it&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fflow-classifier) · [Share on Bluesky](https://bsky.app/intent/compose?text=flow-classifier%3A%20Publishes%2C%20on-chain%2C%20how%20much%20of%20a%20pool's%20order%20flow%20arrives%20first%20in%20the%20block%20and%20how%20far%20it%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fflow-classifier) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fflow-classifier) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fflow-classifier&t=flow-classifier%3A%20Publishes%2C%20on-chain%2C%20how%20much%20of%20a%20pool's%20order%20flow%20arrives%20first%20in%20the%20block%20and%20how%20far%20it) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fflow-classifier&title=flow-classifier%3A%20Publishes%2C%20on-chain%2C%20how%20much%20of%20a%20pool's%20order%20flow%20arrives%20first%20in%20the%20block%20and%20how%20far%20it)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/flow-classifier` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/flow-classifier/issues) or [start a discussion](https://github.com/nirholas/flow-classifier/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/flow-classifier)](https://github.com/nirholas/flow-classifier/graphs/contributors)

<!-- /three.ws:growth -->
