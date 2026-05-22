### [M-1] Output Swap Hook Fee Overcharge on Partial Fill LimitBreakAMM
**Summary**
The LimitBreakAMM incorrectly calculates and stores beforeSwap token hook fees during output-specified swaps that result in a partial fill. The hook fee is recorded in tokensOwed before the pool executes, based on the user's requested output amount. When the pool only delivers a fraction of that amount (partial fill), the stored fee is never corrected downward. The hook collects fees on the full requested amount even though the pool only partially filled the order. The difference is silently drained from LP reserves on every such swap.

**Vulnerability Detail**
An output swap is triggered when amountSpecified < 0. The user declares how many tokens they want to receive and the AMM determines how much input they pay. Pool types are allowed to return actualAmountOut < amountOut this is a partial fill and is accepted by the AMM when minAmountSpecified = 0.

Token hooks are registered on specific ERC20 tokens via amm.setTokenSettings(token, hook, flags). When TOKEN_SETTINGS_BEFORE_SWAP_HOOK_FLAG is set, the AMM calls hook.beforeSwap() before the pool executes and stores the returned fee in tokensOwed. This fee is later collected by the hook beneficiary.

Step 1 — Hook fee stored (BEFORE pool runs):
  _applySwapByOutputOutputFees(swapCache, ...)
    │
    ├─ hook.beforeSwap(amount = swapCache.amountOut)
    │        ↑ this is the REQUESTED output (e.g. 1000)
    │
    └─ _storeHookFees() → tokensOwed += fee
             ↑ fee = 10% × 1000 = 100 — WRITTEN NOW, PERMANENTLY
Step 2 — Pool executes:
  poolType.swapByOutput(amountOut = 1000)
    └─ returns actualAmountOut = 700  ← PARTIAL FILL

What should happen: fee = 10% × 700 = 70 What actually happens: fee = 10% × 1000 = 100 LP loss per swap: 100 − 70 = 30 tokens

The 30-token overcharge is paid out of the AMM's token balance (LP reserves) when the hook beneficiary later calls collectHookFeesByToken().

### [M-2] Stale hook price after reserve cap allows value extraction from liquidity provider
**Summary**
SwapByInput function queries the price hook once using the caller's original amount. If the computed output exceed the reserveOut the code internally switched to the SwapByOutput to compute the reduced input required for the capped output, but the hook never re-queried with the actual trade size (which got change). This allow attacker to inflate amountIn to obtain a favorable larger-trade price, let the reserve cap reduce reduce their actual cost and receive far more output than the hook intended for their real trade size

**Vulnerability In Detail**
SwapByInput working : When a user calls the amm.singleSwap() for an input-based swap, the AMM delegates to SingleProviderPoolType.swapByInput

```solidity
  // SingleProviderPoolType.sol L323-327
  swapCache.sqrtPriceCurrentX96 = ISingleProviderPoolHook(swapCache.poolHook)
      .getPoolPriceForSwap(
          context,
          priceParams,   // priceParams.amount = ORIGINAL amountIn
          swapExtraData
      );

  // price is cached here and never updated again
  pools[poolId].lastSqrtPriceX96 = swapCache.sqrtPriceCurrentX96;
  SingleProviderHelper.swapByInput(swapCache, uint16(poolFeeBPS));

  Inside SingleProviderHelper.swapByInput

  uint256 amountOut = calculateFixedInput(amountInAfterFees, sqrtPriceX96, zeroForOne);

  if (amountOut > swapCache.reserveOut) {
      // Reserve cap: output capped, input recomputed via swapByOutput
      swapCache.amountOut = swapCache.reserveOut;
      swapByOutput(swapCache, poolFeeBPS);
      // sqrtPriceX96 = stale price (from large amountIn) still in swapCache
      //   hook is NOT re-queried for actual smaller trade size
  }

  This is Problem ->
  struct HookPoolPriceParams {
      bool inputSwap;
      bytes32 poolId;
      address tokenIn;
      address tokenOut;
      uint256 amount;   // "Amount being swapped" — hook can and should use this
  }
```
It means hooks are designed to price base on trade size. Hook giving the better price to the larger trade.
**Mitigation**
Re-query the hook after the reserve cap with the actual executed trade size before calling the SwapByOutput function