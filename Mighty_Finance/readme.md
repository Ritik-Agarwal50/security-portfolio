## Finding [H-1]: Slippage Issue in openPosition Function Due to Missing Minimum Amounts

### Summary
The `openPosition` function is vulnerable to MEV attacks and slippage issues due to the absence of minimum slippage tolerance (`amount0Min` and `amount1Min` are set to 0). This flaw allows attackers to manipulate the price during transaction processing, leading to potential user loss by executing arbitrage or front-running attacks.

### Finding Description
The `openPosition` function, when interacting with the position manager to mint liquidity position, sets both `amount0Min` and `amount1Min` to 0:  
- [ShadowRangePositionImpl.sol: line 122-123](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/shadow/ShadowRangePositionImpl.sol?lines=122,123)

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

## Finding [H-2]: Lack of Validation Leading to Ping-Pong Situation and Imbalanced Position Creation
### Summary
The lack of proper validation in the `ShadowRangeVault::openPosition()` function causes an imbalance when creating a position. Without validating the desired token amounts, mismatched values are passed to the `ShadowRangePositionImpl::openPosition()`. This leads to incorrect swaps being executed in an attempt to match the desired token amounts. As a result, the system creates positions with incorrect token quantities, which also causes errors during token minting and results in erroneous emitted data.
### Finding Description
Code of 
[ShadowRangeVault::openPosition](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/shadow/ShadowRangeVault.sol?lines=152,240)

The issue begins in the `ShadowRangeVault::openPosition()` function, where users pass parameters, including `amount0Desired` and `amount1Desired`, representing the desired token amounts. These values can be different from the principal amounts (`amount0Principal` and `amount1Principal`). However, the function lacks any validation to ensure that the desired amounts match the principal amounts or that the sum of the principal and borrowed amounts aligns with the desired amounts. Without this check, an imbalance occurs in the position creation process.

- [Swaping Code](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/shadow/ShadowRangePositionImpl.sol?lines=88,106)

As the function progresses, state variables are set, and the amounts are transferred to the `ShadowRangePositionImpl` contract, which handles the core logic. When calling `ShadowRangePositionImpl::openPosition()` with the desired amounts and other parameters, the function checks the balance of `token0`. If the balance is insufficient to meet `amount0Desired`, the contract swaps `token1` for `token0` to match the desired amount, and similarly for `token1`.

However, a problematic scenario arises when the attacker provides a larger `amount0Desired` than the principal amount, which leads to a swap where `token0` is exchanged for `token1`. This swap decreases the `token1` balance, prompting the contract to swap `token0` back to `token1`, creating a ping-pong situation. Eventually, `token0`ends up with a lower balance than desired, while token1 matches its desired amount.

Despite this imbalance, the contract proceeds with approval and minting. Ideally, the minting process should revert because the desired amounts don't align. However, since both `amount0Min` and `amount1Min` are set to 0, the minting process doesn't revert and instead mints an incorrect amount of tokens. This results in an imbalanced position being created with incorrect data, which is then emitted as incorrect state information.

### Impact Explanation
The lack of validation between the desired amounts and the principal amounts in the `ShadowRangeVault::openPosition()` function impact the position creation process. SInce the system does not validate the principal and desired amount or the sum of principal and borrowed amount, it allows attacker to pass the imbalanced amounts which create incorrect token swap and position token minting. This imbalance amount triggers a **ping-pong** situation where token0 will end up with a lower balance than desired.

### Likelihood Explanation
The likelihood is high due to a lack of validation for mismatched desired amounts in the `openPosition` function. A malicious user could easily exploit this vulnerability by providing wrong inputs, leading imbalanced position.

### Recommendation
Add checks to ensure that the amount0Desired and amount1Desired match the principal amounts or the sum of principal and borrowed amounts. If there is any mismatch, revert the transaction to prevent creating imbalanced positions.
Modify the minting process to include proper slippage constraints. Currently, `amount0Min` and `amount1Min` are set to 0, allowing any amount to be minted. Set these values to a reasonable minimum threshold to prevent slippage and mitigate potential attacks.


