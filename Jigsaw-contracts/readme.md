### [M-1]: Repayments and liquidations can be forced to revert by an attacker that repays minuscule amount of shares
### Summary
A front-running griefing attack allows an attacker to prevent or block liquidations by slightly manipulating collateral or debt amounts, causing key protocol checks to fail and revert the liquidation transaction.

### Finding Description
The liquidation function verifies that the liquidation amount _jUsdAmount does not exceed the user's borrowed amount and that the user’s position is liquidatable. An attacker with front-running capability can add a minimal amount (e.g., 1 wei or 1 unit of collateral) to the user's balance or repay just enough debt to fail the isLiquidatable check or make _jUsdAmount > borrowed. This causes the liquidation transaction to revert, effectively blocking liquidation.

This breaks the protocol’s security guarantees around solvency and liquidation finality, as liquidations are critical to maintaining the health of the protocol. By causing legitimate liquidations to fail, the attacker can keep underwater positions active, increasing systemic risk.

### Impact Explanation
MED -> If liquidations are blocked, the protocol’s health deteriorates due to unresolved undercollateralized positions, increasing the risk of insolvency

### Likelihood Explanation
MED -> requires front-running capability and manipulating token balances or repayments by small amounts.

### Proof of Concept
```solidity
    function testgriefingAttack() public {
        address collateral = address(usdc);
        uint256 collateralAmount = 100_000e6;
        address userHolding = initiateWithUsdc(user, collateralAmount);

        uint256 userJusdBefore = ISharesRegistry(registries[address(usdc)]).borrowed(userHolding);
        uint256 userCollateralBefore = ISharesRegistry(registries[address(usdc)]).collateral(userHolding);

        deal(address(jUsd), address(this), userJusdBefore);
        deal(address(usdc), address(this), 0);

        uint256 totalSupplyBefore = jUsd.totalSupply();
    

        // make investment
         vm.prank(user, user);
         strategyManager.invest(address(usdc), address(strategyWithoutRewardsMock), userCollateralBefore, 0, "");

         ILiquidationManager.LiquidateCalldata memory liquidateCalldata =
             ILiquidationManager.LiquidateCalldata({ strategies: new address[](1), strategiesData: new bytes[](1) });
         liquidateCalldata.strategies[0] = address(strategyWithoutRewardsMock);

        usdcOracle.setPriceForLiquidation();
        deal(address(usdc), user, 1);
        vm.startPrank(user, user);
        usdc.approve(address(holdingManager), 1);
        vm.stopPrank();

        vm.startPrank(user, user);
        deal(address(jUsd), user, 1);
        holdingManager.deposit(address(usdc), 1);
        holdingManager.repay(address(usdc), 1 , true);
        vm.stopPrank();
        //vm.expectRevert("2003");
        liquidationManager.liquidate(user, address(usdc), userJusdBefore, 0, liquidateCalldata);

        (uint256 investedAmount, uint256 totalShares) = IStrategy(strategyWithoutRewardsMock).recipients(userHolding);
    }

```

### Recommendation
Allow to attempt to repay an unlimited amount of shares. Send back to the user tokens that were not required for the full repayment.

### [L-1]: Incorrect Handling of Fee-on-Transfer Tokens During Deposits
### Summary
The protocol does not correctly handle Fee-on-Transfer (FoT) tokens during deposit operations, potentially leading to inaccurate accounting of user balances.

### Finding Description
During deposits, the protocol assumes the full token amount is received by the holding contract without accounting for tokens that deduct a fee on transfer. This leads to discrepancies between the actual token amount held and the recorded amount in the state variables, breaking the accounting integrity and potentially allowing users to manipulate balances or cause unexpected failures in liquidation or repayments.

The supported tokens are USDC, USDT, USDe, USDs, rUSD, BTC, ETH, wETH, wstETH, rswETH, weETH, and ezETH. All these tokens do not support Fee-on-Transfer except USDT, which historically had FoT but currently has FoT disabled in its main implementations. Thus, the protocol's deposit logic is safe for these tokens under current conditions.

### Impact Explanation
If Fee-on-Transfer tokens were introduced or enabled, this issue could lead to incorrect collateral and debt accounting, potentially enabling attackers to exploit the protocol by manipulating token transfers and balances.

### Likelihood Explanation
Given that all supported tokens currently do not have active Fee-on-Transfer mechanisms (with USDT's FoT disabled), the likelihood of this issue occurring in practice is low. However, future token upgrades or newly supported tokens with FoT could expose this vulnerability.

### Recommendation
Ensure the protocol implements proper handling for Fee-on-Transfer tokens during deposit operations to prevent discrepancies between transferred amounts and state accounting.

