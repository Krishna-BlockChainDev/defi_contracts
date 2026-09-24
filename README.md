# SushiSwap V2 & V3 — Contract Reference

Brief overview of all core contracts in this repository.

---

## V2 Core

### [`UniswapV2Factory`](v2-core/contracts/UniswapV2Factory.sol)

Deploys pair contracts using `CREATE2` with a salt of `keccak256(token0, token1)`. Tracks all pairs in a mapping and array. Manages `feeTo` (protocol fee recipient) and `feeToSetter`. One pair per token pair — no fee tiers.

### [`UniswapV2Pair`](v2-core/contracts/UniswapV2Pair.sol)

The core AMM pool. Holds `reserve0` and `reserve1`, enforces the constant product invariant `x * y = k`. Issues **SLP (SushiSwap LP)** tokens to liquidity providers. Key functions:

- `mint(to)` — add liquidity, issue LP tokens
- `burn(to)` — remove liquidity, redeem LP tokens
- `swap(amount0Out, amount1Out, to, data)` — execute swap (and flash swap if `data.length > 0`)
- `sync()` / `skim()` — force reserves to match balances or vice versa

Fee is a fixed **0.3%**. Protocol fee (1/6 of swap fee) is minted as LP tokens to `feeTo` when enabled.

### [`UniswapV2ERC20`](v2-core/contracts/UniswapV2ERC20.sol)

Standard ERC20 LP token (`SLP`) inherited by `UniswapV2Pair`. Implements EIP-712 `permit()` for gasless approvals.

### [`UniswapV2Router02`](v2-core/contracts/UniswapV2Router02.sol)

User-facing entry point. Wraps low-level pair calls with slippage checks, deadline enforcement, and ETH/WETH handling. Key functions:

- `addLiquidity` / `addLiquidityETH`
- `removeLiquidity` / `removeLiquidityETH` (+ permit variants)
- `swapExactTokensForTokens`, `swapTokensForExactTokens`, `swapExactETHForTokens`, etc.
- Fee-on-transfer token support variants

### [`UniswapV2Library`](v2-core/contracts/libraries/UniswapV2Library.sol)

Pure/view helpers used by the router: `sortTokens`, `pairFor` (computes pair address off-chain), `getReserves`, `quote`, `getAmountOut`, `getAmountIn`, `getAmountsOut`, `getAmountsIn`.

---

## V3 Core

### [`UniswapV3Factory`](v3-core/contracts/UniswapV3Factory.sol)

Deploys pools via `UniswapV3PoolDeployer`. Supports multiple fee tiers per token pair — default tiers are **0.05%** (tickSpacing 10), **0.3%** (tickSpacing 60), **1%** (tickSpacing 200). Owner can add new fee tiers. Pool lookup: `getPool[token0][token1][fee]`.

### [`UniswapV3PoolDeployer`](v3-core/contracts/UniswapV3PoolDeployer.sol)

Helper inherited by Factory. Transiently stores pool parameters (factory, token0, token1, fee, tickSpacing) in storage before deploying, then deletes them. The pool reads these in its constructor via `IUniswapV3PoolDeployer(msg.sender).parameters()`.

### [`UniswapV3Pool`](v3-core/contracts/UniswapV3Pool.sol)

The core concentrated liquidity pool. State is packed into `slot0` (sqrtPrice, tick, oracle index, fee protocol, lock). Key functions:

- `initialize(sqrtPriceX96)` — set initial price, must be called once before any action
- `mint(recipient, tickLower, tickUpper, amount, data)` — add liquidity to a tick range; calls `IUniswapV3MintCallback` on the caller to pull tokens
- `burn(tickLower, tickUpper, amount)` — remove liquidity, credit tokens to `tokensOwed`
- `collect(recipient, tickLower, tickUpper, ...)` — withdraw `tokensOwed` (principal + fees) separately from burn
- `swap(recipient, zeroForOne, amountSpecified, sqrtPriceLimitX96, data)` — tick-by-tick swap loop; calls `IUniswapV3SwapCallback` on caller to pull input
- `flash(recipient, amount0, amount1, data)` — flash loan of any amount; calls `IUniswapV3FlashCallback`; repayment must include pool fee
- `collectProtocol(...)` — factory owner withdraws accumulated protocol fees
- `observe(secondsAgos[])` — read historical TWAP data from oracle ring buffer

### Key Libraries

| Library                                                          | Purpose                                                                                      |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| [`Tick`](v3-core/contracts/libraries/Tick.sol)                   | Per-tick state, fee growth outside, `cross()` on tick transition                             |
| [`TickBitmap`](v3-core/contracts/libraries/TickBitmap.sol)       | Packed bitmap; efficiently finds next initialized tick                                       |
| [`Position`](v3-core/contracts/libraries/Position.sol)           | Per-LP position: liquidity, fee growth snapshot, tokens owed                                 |
| [`Oracle`](v3-core/contracts/libraries/Oracle.sol)               | Circular buffer of up to 65535 `(tickCumulative, secondsPerLiquidity)` observations for TWAP |
| [`SwapMath`](v3-core/contracts/libraries/SwapMath.sol)           | Computes `amountIn`, `amountOut`, `feeAmount` for one swap step within a tick boundary       |
| [`SqrtPriceMath`](v3-core/contracts/libraries/SqrtPriceMath.sol) | Token amounts from sqrtPrice deltas                                                          |
| [`TickMath`](v3-core/contracts/libraries/TickMath.sol)           | `getSqrtRatioAtTick` and `getTickAtSqrtRatio`                                                |
| [`FullMath`](v3-core/contracts/libraries/FullMath.sol)           | 512-bit multiplication with 256-bit result (no overflow)                                     |

### Callback Interfaces

| Interface                                                                                      | Called by         | Implementor must                                         |
| ---------------------------------------------------------------------------------------------- | ----------------- | -------------------------------------------------------- |
| [`IUniswapV3MintCallback`](v3-core/contracts/interfaces/callback/IUniswapV3MintCallback.sol)   | Pool on `mint()`  | Transfer `amount0Owed` + `amount1Owed` of tokens to pool |
| [`IUniswapV3SwapCallback`](v3-core/contracts/interfaces/callback/IUniswapV3SwapCallback.sol)   | Pool on `swap()`  | Transfer the positive delta token amount to pool         |
| [`IUniswapV3FlashCallback`](v3-core/contracts/interfaces/callback/IUniswapV3FlashCallback.sol) | Pool on `flash()` | Repay borrowed amounts plus `fee0` / `fee1`              |

---

## V2 vs V3 at a Glance

|                  | V2                     | V3                                        |
| ---------------- | ---------------------- | ----------------------------------------- |
| Liquidity range  | Full range (0 → ∞)     | Concentrated (`tickLower` → `tickUpper`)  |
| Fee tiers        | Fixed 0.3%             | 0.05% / 0.3% / 1%                         |
| LP token         | Fungible ERC20 (SLP)   | Non-fungible position (NFT via periphery) |
| Price storage    | `reserve0`, `reserve1` | `sqrtPriceX96` (Q64.96)                   |
| Pools per pair   | 1                      | Many (one per fee tier)                   |
| Flash loan       | Via `swap()` callback  | Dedicated `flash()` + callback            |
| Remove liquidity | Single `burn()`        | `burn()` + `collect()` (two steps)        |
| Oracle           | Cumulative price       | Ring buffer TWAP (up to 65535 points)     |

> For detailed flow diagrams and step-by-step sequences, see [`contract-flows.md`](contract-flows.md).
