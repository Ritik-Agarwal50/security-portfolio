#  QuantAMM Finding

## [M-1] Incorrect Change of State Variable

###  Summary
In the `UpdateWeightRunner` contract, two different types of protocol fees exist: one for swapping and one for withdrawal. However, the `setQuantAMMUpliftFeeTake()` function mistakenly modifies the **swapping** fee variable, which is incorrect.

###  Vulnerability Details
The state variable `quantAMMSwapFeeTake` is being modified in **both**:
- `setQuantAMMSwapFeeTake()` — (intended)
- `setQuantAMMUpliftFeeTake()` — (unintended)

This creates a conflict because `setQuantAMMUpliftFeeTake()` should be updating a separate variable named `quantAMMUpliftFeeTake`. But instead, it's overwriting `quantAMMSwapFeeTake`, affecting swap fee logic incorrectly.

####  Affected Code:
```solidity
function setQuantAMMUpliftFeeTake(uint256 _quantAMMUpliftFeeTake) external {
    require(msg.sender == quantammAdmin, "ONLYADMIN");
    require(_quantAMMUpliftFeeTake <= 1e18, "Uplift fee must be less than 100%");
    uint256 oldSwapFee = quantAMMSwapFeeTake;
    quantAMMSwapFeeTake = _quantAMMUpliftFeeTake; //  Bug: modifies wrong variable
    emit UpliftFeeTakeSet(oldSwapFee, _quantAMMUpliftFeeTake);
}

function getQuantAMMUpliftFeeTake() external view returns (uint256) {
    return quantAMMSwapFeeTake; //  Bug: returns wrong variable
}
```

### Impact:
- Incorrect state management may result in:
- Misconfigured fees for either swap or withdrawal,
- Unintended economic behavior,
- Potential financial loss to users or protocol due to incorrect fee accounting.

### Recommendation:
``` solidity
uint256 public quantAMMUpliftFeeTake = 0.5e18;

function setQuantAMMUpliftFeeTake(uint256 _quantAMMUpliftFeeTake) external {
    require(msg.sender == quantammAdmin, "ONLYADMIN");
    require(_quantAMMUpliftFeeTake <= 1e18, "Uplift fee must be less than 100%");
    uint256 oldUpliftFee = quantAMMUpliftFeeTake;
    quantAMMUpliftFeeTake = _quantAMMUpliftFeeTake; //  Fix: use correct variable
    emit UpliftFeeTakeSet(oldUpliftFee, _quantAMMUpliftFeeTake);
}

function getQuantAMMUpliftFeeTake() external view returns (uint256) {
    return quantAMMUpliftFeeTake;
}
```