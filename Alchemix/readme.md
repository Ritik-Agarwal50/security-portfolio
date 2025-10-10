### [H-1]: User Unable to Withdraw Full Collateral After Redemption

### Summary
The issue revolves around incorrect management of collateral within the contract, resulting in tokens being stuck in the contract and not properly freed. This primarily affects the redemption and withdrawal functions, where collateral is not updated as expected after the operations, causing discrepancies in the user's balance and leading to potential token lockup.

### Finding Description
During the redemption and withdrawal processes, collateral is expected to be freed or updated based on certain logic, but it fails to do so properly. As a result, tokens are not released from the contract, and users are unable to withdraw or redeem the full amount of tokens they expect.

Claim Redemption: While the claimRedemption function is handling transmutation and redemption well, it doesn't properly free the collateral in some cases, leading to a situation where collateral remains locked within the contract.

Withdraw: Similarly, in the withdrawal logic, collateral is not being reduced or freed as expected after a withdrawal, causing a buildup of unclaimed tokens within the contract.

The most likely cause is that the contract is not properly accounting for collateral when users perform certain actions (e.g., redemption and withdrawal). This results in the failure to adjust the collateral value, which is causing tokens to be stuck in the contract.

### Impact Explanation
The primary impact of this issue is that users are unable to withdraw or redeem their collateral as expected. In some cases, tokens might get stuck within the contract because the collateral value isn't updated correctly.

### Likelihood Explanation
The likelihood of this issue occurring is high since it stems from how collateral is managed across multiple functions (claimRedemption and withdraw). Since these are core operations within the contract, it’s likely to affect users attempting to withdraw or redeem their tokens

### Proof of Concept
```solidity
 function testStateOfUserCollateral() public {
        uint256 amount = 100e18;
        uint256 protocolFeePercentage = 1000; // 10% fee (1000 BPS)

        //setting Fees baby
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(protocolFeePercentage); // Set protocol fee to 10%
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));
        vm.stopPrank();

        //Redemption time baby 
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic) , 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();
        
        
        vm.roll(block.number + 5_256_000);
        
        (uint256 collateral, uint256 userDebt,uint256 earmarked) = alchemist.getCDP(tokenId);
        uint256 creditToYield = alchemist.convertDebtTokensToYield(userDebt);
        uint256 expectedFeeAmount = (creditToYield * protocolFeePercentage) / 10000;
        address feeReceiver = alchemist.protocolFeeReceiver();
        uint256 protocolFeeBefore = fakeYieldToken.balanceOf(feeReceiver);
        uint256 DebtToYeild = alchemist.convertYieldTokensToDebt(userDebt);
        
        console.log("Pre-Redemption Logs:");
        console.log("Protocol Fee Before: ", protocolFeeBefore);
        console.log("Collateral: ", collateral);
        console.log("User Debt: ", userDebt);
        console.log("Earmarked: ", earmarked);
        console.log("Credit to Yield: ", creditToYield);
        console.log("Expected Fee Amount: ", expectedFeeAmount);
        console.log("Debt to Yield: ", DebtToYeild);
        console.log(" ");

        vm.startPrank(address(0xdad));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

        alchemist.poke(tokenId);

        // Post-balances
        (uint256 collatAfter, uint256 userDebtAfter,uint256 earmarkedAfter) = alchemist.getCDP(tokenId);
        uint256 protocolFeeAfter = fakeYieldToken.balanceOf(feeReceiver);
        uint256 actualFeeTransferred = protocolFeeAfter - protocolFeeBefore;


        creditToYield = alchemist.convertDebtTokensToYield(userDebtAfter    );
        DebtToYeild = alchemist.convertYieldTokensToDebt(userDebtAfter);
        console.log("Post-Redemption Logs:");
        console.log("Post Credit To Yeild: ", creditToYield); 
        console.log("Protocol Fee After: ", protocolFeeAfter);
        console.log("Earmarked After: ", earmarkedAfter);
        console.log("Collateral After: ", collatAfter);
        console.log("User Debt After: ", userDebtAfter);
        console.log("Actual Fee Transferred: ", actualFeeTransferred);
        console.log("Debt to Yield After: ", DebtToYeild);

        vm.startPrank(address(0xbeef));
        alchemist.withdraw(44e18,address(0xbeef),tokenId);
        vm.stopPrank();

        alchemist.poke(tokenId);


        (collatAfter, userDebtAfter,earmarkedAfter) = alchemist.getCDP(tokenId);
        console.log(
            " "
        );
        console.log("Final Logs:");
        console.log("Final Collateral: ", collatAfter);
        console.log("Final User Debt: ", userDebtAfter);
        console.log("Final Earmarked: ", earmarkedAfter);

        console.log("Yield Token Balance: ", fakeYieldToken.balanceOf(address(0xbeef)));
        console.log("Transmuter Logic Balance: ", fakeYieldToken.balanceOf(address(transmuterLogic)));


        vm.startPrank(address(0xbeef));
        alchemist.withdraw(0.4e18,address(0xbeef),tokenId);
        vm.stopPrank();
        alchemist.poke(tokenId);
        (collatAfter, userDebtAfter,earmarkedAfter) = alchemist.getCDP(tokenId);
        console.log(
            " "
        );
        console.log("Final Logs After 0.8e18 Withdraw:");
        console.log("Final Collateral After 0.8e18 Withdraw: ", collatAfter);
        console.log("Final User Debt After 0.8e18 Withdraw: ", userDebtAfter);
        console.log("Final Earmarked After 0.8e18 Withdraw: ", earmarkedAfter);

        alchemist.poke(tokenId);
        (collatAfter, userDebtAfter,earmarkedAfter) = alchemist.getCDP(tokenId);
        console.log(
            " "
        );
        console.log("Final Logs After 0.8e18 Withdraw:");
        console.log("Final Collateral After 0.8e18 Withdraw: ", collatAfter);
        console.log("Final User Debt After 0.8e18 Withdraw: ", userDebtAfter);
        console.log("Final Earmarked After 0.8e18 Withdraw: ", earmarkedAfter);

    }

```

