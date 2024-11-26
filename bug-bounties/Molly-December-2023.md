# Incorrect fee calculation results in loss of funds for private sale buyers

Submitted on Dec 27th 2023 at 16:42:00 UTC by @marchev for [Molly](https://immunefi.com/bounty/molly/)

Report ID: #27297

Report type: Smart Contract

Report severity: High

Target: https://etherscan.io/address/0x0065a7f9f136d2d77d81ee371c72fd5c8c079237

Impacts:
- Temporary freezing of funds
## Brief/Intro

During the vesting period in the MollyV2 protocol, there is an issue where private sale buyers are inaccurately charged the angel investor fee instead of their designated dynamic fee. This error results in private sale buyers incurring higher fees and experiencing an extended vesting period of 120 days, rather than the intended 90 days.

The MollyV2 protocol implements two distinct dynamic fee structures:

- **Angel Investor Dynamic Fee** - This fee starts at 90% and progressively decreases to 0% over a span of 120 days.
- **Private Sale Buyers Dynamic Fee** - This fee begins at 80% and similarly reduces to 0%, but over a shorter duration of 90 days.

The problem arises due to the application of the angel investor fee for both angel investors and private sale buyers, leading to higher fees and longer vesting period for private sale buyers.

## Vulnerability Details

The issue is rooted in the contract code responsible for applying dynamic fees:

```solidity
	// ...
	// on sell
	
	if (_getStorage().automatedMarketMakerPairs[to] && _getStorage().sellFees > 0) {
		if (_getStorage().isAngelBuyer[from]) {
			additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;
			if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
		} else if (_getStorage().isPrivateSaleBuyer[from]) {
			additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR; // @audit Wrong fee applied here.
			if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
		}
		fees = (amount * _getStorage().sellFees) / FEE_DENOMINATOR;
	}
	// on buy
	else if (_getStorage().automatedMarketMakerPairs[from] && _getStorage().buyFees > 0) {
		if (_getStorage().isAngelBuyer[to]) {
			additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;
			if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
		} else if (_getStorage().isPrivateSaleBuyer[to]) {
			additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;  // @audit Wrong fee applied here.
			if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
		}
		fees = (amount * (_getStorage().buyFees)) / FEE_DENOMINATOR;
	}
	// ...
```

The `getCurrentAngelFee()` function is erroneously used for calculating fees for both angel investors and private sale buyers, instead of using the `getCurrentFee()` function for private sale buyers.

As a matter of fact the `getCurrentFee()` function which should be used for calculating the fee for private sale buyers is never used.

## Impact Details

This vulnerability has two significant consequences:

- **Financial Loss for Private Sale Buyers:** They are subject to higher fees, leading to a direct financial loss.
- **Extended Vesting Period:** Their tokens are subject to additional fees for 120 days instead of the documented 90 days.

The additional fee charged is transferred to the `controllerWallet`. If the protocol owner cooperates, it may be possible to recover these funds. Consequently, the severity of this issue is assessed as **High: Temporary Freezing of Funds**.

Consider this scenario which illustrates the impact:

1. Alice, holding 1000 MOLLY tokens as a private sale buyer, attempts to exchange them for WETH on Uniswap 90 days post-launch.
2. Due to the incorrect fee application, she incurs a 22.5% fee, resulting in a loss of 225 MOLLY tokens.

For a visual representation of the fee discrepancy, refer to the Desmos Calculator: [https://www.desmos.com/calculator/wzetainkoy](https://www.desmos.com/calculator/wzetainkoy)

## Risk Breakdown

The likelihood of this vulnerability materializing is high, especially since private sale buyers anticipate no additional fees after the 90th day from the protocol's launch.

## Recommendation

Apply the correct fee for private sale buyers:

```diff
@@ -701,7 +701,7 @@ contract MollyV2 is ERC20Upgradeable, Ownable2StepUpgradeable, EIP712, UUPSUpgra
                     additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;
                     if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
                 } else if (_getStorage().isPrivateSaleBuyer[from]) {
-                    additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;
+                    additionalFees = (amount * getCurrentFee()) / FEE_DENOMINATOR;
                     if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
                 }
                 fees = (amount * _getStorage().sellFees) / FEE_DENOMINATOR;
@@ -712,7 +712,7 @@ contract MollyV2 is ERC20Upgradeable, Ownable2StepUpgradeable, EIP712, UUPSUpgra
                     additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;
                     if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
                 } else if (_getStorage().isPrivateSaleBuyer[to]) {
-                    additionalFees = (amount * getCurrentAngelFee()) / FEE_DENOMINATOR;
+                    additionalFees = (amount * getCurrentFee()) / FEE_DENOMINATOR;
                     if (additionalFees > 0) _transfer(from, _getStorage().controllerWallet, additionalFees);
                 }
                 fees = (amount * (_getStorage().buyFees)) / FEE_DENOMINATOR;
```

## Proof of Concept

The following is a PoC which highlights the problem and quantifies the loss of MOLLY tokens by a private sale buyer executing a swap on the 90th day, under the assumption of no additional fees.

The PoC is based on the `default` template from `immunefi-team/forge-poc-templates`.

To run the PoC, initialize a repo:

```sh
forge init --template immunefi-team/forge-poc-templates --branch template
```

Then create a new file `test/MollyTest.sol` with the following contents:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "@immunefi/src/PoC.sol";

import "forge-std/console2.sol";

contract MollyTest is PoC {
    address public owner; // MollyV2 owner
    address public controllerWallet; // Collects additional fees
    address public alice; // Private sale buyer
    MollyV2 public molly;
    UniswapV2Router02 public uniswapV2Router;
    IERC20 WETH;

    function setUp() public {
        //vm.createSelectFork("https://rpc.ankr.com/eth_goerli"); 
        vm.createSelectFork("https://eth-goerli.g.alchemy.com/v2/PzxZ3FVy3dgufKhHk89N7NM3yb3qTbO-");

        // Add some labels for improved test traces readability 
        vm.label(address(0xB4FBF271143F4FBf7B91A5ded31805e42b2208d6), "WETH");
        vm.label(address(0xbcC6f48520fE690559bcec7E17E0C5e2f791FE8D), "MollyV2");
        vm.label(address(0xA5DDA2987c614aD62dE221441FbfEE3D18B26181), "UniswapV2Router");
        vm.label(address(0x7A0299E2a9Df4e1144bB5bf7Ac0fA1E3d7cbcc61), "MollyV2Impl");
        vm.label(address(0x65849de03776Ef05A9C88E367B395314999826ed), "ControllerWallet");

        owner = address(0x800853a0D3ad9Ac29c6b8a89A6318ABb7EaeC8A0);
        controllerWallet = address(0x65849de03776Ef05A9C88E367B395314999826ed);
        alice = makeAddr("alice");

        molly = MollyV2(0xbcC6f48520fE690559bcec7E17E0C5e2f791FE8D);
        uniswapV2Router = UniswapV2Router02(payable(molly.UNISWAP_V2_ROUTER()));
        WETH = IERC20(address(uniswapV2Router.WETH()));
    }

    function testPrivateSaleBuyersGetChargedMore() public {
        // Molly needs some ETH to open trading on Uniswap
        vm.deal(address(molly), 1_000 ether);

        vm.startPrank(owner);
        address[] memory addresses = new address[](2);
        addresses[0] = alice;
        addresses[1] = address(molly);
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100 ether;
        amounts[1] = 1_000 ether;
        // Distribute MOLLY tokens:
        // Alice -> 100e18
        // Molly -> 1_000e18
        molly.distributeTokens(addresses, amounts);

        // Set alice as private sale buyer
        molly.bulkSetPrivateBuyers(addresses, true);

        // Open trading with 1_000 ETH.
        molly.openTrading(1_000 ether);
        vm.stopPrank();

        // Save a snapshot to compare the fees on 90th day vs 120th day
        uint256 snapshot = vm.snapshot();
        
        _dash();
        _log("            Alice swaps 100e18 MOLLY tokens for WETH (Actual)");
        // Fast forward 90 days after START_DATE. As per the documentation, the additional fee
        // for private sale buyers should be 0% at this point.
        uint256 actualFeeCharged = swapAllOfAliceMollyTokens(90 days);


        // As a comparison, perform the swap 120 days after START_DATE. This demonstrates
        // that the angel fee fee is used for private sale buyers instead.
        vm.revertTo(snapshot);
        _dash();
        _log("            Alice swaps 100e18 MOLLY tokens for WETH (Expected)");
        uint256 expectedFeeCharged = swapAllOfAliceMollyTokens(120 days);

        uint256 loss = actualFeeCharged - expectedFeeCharged;
        _dash();
        _log("           Alice's loss: ~", loss, "MOLLY (in wei)");
    }

    function swapAllOfAliceMollyTokens(uint256 daysAfterStart) internal returns (uint256 feeCharged) {
        vm.roll(block.number + 1e6);
        vm.warp(molly.START_DATE() + daysAfterStart + 1 hours);

        _dash();
        _log("Alice's MOLLY balance before swap:", molly.balanceOf(alice));
        _log("Alice's WETH balance before swap:", WETH.balanceOf(alice));
        _log("ControllerWallet's MOLLY balance before swap:", molly.balanceOf(controllerWallet));

        // Alice swaps all her 100e18 MOLLY tokens for WETH
        vm.startPrank(alice);
        molly.approve(address(uniswapV2Router), type(uint256).max);
        address[] memory path = new address[](2);
        path[0] = address(molly);
        path[1] = address(WETH);
        uniswapV2Router.swapExactTokensForTokensSupportingFeeOnTransferTokens(
            100 ether,
            0,
            path,
            alice,
            block.timestamp
        );
        vm.stopPrank();

        _dash();
        _log("Alice's MOLLY balance after swap:", molly.balanceOf(alice));
        _log("Alice's WETH balance after swap:", WETH.balanceOf(alice));
        _log("ControllerWallet's MOLLY balance after swap:", molly.balanceOf(controllerWallet));
        feeCharged = molly.balanceOf(controllerWallet);
    }
}

interface MollyV2 {
    function START_DATE() external view returns (uint256);
    function UNISWAP_V2_ROUTER() external view returns (address);
    function approve(address spender, uint256 value) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function bulkSetPrivateBuyers(address[] memory _addresses, bool _state) external;
    function distributeTokens(address[] memory addresses, uint256[] memory amounts) external;
    function openTrading(uint256 _amount) external payable;
}

interface UniswapV2Router02 {
    function WETH() external view returns (address);
    function swapExactTokensForTokensSupportingFeeOnTransferTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] memory path,
        address to,
        uint256 deadline
    ) external;
}
```

The output in the presented scenario is as follows:

```sh
Running 1 test for test/MollyTest.sol:MollyTest
[PASS] testPrivateSaleBuyersGetChargedMore() (gas: 827089)
Logs:
  ---------------------------------------------------------------------------
              Alice swaps 100e18 MOLLY tokens for WETH (Actual)
  ---------------------------------------------------------------------------
  Alice's MOLLY balance before swap: 100000000000000000000
  Alice's WETH balance before swap: 0
  ControllerWallet's MOLLY balance before swap: 0
  ---------------------------------------------------------------------------
  Alice's MOLLY balance after swap: 0
  Alice's WETH balance after swap: 65672479896775495578
  ControllerWallet's MOLLY balance after swap: 22500000000000000000
  ---------------------------------------------------------------------------
              Alice swaps 100e18 MOLLY tokens for WETH (Expected)
  ---------------------------------------------------------------------------
  Alice's MOLLY balance before swap: 100000000000000000000
  Alice's WETH balance before swap: 0
  ControllerWallet's MOLLY balance before swap: 0
  ---------------------------------------------------------------------------
  Alice's MOLLY balance after swap: 0
  Alice's WETH balance after swap: 84853315713709171874
  ControllerWallet's MOLLY balance after swap: 0
  ---------------------------------------------------------------------------
             Alice's loss: ~ 22500000000000000000 MOLLY (in wei)
```