## Finding [I-1]:Unwrapping excessinve WETH in unwrapWETH9() function
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

## Finding [I-2]: fee-on-transfer token create incorrect accounting in stake function also revert tx of last user
### Summary
The StakingRewards::stake() function assumes that the full amount is passed and transferred, but a fee-on-transfer token results in less being received than expected, leading to reward miss calculation and potential fund loss.
### Finding Description
The StakingRewards::stake() function records the amount passed by the user without validatingthe actual amount of tokens received by the contract. For tokens with a fee-on-transfer mechanism, the amount send form the user is greater than the amount contract actually received. THis causes a mismatch in accounting.
- [StakingRewards.sol lines: 146,147](ttps://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/lendingpool/StakingRewards.sol?lines=146,147)
```solidity
function stake(uint256 amount, address onBehalfOf) external nonReentrant updateReward(onBehalfOf) {
        require(amount > 0, "amount = 0");

        stakedToken.safeTransferFrom(msg.sender, address(this), amount);

        balanceOf[onBehalfOf] += amount;
        totalStaked += amount;

        emit Staked(msg.sender, onBehalfOf, amount);
    }
```
If the user send 100 tokens and 5 are burned due to transfer fee, the contract only receives 95 tokens however, `balanceOf[onBehalfOf]` and `totalStaked` are both increase by `100` not `95` leading to accounting issue. This will be problematic on `unstake` as the last user to call it will get their transaction reverted because of insufficient balance in the contract.    

### Impact Explanation
- Accounting Issue
- Over-distribution of rewards
- Last user unstake revert.

### Likelihood Explanation
This is likely to occur every time the last user tries to unstake.

### Recommendation
Check the balance before and after the deposit(stake) and use the difference between the two as the actual transferred value.



## Finding [I-3]: ShadwoRangeVault::openPosition() Allows Creation of Dead Positions with Zero Liquidity
### Summary
Liquidity positions can be created with zero liquidity, but no mechanism is available to increase liquidity for them post-creation. This results in either a trapped or unusable position.

### Finding Description
[ShadowRangeVault::OpenPosition](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/contracts/shadow/ShadowRangeVault.sol?lines=152,240)

In the current implementation, a user can open a new liquidity position by using the ShadowRangeVault::openPosition() function. However, this function does not enforce a minimum liquidity amount and allows the creation of a position with zero liquidity. Once created, there is no exposed or accessible function that enables the user to later increase the liquidity of this existing position.. The system uses unique IDs per position, and creating a new position mints a new contract. Therefore, users are permanently locked into a zero-liquidity position without recourse.

### Impact Explanation
This issue does not directly result in loss of funds or critical security compromise. However, it affects protocol usability, expected user behavior, and capital efficiency. If left unchecked, it may lead to a large number of zero-liquidity positions that cannot be upgraded, polluting state mappings.

### Likelihood Explanation
Creating a position with zero liquidity is trivially easy, as no safeguards (e.g., require(amount > 0)) are in place. Given the standard practice in other CLMM-based protocols, users may assume that liquidity can be added later, and therefore, unintentionally create such positions.

### Proof of Concept
Create a newfolder called test outside contract folder and create new file and paste the below line of code:-

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "lib/forge-std/src/Test.sol";
import {console2} from "lib/forge-std/src/console2.sol";
import {IPriceOracle} from "../contracts/interfaces/IPriceOracle.sol";
import {console} from "../lib/forge-std/src/console.sol";
import "../contracts/shadow/ShadowRangeVault.sol";
import "../contracts/shadow/ShadowRangePositionImpl.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {AddressRegistry} from "../contracts/AddressRegistry.sol";
import {VaultRegistry} from "../contracts/VaultRegistry.sol";
import {IVault} from "../contracts/interfaces/IVault.sol";
import {LendingPool} from "../contracts/lendingpool/LendingPool.sol";
import {ShadowPositionValueCalculator} from "../contracts/shadow/ShadowPositionValueCalculator.sol";
import {IVault} from "../contracts/interfaces/IVault.sol";

import {IShadowNonfungiblePositionManager} from "../contracts/interfaces/IShadowNonfungiblePositionManager.sol";
import {IShadowV3Pool} from "../contracts/interfaces/IShadowV3Pool.sol";
import {IShadowSwapRouter} from "../contracts/interfaces/IShadowSwapRouter.sol";

contract MockPriceOracle is IPriceOracle {
    mapping(address => uint256) public prices;

    function setTokenPrice(address token, uint256 price) external {
        prices[token] = price;
    }

    function getTokenPrice(
        address token
    ) public view override returns (uint256) {
        return prices[token];
    }
}

contract ShadowRange is Test {
    address public constant TREASURY = address(0x1);
    IERC20 token0;
    IERC20 token1;

    VaultRegistry vaultRegistry;
    AddressRegistry registry;
    ShadowRangeVault vault;
    ShadowRangePositionImpl positionImpl;

    IShadowNonfungiblePositionManager shadowNonfungiblePositionManager;
    IShadowV3Pool shadowV3Pool;
    IShadowSwapRouter shadowSwapRouter;

    LendingPool public lendingPool;
    MockPriceOracle priceOracle;
    ShadowPositionValueCalculator valueCalculator;

    address attacker = address(0xdead);
    address lender1 = address(0x1337);
    address lender2 = address(0x1338);

    uint256 public token0ReserveId;
    uint256 public token1ReserveId;
    address public token0Address;
    address public token1Address;
    address public stakingToken0Address;
    address public stakingToken1Address;
    uint256 public initialDepositAmount = 1000 ether;

    address constant WETH9 = 0x039e2fB66102314Ce7b64Ce5Ce3E5183bc94aD38;
    address constant EGGS = 0xf26Ff70573ddc8a90Bd7865AF8d7d70B8Ff019bC;
    address constant USDC = 0x29219dd400f2Bf60E5a23d13Be72B486D4038894;
    address constant ShadowNonFungible =
        0x12E66C8F215DdD5d48d150c8f46aD0c6fB0F4406;
    address constant ShadowPool = 0x324963c267C354c7660Ce8CA3F5f167E05649970;
    address constant ShadowRouter = 0x5543c6176FEb9B4b179078205d7C29EEa2e2d695;

    address constant UNI_ROUTER = 0x2626664c2603336E57B271c5C0b26F421741e481;

    string rpcUrl = "https://rpc.soniclabs.com";
    uint24 tickSpacing = 50;

    function setUp() public {
        vm.createSelectFork(rpcUrl, 21039807);
        token0 = IERC20(WETH9);
        token1 = IERC20(USDC);

        shadowNonfungiblePositionManager = IShadowNonfungiblePositionManager(
            ShadowNonFungible
        );
        shadowV3Pool = IShadowV3Pool(ShadowPool);
        shadowSwapRouter = IShadowSwapRouter(ShadowRouter);

        // mockRouter = new MockShadowSwapRouter();
        registry = new AddressRegistry(address(EGGS));
        vaultRegistry = new VaultRegistry(address(registry));
        positionImpl = new ShadowRangePositionImpl();
        priceOracle = new MockPriceOracle();
        valueCalculator = new ShadowPositionValueCalculator();

        registry.setAddress(AddressId.ADDRESS_ID_TREASURY, TREASURY);
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_NONFUNGIBLE_POSITION_MANAGER,
            address(shadowNonfungiblePositionManager)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_ROUTER,
            address(shadowSwapRouter)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_SHADOW_POSITION_VALUE_CALCULATOR,
            address(valueCalculator)
        );

        lendingPool = new LendingPool();
        lendingPool.initialize(address(registry), address(EGGS));

        // Initialize reserves
        token0ReserveId = 1; // First reserve will have ID 1
        lendingPool.initReserve(address(token0));

        token1ReserveId = 2; // Second reserve will have ID 2
        lendingPool.initReserve(address(token1));

        vault = new ShadowRangeVault();
        vault.initialize(
            address(registry),
            address(vaultRegistry),
            address(shadowV3Pool),
            address(positionImpl)
        );

        // Get eToken addresses
        token0Address = lendingPool.getETokenAddress(token0ReserveId);
        token1Address = lendingPool.getETokenAddress(token1ReserveId);

        // Get staking addresses
        stakingToken0Address = lendingPool.getStakingAddress(token0ReserveId);
        stakingToken1Address = lendingPool.getStakingAddress(token1ReserveId);

        // Set price oracle
        registry.setAddress(
            AddressId.ADDRESS_ID_PRICE_ORACLE,
            address(priceOracle)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_VAULT_FACTORY,
            address(vaultRegistry)
        );
        registry.setAddress(
            AddressId.ADDRESS_ID_LENDING_POOL,
            address(lendingPool)
        );
        vaultRegistry.newVault(address(vault));

        vault.setReserveIds(token0ReserveId, token1ReserveId);

        priceOracle.setTokenPrice(address(token0), 0.5 ether);
        priceOracle.setTokenPrice(address(token1), 1 ether);

        deal(address(token0), attacker, 100 ether);
        deal(address(token1), attacker, 100 ether);

        vm.label(attacker, "Attacker");
        vm.label(lender1, "lender1");
        vm.label(lender2, "lender2");

        deal(address(token0), lender1, 1_000_000 ether);
        deal(address(token1), lender1, 1_000_000 ether);
        deal(address(token0), lender2, 1_000_000 ether);
        deal(address(token1), lender2, 1_000_000 ether);
    }

    function testReducingPositionTill98Percent() public {
        //Steps
        // 1. Open a Position
        vm.startPrank(attacker);

        //approve
        token0.approve(address(vault), type(uint256).max);
        token1.approve(address(vault), type(uint256).max);

        IVault.OpenPositionParams memory param = IVault.OpenPositionParams({
            amount0Principal: 10 ether,
            amount1Principal: 0 ether,
            amount0Borrow: 0,
            amount1Borrow: 0 ,
            amount0SwapNeededForPosition: 0,
            amount1SwapNeededForPosition: 0,
            amount0Desired: 10 ether,
            amount1Desired: 0 ether,
            deadline: block.timestamp + 1000,
            tickLower: -1000,
            tickUpper: 1000,
            ul: 0,
            ll: 0
        });
        vault.openPosition(param);
        //fetching and logging the position
        uint256[] memory PositionId = vault.getPositionIds(attacker);
        console2.log("postion ID is: ", PositionId[0]);
        //Fetching and looging the Position Info
        IVault.PositionInfo memory position = vault.getPositionInfos(
            PositionId[0]
        );
        console2.log("Position Info");
        console2.log("Position Info : ", position.owner);
        console2.log("Position ID : ", position.positionId);
        console2.log("Vault ID : ", position.vaultId);
        console2.log("Position Address : ", position.positionAddress);
        console2.log("Shadow Position ID : ", position.shadowPositionId);
        console2.log("Token0 Debt ID : ", position.token0DebtId);
        console2.log("Token1 Debt ID : ", position.token1DebtId);
        console2.log("Tick Lower : ", position.tickLower);
        console2.log("Tick Upper : ", position.tickUpper);
        console2.log("UL : ", position.ul);
        console2.log("LL : ", position.ll);
        console2.log("--------------------------------------------------");

        vm.stopPrank();
        console.log("Position is created Succefully");
        uint256 balanceBeforeToken0 = token0.balanceOf(attacker);
        uint256 balanceBeforeToken1 = token1.balanceOf(attacker);
        console2.log("Balance Before Token0: ", balanceBeforeToken0);
        console2.log("Balance Before Token1: ", balanceBeforeToken1);
        console2.log("--------------------------------------------------");
    }
}
```

### Recommendation
Add a Function to increase liquidity or don't allow 0 liquidity position.
