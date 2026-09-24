# SushiSwap / Uniswap V2 & V3 — Contract Flows

> Minimal, essential reference for understanding how V2 and V3 AMM contracts work, their key functions, and how they interact.

---

## Table of Contents

1. [V2 Architecture Overview](#v2-architecture-overview)
2. [V2 Contract Flows](#v2-contract-flows)
   - [Pair Creation](#v2-pair-creation)
   - [Add Liquidity (Mint)](#v2-add-liquidity-mint)
   - [Remove Liquidity (Burn)](#v2-remove-liquidity-burn)
   - [Swap](#v2-swap)
   - [Flash Swap](#v2-flash-swap)
3. [V3 Architecture Overview](#v3-architecture-overview)
4. [V3 Contract Flows](#v3-contract-flows)
   - [Pool Creation](#v3-pool-creation)
   - [Initialize Pool](#v3-initialize-pool)
   - [Add Liquidity (Mint)](#v3-add-liquidity-mint)
   - [Remove Liquidity (Burn + Collect)](#v3-remove-liquidity-burn--collect)
   - [Swap](#v3-swap)
   - [Flash Loan](#v3-flash-loan)
5. [V2 vs V3 — Key Differences](#v2-vs-v3--key-differences)
6. [Key Formulas](#key-formulas)

---

## V2 Architecture Overview

```mermaid
graph TD
    Router[UniswapV2Router02]
    Factory[UniswapV2Factory]
    Pair[UniswapV2Pair]
    ERC20[UniswapV2ERC20 - SLP Token]
    Library[UniswapV2Library]

    Router --> Factory
    Router --> Library
    Router --> Pair
    Factory --> Pair
    Pair --> ERC20
```

### Contracts

| Contract                                                               | Role                                                                    |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| [`UniswapV2Factory`](v2-core/contracts/UniswapV2Factory.sol)           | Deploys and tracks all pair contracts via `CREATE2`                     |
| [`UniswapV2Pair`](v2-core/contracts/UniswapV2Pair.sol)                 | Core AMM pool — holds reserves, issues SLP tokens                       |
| [`UniswapV2ERC20`](v2-core/contracts/UniswapV2ERC20.sol)               | LP token logic (SLP), supports EIP-712 `permit`                         |
| [`UniswapV2Router02`](v2-core/contracts/UniswapV2Router02.sol)         | User-facing entry point — wraps low-level pair calls with safety checks |
| [`UniswapV2Library`](v2-core/contracts/libraries/UniswapV2Library.sol) | Pure helper: `getAmountOut`, `getAmountIn`, `pairFor`, `quote`          |

---

## V2 Contract Flows

### V2 Pair Creation

```mermaid
sequenceDiagram
    participant User
    participant Router
    participant Factory
    participant Pair

    User->>Router: addLiquidity(tokenA, tokenB, ...)
    Router->>Factory: getPair(tokenA, tokenB)
    Factory-->>Router: address(0) — pair not found
    Router->>Factory: createPair(tokenA, tokenB)
    Factory->>Factory: sort tokens (token0 < token1)
    Factory->>Pair: CREATE2 deploy UniswapV2Pair
    Factory->>Pair: initialize(token0, token1)
    Factory-->>Router: pair address
```

**Key points:**

- `CREATE2` salt = `keccak256(token0, token1)` — deterministic address
- Pair address computable off-chain using `pairFor()` in the library
- One pair per `(token0, token1)` — no fee tiers in V2

---

### V2 Add Liquidity (Mint)

```mermaid
sequenceDiagram
    participant User
    participant Router
    participant Pair

    User->>Router: addLiquidity(tokenA, tokenB, amtADesired, amtBDesired, ...)
    Router->>Router: _addLiquidity() — compute optimal amounts
    Router->>Pair: transferFrom(tokenA, user→pair, amtA)
    Router->>Pair: transferFrom(tokenB, user→pair, amtB)
    Router->>Pair: mint(to)
    Pair->>Pair: read balances, compute deposited amounts
    Pair->>Pair: _mintFee() — if protocol fee on, mint to feeTo
    Pair->>Pair: calculate liquidity shares
    Pair->>Pair: _mint(to, liquidity) — issue SLP tokens
    Pair->>Pair: _update() — update reserves + price accumulators
    Pair-->>Router: liquidity (SLP amount)
    Router-->>User: (amtA, amtB, liquidity)
```

**First deposit special case:**

- `liquidity = sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY`
- `MINIMUM_LIQUIDITY` (1000 wei) permanently burned to `address(0)` — prevents price manipulation

**Subsequent deposits:**

- `liquidity = min(amount0 * totalSupply / reserve0, amount1 * totalSupply / reserve1)`

---

### V2 Remove Liquidity (Burn)

```mermaid
sequenceDiagram
    participant User
    participant Router
    participant Pair

    User->>Router: removeLiquidity(tokenA, tokenB, liquidity, ...)
    Router->>Pair: transferFrom(user→pair, liquidity SLP)
    Router->>Pair: burn(to)
    Pair->>Pair: read balances + liquidity held
    Pair->>Pair: _mintFee() — accrue any protocol fees first
    Pair->>Pair: amount0 = liquidity * balance0 / totalSupply
    Pair->>Pair: amount1 = liquidity * balance1 / totalSupply
    Pair->>Pair: _burn(pair, liquidity) — destroy SLP
    Pair->>Token0: safeTransfer(to, amount0)
    Pair->>Token1: safeTransfer(to, amount1)
    Pair->>Pair: _update() — update reserves
    Pair-->>Router: (amount0, amount1)
    Router->>Router: check >= amountMin
    Router-->>User: (amtA, amtB)
```

---

### V2 Swap

```mermaid
sequenceDiagram
    participant User
    participant Router
    participant Pair

    User->>Router: swapExactTokensForTokens(amtIn, amtOutMin, path[], to)
    Router->>Library: getAmountsOut(factory, amtIn, path)
    Library-->>Router: amounts[]
    Router->>Router: check amounts[last] >= amtOutMin
    Router->>Token0: safeTransferFrom(user→pair[0], amounts[0])
    Router->>Pair: _swap(amounts, path, to)
    loop for each hop in path
        Pair->>Pair: optimistically send amountOut to next pair or to
        Pair->>Pair: read new balances
        Pair->>Pair: verify K invariant: (balance0*1000 - amtIn*3) * (balance1*1000 - amtIn*3) >= reserve0*reserve1*1000^2
        Pair->>Pair: _update() reserves
    end
    Pair-->>User: output tokens
```

**Fee:** 0.3% (hardcoded) — `amountInWithFee = amountIn * 997`

**Constant Product:** `x * y = k` must hold (with fee adjustment)

**Formula:**

```
amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)
```

**Multi-hop:** Router chains pairs — output of pair[i] is sent directly to pair[i+1]

---

### V2 Flash Swap

```mermaid
sequenceDiagram
    participant Caller
    participant Pair

    Caller->>Pair: swap(amount0Out, amount1Out, to, data)
    Pair->>Pair: send tokens optimistically to `to`
    Pair->>Caller: uniswapV2Call(sender, amount0Out, amount1Out, data)
    Note over Caller: Execute arbitrage or other logic
    Caller->>Pair: repay (input token + 0.3% fee)
    Pair->>Pair: verify K invariant holds
```

- If `data.length > 0`, pair calls `IUniswapV2Callee(to).uniswapV2Call()` — enabling flash swaps
- Caller must repay before the transaction ends or it reverts

---

## V3 Architecture Overview

```mermaid
graph TD
    Factory[UniswapV3Factory]
    Deployer[UniswapV3PoolDeployer]
    Pool[UniswapV3Pool]
    Oracle[Oracle Library]
    Tick[Tick Library]
    TickBitmap[TickBitmap Library]
    Position[Position Library]
    SwapMath[SwapMath Library]
    SqrtPriceMath[SqrtPriceMath Library]
    TickMath[TickMath Library]
    MintCB[IUniswapV3MintCallback]
    SwapCB[IUniswapV3SwapCallback]
    FlashCB[IUniswapV3FlashCallback]

    Factory --> Deployer
    Deployer --> Pool
    Pool --> Oracle
    Pool --> Tick
    Pool --> TickBitmap
    Pool --> Position
    Pool --> SwapMath
    Pool --> SqrtPriceMath
    Pool --> TickMath
    Pool --> MintCB
    Pool --> SwapCB
    Pool --> FlashCB
```

### Contracts & Libraries

| Contract / Library                                                     | Role                                                   |
| ---------------------------------------------------------------------- | ------------------------------------------------------ |
| [`UniswapV3Factory`](v3-core/contracts/UniswapV3Factory.sol)           | Deploys pools, manages fee tiers and owner             |
| [`UniswapV3PoolDeployer`](v3-core/contracts/UniswapV3PoolDeployer.sol) | Transient parameter storage for pool constructor       |
| [`UniswapV3Pool`](v3-core/contracts/UniswapV3Pool.sol)                 | Core pool — concentrated liquidity, tick-based pricing |
| [`Tick`](v3-core/contracts/libraries/Tick.sol)                         | Per-tick state, fee growth tracking, cross logic       |
| [`TickBitmap`](v3-core/contracts/libraries/TickBitmap.sol)             | Efficient bitmap to find next initialized tick         |
| [`Position`](v3-core/contracts/libraries/Position.sol)                 | Per-position state (liquidity, fees owed)              |
| [`Oracle`](v3-core/contracts/libraries/Oracle.sol)                     | TWAP observations ring buffer                          |
| [`SwapMath`](v3-core/contracts/libraries/SwapMath.sol)                 | Computes step amounts within a tick range              |
| [`SqrtPriceMath`](v3-core/contracts/libraries/SqrtPriceMath.sol)       | Token amounts from sqrt price deltas                   |
| [`TickMath`](v3-core/contracts/libraries/TickMath.sol)                 | Convert tick ↔ sqrtPriceX96                            |

### Fee Tiers (default)

| Fee         | Tick Spacing | Use Case       |
| ----------- | ------------ | -------------- |
| 0.05% (500) | 10           | Stable pairs   |
| 0.3% (3000) | 60           | Standard pairs |
| 1% (10000)  | 200          | Exotic pairs   |

---

## V3 Contract Flows

### V3 Pool Creation

```mermaid
sequenceDiagram
    participant Anyone
    participant Factory
    participant Deployer
    participant Pool

    Anyone->>Factory: createPool(tokenA, tokenB, fee)
    Factory->>Factory: sort tokens, validate fee tier
    Factory->>Factory: check pool not already deployed
    Factory->>Deployer: deploy(factory, token0, token1, fee, tickSpacing)
    Deployer->>Deployer: store Parameters transiently
    Deployer->>Pool: new UniswapV3Pool (CREATE2, salt=keccak256(token0,token1,fee))
    Pool->>Deployer: read parameters() in constructor
    Deployer->>Deployer: delete parameters
    Factory->>Factory: store getPool[token0][token1][fee] = pool
    Factory-->>Anyone: pool address
```

**Key difference from V2:** Multiple pools per token pair — one per fee tier

---

### V3 Initialize Pool

```mermaid
sequenceDiagram
    participant Caller
    participant Pool

    Caller->>Pool: initialize(sqrtPriceX96)
    Pool->>Pool: require slot0.sqrtPriceX96 == 0 (not already init)
    Pool->>TickMath: getTickAtSqrtRatio(sqrtPriceX96) → tick
    Pool->>Oracle: initialize observation at current timestamp
    Pool->>Pool: set slot0 (price, tick, oracle index, unlocked=true)
    Pool-->>Caller: emits Initialize(sqrtPriceX96, tick)
```

- Must be called before any `mint` or `swap`
- `sqrtPriceX96` = `sqrt(price) * 2^96` (Q64.96 fixed-point)

---

### V3 Add Liquidity (Mint)

```mermaid
sequenceDiagram
    participant Caller
    participant Pool
    participant Position
    participant Tick
    participant TickBitmap

    Caller->>Pool: mint(recipient, tickLower, tickUpper, amount, data)
    Pool->>Pool: lock()
    Pool->>Pool: _modifyPosition(owner, tickLower, tickUpper, +liquidityDelta)
    Pool->>Position: get(owner, tickLower, tickUpper)
    Pool->>Tick: update(tickLower) + update(tickUpper)
    Pool->>TickBitmap: flipTick if tick newly initialized or emptied
    Pool->>Pool: compute amount0, amount1 required (based on current tick vs range)
    Pool->>Position: update fees owed
    Pool->>Pool: record balance0Before, balance1Before
    Pool->>Caller: uniswapV3MintCallback(amount0Owed, amount1Owed, data)
    Note over Caller: Caller must transfer tokens into pool
    Pool->>Pool: verify balance increased by at least amount0, amount1
    Pool-->>Caller: emits Mint(...)
```

**Callback pattern:** Pool calls back `msg.sender` to pull tokens — no pre-approval to router needed at pool level

**Amount calculation depends on current tick position:**

| Current Tick vs Range | Token Required         |
| --------------------- | ---------------------- |
| Below `tickLower`     | Only token0            |
| Inside range          | Both token0 and token1 |
| Above `tickUpper`     | Only token1            |

---

### V3 Remove Liquidity (Burn + Collect)

V3 separates removal into **two steps**:

#### Step 1 — Burn (remove liquidity, credit fees owed)

```mermaid
sequenceDiagram
    participant LP
    participant Pool

    LP->>Pool: burn(tickLower, tickUpper, amount)
    Pool->>Pool: _modifyPosition(msg.sender, tickLower, tickUpper, -liquidityDelta)
    Pool->>Pool: compute amount0, amount1 to return (pro-rata of position)
    Pool->>Position: add amount0 + amount1 to tokensOwed
    Pool-->>LP: emits Burn(amount0, amount1) — tokens not yet transferred
```

#### Step 2 — Collect (actually withdraw tokens + fees)

```mermaid
sequenceDiagram
    participant LP
    participant Pool

    LP->>Pool: collect(recipient, tickLower, tickUpper, amount0Requested, amount1Requested)
    Pool->>Position: get(msg.sender, tickLower, tickUpper)
    Pool->>Pool: transfer min(requested, tokensOwed0) of token0
    Pool->>Pool: transfer min(requested, tokensOwed1) of token1
    Pool-->>LP: emits Collect(amount0, amount1)
```

> **Why two steps?** `burn()` decrements liquidity and accrues fees into `tokensOwed`. `collect()` actually transfers. This lets LPs accrue fees without removing liquidity.

---

### V3 Swap

```mermaid
sequenceDiagram
    participant Caller
    participant Pool
    participant TickBitmap
    participant SwapMath
    participant Oracle

    Caller->>Pool: swap(recipient, zeroForOne, amountSpecified, sqrtPriceLimitX96, data)
    Pool->>Pool: validate price limit, lock pool
    Pool->>Pool: init SwapState (price, tick, liquidity, feeGrowth)

    loop while amountRemaining != 0 and price != limit
        Pool->>TickBitmap: nextInitializedTickWithinOneWord(currentTick, zeroForOne)
        TickBitmap-->>Pool: tickNext, initialized
        Pool->>TickMath: getSqrtRatioAtTick(tickNext)
        Pool->>SwapMath: computeSwapStep(sqrtPrice, target, liquidity, amountRemaining, fee)
        SwapMath-->>Pool: newSqrtPrice, amountIn, amountOut, feeAmount
        Pool->>Pool: update amountRemaining, protocol fee, feeGrowthGlobal
        alt reached tickNext (initialized tick)
            Pool->>Oracle: write observation if tick changed
            Pool->>Tick: cross(tickNext) → liquidityNet
            Pool->>Pool: adjust active liquidity by ±liquidityNet
        end
        Pool->>Pool: update current tick
    end

    Pool->>Pool: write oracle observation if tick changed
    Pool->>Pool: update slot0 (price, tick, oracle index)
    Pool->>Pool: update feeGrowthGlobal, protocolFees, liquidity
    Pool->>Pool: transfer output tokens to recipient
    Pool->>Caller: uniswapV3SwapCallback(amount0Delta, amount1Delta, data)
    Note over Caller: Caller must send input tokens to pool
    Pool->>Pool: verify input balance received
    Pool-->>Caller: emits Swap(...)
    Pool->>Pool: unlock pool
```

**Swap direction:**

- `zeroForOne = true` → selling token0 for token1 (price decreases)
- `zeroForOne = false` → selling token1 for token0 (price increases)

**Exact input vs exact output:**

- `amountSpecified > 0` → exact input (pay exactly, receive at least)
- `amountSpecified < 0` → exact output (receive exactly, pay at most)

**Tick crossing:** When price crosses an initialized tick, `liquidity` is adjusted by `liquidityNet` of that tick — activating or deactivating position ranges.

---

### V3 Flash Loan

```mermaid
sequenceDiagram
    participant Caller
    participant Pool

    Caller->>Pool: flash(recipient, amount0, amount1, data)
    Pool->>Pool: lock() + require liquidity > 0
    Pool->>Pool: compute fee0 = amount0 * fee / 1e6
    Pool->>Pool: compute fee1 = amount1 * fee / 1e6
    Pool->>Pool: record balance0Before, balance1Before
    Pool->>Token0: safeTransfer(recipient, amount0)
    Pool->>Token1: safeTransfer(recipient, amount1)
    Pool->>Caller: uniswapV3FlashCallback(fee0, fee1, data)
    Note over Caller: Caller executes logic, must repay principal + fee
    Pool->>Pool: verify balance0After >= balance0Before + fee0
    Pool->>Pool: verify balance1After >= balance1Before + fee1
    Pool->>Pool: distribute fees to LPs via feeGrowthGlobal
    Pool->>Pool: deduct protocol fee portion if enabled
    Pool-->>Caller: emits Flash(...)
```

---

## V2 vs V3 — Key Differences

| Feature                  | V2                             | V3                                                     |
| ------------------------ | ------------------------------ | ------------------------------------------------------ |
| **Liquidity model**      | Full range (0 → ∞)             | Concentrated within `[tickLower, tickUpper]`           |
| **Fee tiers**            | Fixed 0.3%                     | 0.05% / 0.3% / 1% (extendable)                         |
| **LP token**             | ERC20 SLP token (fungible)     | NFT position (non-fungible via periphery)              |
| **Price representation** | `reserve0`, `reserve1`         | `sqrtPriceX96` (Q64.96 fixed-point)                    |
| **Oracle**               | Cumulative price per block     | Ring buffer of up to 65535 observations (TWAP)         |
| **Pool per pair**        | 1                              | Multiple (one per fee tier)                            |
| **Swap callback**        | Optional (`IUniswapV2Callee`)  | Required (`IUniswapV3SwapCallback`)                    |
| **Flash loans**          | Via `swap()` + callee          | Dedicated `flash()` function                           |
| **Remove liquidity**     | Single `burn()` tx             | Two-step: `burn()` then `collect()`                    |
| **Protocol fee**         | 1/6 of LP fee (off by default) | Configurable 1/N of LP fee per token                   |
| **Pool deployer**        | `CREATE2` in Factory           | Separate `UniswapV3PoolDeployer` with transient params |
| **Reentrancy guard**     | `uint unlocked` flag           | `slot0.unlocked` bool in packed storage                |

---

## Key Formulas

### V2 Constant Product AMM

```
x * y = k
amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)
```

### V2 Liquidity (first deposit)

```
liquidity = sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY
```

### V2 Price Accumulators (TWAP)

```
price0Cumulative += (reserve1 / reserve0) * timeElapsed   [UQ112x112]
price1Cumulative += (reserve0 / reserve1) * timeElapsed
```

### V3 Price Representation

```
sqrtPriceX96 = sqrt(token1/token0) * 2^96    [Q64.96]
price = (sqrtPriceX96 / 2^96)^2
```

### V3 Tick ↔ Price

```
price = 1.0001 ^ tick
tick  = floor(log(price) / log(1.0001))
```

### V3 Fee Growth

```
feeGrowthGlobalX128 += (feeAmount * Q128) / liquidity
tokensOwed += liquidity * (feeGrowthInside - feeGrowthInsideLast)
```

### V3 Flash Fee

```
fee = ceil(amount * poolFee / 1_000_000)
```
