## Finding [H-1]: MEV and Slippage Vulnerability in openPosition() Function

### Summary
The `openPosition` function is vulnerable to MEV attacks and slippage issues due to the absence of minimum slippage tolerance (`amount0Min` and `amount1Min` are set to 0). This flaw allows attackers to manipulate the price during transaction processing, leading to potential user loss by executing arbitrage or front-running attacks.

### Finding Description
The `openPosition` function, when interacting with the position manager to mint liquidity position, sets both `amount0Min` and `amount1Min` to 0:  
https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/shadow/ShadowRangePositionImpl.sol?lines=122,123

```solidity
amount0Min: 0,
amount1Min: 0
```

This causes the slippage issue. Without setting a minimum limit, an attacker can front-run the transaction and try to manipulate the price, leading to the victim accepting worse rates than expected.

### Impact Explanation
-> Loss of funds - the user will get less than the expected rate.

### Likelihood Explanation
HIGH -> In an active defi market where bots frequently monitor unprotected transactions to get benefits. The absence of slippage tolerance makes thisan attractive opportunity for the attackers.

### Recommendation
To mitigate this issue, add slippage protection on amount0min and amount1min. mitigate




















## Finding [M-1]: Non-Standard ERC20 `transfer()` Usage Causes Reverts

### Summary
Several functions use the `transfer()` function assuming strict ERC20 standard compliance. However, tokens like USDT and WBTC deviate from the standard by not returning a boolean value, which causes calls to revert unexpectedly.

### Vulnerability Details
Affected functions are located at:

- [PaymentsUpgradeable.sol: line 50](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/PaymentsUpgradeable.sol?lines=50,50)
- [Payments.sol: line 50](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/Payments.sol?lines=50,50)
- [StakingRewards.sol: lines 165, 190, 201](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/StakingRewards.sol?lines=165,165)
- [StakingRewards.sol: line 190](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/StakingRewards.sol?lines=190,190)
- [StakingRewards.sol: line 201](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/StakingRewards.sol?lines=201,201)

These instances use `transfer()` to send tokens. However, tokens like USDT do **not** return a boolean value; they revert on failure and return nothing on success. Solidity expects return data, so if none is returned, the call reverts even if the transfer succeeded.

### Impact
- User funds can get locked if `withdraw()` fails due to a non-standard token behavior.
- Transfers that actually succeed might still cause contract logic to revert because of missing return data.

### Likelihood
High likelihood with USDT and similar tokens that do not return boolean values on `transfer()`.

### Recommendation
Replace all `transfer()` calls with OpenZeppelin’s `safeTransfer()` method, which safely handles tokens that do not return values.


## Finding [M-2]:Unwrapping excessinve WETH in unwrapWETH9() function
### Summary
The unwrapWETH9 function unwraps the entire WETH balance of the contract and sends it to the user.

### Finding Description
The function:
`function unwrapWETH9(uint256 amountMinimum, address recipient) internal` The intention of this amountMinimum parameter suggests that the contract should unwrap at least that much WETH to ETH and send it to the recipient. However, the implementation unwraps the full WETH balance of the contract. https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/PaymentsUpgradeable.sol?lines=27,32

```solidity
uint256 balanceWETH9 = IWETH9(WETH9).balanceOf(address(this));
require(balanceWETH9 >= amountMinimum, Errors.VL_INSUFFICIENT_WETH9);
if (balanceWETH9 > 0) {
    IWETH9(WETH9).withdraw(balanceWETH9);
    TransferHelper.safeTransferETH(recipient, balanceWETH9);
}
```
This creates the issue:
- All WETH is unwrapped, not only the requested amount.
- May interfere with accounting logic, pool logic, or upcoming transactions that expect remaining balances.
Describe which security guarantees it breaks and how it breaks them. If this bug does not automatically happen, showcase how a malicious input would propagate through the system to the part of the code where the issue occurs.

### Impact Explanation
HIGH -> Contract unintentionally sends more ETH than it was supposed to. -> Loss of funds -> Send funds to a third party who didn't own it.

### Likelihood Explanation
-> Likely at the time of Redeem, it will get triggered if the user wants native ETH parameter

### Recommendation
-> Unwrap and send only the required amount
``` solidity
function unwrapWETH9(uint256 amountMinimum, address recipient) internal {
    uint256 balanceWETH9 = IWETH9(WETH9).balanceOf(address(this));
    require(balanceWETH9 >= amountMinimum, Errors.VL_INSUFFICIENT_WETH9);

    IWETH9(WETH9).withdraw(amountMinimum);
    TransferHelper.safeTransferETH(recipient, amountMinimum);
}
```
This will ensure that only the required ETH amount is unwrapped and sent.