### Recommendation
Ensure free collateral happens correctly after redemption


### [H-2]: Early Redemption Vulnerability in Self-Repayment Loan Protocol Leading to Excessive Fee Transfer
### Summary
The repay function miscalculates and incorrectly transfers the protocol fee amount, resulting in the Entire repayment amount being forwarded to the protocol fee receiver instead of only the expected protocol fee. This causes both user balance inconsistencies and overpayments to the fee receiver.

### Finding Description
Impacted Function - https://cantina.xyz/code/e68909e6-3491-4a94-a707-ecf0c89cf72a/src/AlchemistV3.sol?lines=488,527
The core issue lies within the internal handling of the repayment logic in the repay function. Specifically, the function improperly transfers the entire amount of credit to Yield to the protocol fee receiver, without subtracting the expected ProtocolFeeAmount. This results in an overpayment of the fee and an underpayment of the actual debt, thus leaving the user’s debt uncleared despite a full repayment attempt.

For instance, if a user repays 100 tokens and the protocol fee is 10%, the fee receiver should only receive 10 tokens, while the remaining 90 should go towards paying the debt via the transmuter. However, the implementation sends the full 100 tokens to the fee receiver, breaking the assumption that debt is being reduced.

This issue breaks the core accounting guarantees of the protocol and can be exploited to either bypass proper debt clearing or cause users to overpay during repayment, particularly when high fees are configured.

### Impact Explanation
High -> Incorrect protocol fee collection, broken accounting

### Likelihood Explanation
Medium -> The likelihood is medium because users may trigger early redemption or self-repayment before the expected time, leading to incorrect fee transfers if the protocol fee handling is flawed

### POC
```solidity

function test_repaymentofDebtFeeIssuee() external {
    uint256 amount = 100e18;
    uint256 protocolFeePercentage = 1000; // 10% fee (1000 BPS)

    // Set the protocol fee before testing repayment
    vm.startPrank(alOwner);  // Set protocol fee as alOwner
    alchemist.setProtocolFee(protocolFeePercentage);  // Set protocol fee to 10%
    vm.stopPrank();

    // Simulate deposit + mint
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenIdFor0xBeef, (amount / 2), address(0xbeef));
    vm.stopPrank();

    // Fetch user debt before repayment
    (, uint256 userDebtBefore,) = alchemist.getCDP(tokenIdFor0xBeef);

    // Compute expected fee properly
    uint256 yieldToDebt = alchemist.convertYieldTokensToDebt(amount);
    uint256 credit = yieldToDebt > userDebtBefore ? userDebtBefore : yieldToDebt;
    uint256 creditToYield = alchemist.convertDebtTokensToYield(credit);
    uint256 expectedFeeAmount = (creditToYield * protocolFeePercentage) / 10000;

    // Pre-balances
    address feeReceiver = alchemist.protocolFeeReceiver();
    uint256 preRepayBalance = fakeYieldToken.balanceOf(address(0xbeef));
    uint256 preRepayProtocolFeeReceiverBalance = fakeYieldToken.balanceOf(feeReceiver);

    // Execute repayment
    vm.startPrank(address(0xbeef));
    vm.roll(block.number + 1);  // Ensure next block
    alchemist.repay(amount, tokenIdFor0xBeef);
    vm.stopPrank();

    // Post-balances
    uint256 postRepayBalance = fakeYieldToken.balanceOf(address(0xbeef));
    uint256 postRepayProtocolFeeReceiverBalance = fakeYieldToken.balanceOf(feeReceiver);

    // Final debt
    (, uint256 userDebtAfter,) = alchemist.getCDP(tokenIdFor0xBeef);

    // Assertions
    assertEq(userDebtAfter, userDebtBefore - credit, "User debt should decrease correctly.");
    assertEq(postRepayProtocolFeeReceiverBalance, preRepayProtocolFeeReceiverBalance + expectedFeeAmount, "BUG: Protocol fee receiver received wrong amount.");
    assertEq(postRepayBalance, preRepayBalance - creditToYield, "User token balance did not decrease correctly.");
}


```

### Recommendation
```solidity
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        _checkArgument(amount > 0);
        _checkForValidAccountId(recipientTokenId);
        Account storage account = _accounts[recipientTokenId];
        // Check that the user did not mint in this same block
        // This is used to prevent flash loan repayments
        if (block.number == account.lastMintBlock) revert CannotRepayOnMintBlock();

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(recipientTokenId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 yieldToDebt = convertYieldTokensToDebt(amount);
        uint256 credit = yieldToDebt > debt ? debt : yieldToDebt;
        uint256 creditToYield = convertDebtTokensToYield(credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        // Debt is subject to protocol fee similar to redemptions
        account.collateralBalance -= (creditToYield * protocolFee) / BPS;

        _subDebt(recipientTokenId, credit);

        // Transfer the repaid tokens to the transmuter.
        
-        TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
-        TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield );
 
+        uint256 feeAmount = creditToYeild * protocolFee / BPS;
+       TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, feeAmount);
+       TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield - feeAmount);

        emit Repay(msg.sender, amount, recipientTokenId, creditToYield);

        return creditToYield;
    }

```