# Precision loss in ERC2981 royalties calculation results in loss of funds for the royalties receiver

Submitted on Jul 22nd 2024 at 00:48:00 UTC by @marchev for [GhostMarket](https://immunefi.com/bug-bounty/ghostmarket/)

Report ID: #33500

Report type: Smart Contract

Report severity: High

Target: https://etherscan.io/address/0x3dA0bD10dfD98E96E04fbAa8e0512b2c413b096A

Impacts:
- Theft of unclaimed royalties
## Brief/Intro

The GhostMarket supports ERC2981-compliant NFTs. Orders with ERC2981-compliant NFTs result in a loss of funds for the royalties receiver due to precision loss in the calculation of the royalty fee.

## Vulnerability Details

During order matching, the GhostMarket's Exchange contract (`ExchangeV2`) relies on the `RoyaltiesRegistry` contract to fetch the royalties for a given NFT via `RoyaltiesRegistry#getRoyalties(token, tokenId)`.

The `getRoyalties()` function has different strategies to fetch the royalties for a given NFT token. For ERC-2981-complaint NFT tokens, the function relies on the `getRoyaltiesEIP2981()` internal function:

```sol
    function getRoyaltiesEIP2981(address token, uint256 tokenId) internal view returns (LibPart.Part[] memory) {
        try IERC2981(token).royaltyInfo(tokenId, LibRoyalties2981._WEIGHT_VALUE) returns (
            address receiver,
            uint256 royaltyAmount
        ) {
            return LibRoyalties2981.calculateRoyalties(receiver, royaltyAmount);
        } catch {
            return new LibPart.Part[](0);
        }
    }
```

The function fetches the royalty info from the token for a baseline value called `_WEIGHT_VALUE`. Then based on the result of this function call, it calculates the royalty amount as an array of `Part`s by calling `LibRoyalties2981.calculateRoyalties()`. The `Part` structure is used in the protocol as an abstraction to represent royalties as a percentage of an order amount.

The root cause for the vulnerability lies in the implementation of the `LibRoyalties2981.calculateRoyalties()` function:

```sol
    function calculateRoyalties(address to, uint256 amount) internal pure returns (LibPart.Part[] memory) {
        LibPart.Part[] memory result;
        if (amount == 0) {
            return result;
        }
        uint256 percent = ((amount * 100) / _WEIGHT_VALUE) * 100; //@audit Precison loss leads to trunacted percentage
        require(percent < 10000, "Royalties 2981, than 100%");
        result = new LibPart.Part[](1);
        result[0].account = payable(to);
        result[0].value = uint96(percent);
        return result;
    }
```

The percent variable is incorrectly calculated due to the division occurring before the multiplication, which leads to a loss of precision in the decimal part of the percentage. For example, if the royalties percentage is 9.99%, the function will incorrectly calculate it as 9.00%, resulting in a loss of 0.99% of the NFT price for the royalties receiver.

## Impact Details

The miscalculation of the percent variable leads to an underpayment of royalties, causing the royalties receiver to lose out on the decimal portion of their intended earnings.

## References

https://github.com/OnBlockIO/evm-trading-contracts/blob/master/src/royalties/LibRoyalties2981.sol#L20

## Proof of Concept

The following PoC demonstrates the issue:

```sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.20;

import {Test, console} from "forge-std/Test.sol";

contract LossOfRoyaltiesTest is Test {

    uint256 public constant WEIGHT_VALUE = 1000000;

    ExchangeV2 ghostMarketExchangeV2;
    address public quackyDucksNFT;
    uint256 public tokenId;
    uint256 public nftPrice;
    address public royaltiesReceiver;

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth", 20031164);
        ghostMarketExchangeV2 = ExchangeV2(0xfB2F452639cBB0850B46b20D24DE7b0a9cCb665f);
        quackyDucksNFT = address(0x519272821dddb3Afc7aCcD1823B674d30946185E);
        tokenId = 1834;
        nftPrice = 0.008 ether;
        royaltiesReceiver = address(0xbAADc0FfeebaAdC0fFeebaADc0ffeEbAAdC0ffEe);
    }

    function test_loss_of_royalties_due_to_precision_loss() public {
        // Re-using existing sellOrder to avoid redundant setup.
        // We will simulate EIP-2981 compliance of the NFT contract to demonstrate precision loss in royalty calculations.
        // This illustrates how royalty receivers will receive less than expected due to precision loss.
        // Using direct purchase order data from 0xdd5b626dfe77aaa86662e844518be224351ec51f1a5b9142625990c55ea41264
        vm.mockCall(
            quackyDucksNFT,
            abi.encodeCall(IERC2981.supportsInterface, (bytes4(0x2a55205a))),
            abi.encode(bool(true)) // supports ERC2981
        );
        vm.mockCall(
            quackyDucksNFT,
            abi.encodeCall(IERC2981.royaltyInfo, (tokenId, WEIGHT_VALUE)),
            abi.encode(royaltiesReceiver, uint256(99900)) // 9.99% of 1000000 (weight value)
        );

        address alice = makeAddr("alice");
        vm.deal(alice, 1 ether);

        ExchangeV2.Purchase memory directPurchase = ExchangeV2.Purchase({
            sellOrderMaker: address(0x8e49718Bb7Eee02E41746AF8FF03cB7042fF0A76),
            sellOrderNftAmount: 1,
            nftAssetClass: bytes4(0x73ad2146),
            nftData: hex"000000000000000000000000519272821dddb3afc7accd1823b674d30946185e000000000000000000000000000000000000000000000000000000000000072a",
            sellOrderPaymentAmount: nftPrice,
            paymentToken: address(0x0000000000000000000000000000000000000000),
            sellOrderSalt: 2587753978757149132,
            sellOrderStart: 1717655580,
            sellOrderEnd: 1720247580,
            sellOrderDataType: bytes4(0x2fa3cfd3),
            sellOrderData: hex"00000000000000000000000000000000000000000000000000000000000000000000000000000000000000c807714a8bf073510996d948d8aa39f8e32627fe62000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c8d52e07e200000000000000000000000000000000000000000000000000000000",
            sellOrderSignature: hex"8605ad7a6a08539cf2647f9bff6b64365e81256e30e3b38532099606111fc05c7fcd8ece4a205674e50e8cd9db4a106fb7ae53950a58b882b547ada860142b861b",
            buyOrderPaymentAmount: nftPrice,
            buyOrderNftAmount: 1,
            buyOrderData: hex"0000000000000000000027108e49718bb7eee02e41746af8ff03cb7042ff0a7600000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000d52e07e200000000000000000000000000000000000000000000000000000000"
        });

        vm.prank(alice);
        ghostMarketExchangeV2.directPurchase{value: 0.008 ether}(directPurchase);

        uint256 expected = (999 * nftPrice) / 100_00;
        uint256 actual = royaltiesReceiver.balance;
        uint256 loss = expected - actual;
        console.log("Royalties received: expected=%e actual=%e loss=%e", expected, actual, loss);
    }
}

interface IERC2981 {

    function royaltyInfo(uint256 _tokenId, uint256 _salePrice) external view returns (address receiver,uint256 royaltyAmount);
    
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}

interface ExchangeV2 {

    struct Purchase {
        address sellOrderMaker;
        uint256 sellOrderNftAmount;
        bytes4 nftAssetClass;
        bytes nftData;
        uint256 sellOrderPaymentAmount;
        address paymentToken;
        uint256 sellOrderSalt;
        uint256 sellOrderStart;
        uint256 sellOrderEnd;
        bytes4 sellOrderDataType;
        bytes sellOrderData;
        bytes sellOrderSignature;
        uint256 buyOrderPaymentAmount;
        uint256 buyOrderNftAmount;
        bytes buyOrderData;
    }

    function directPurchase(Purchase memory direct) external payable;
}
```

The PoC uses an existing order to simplify the setup code. It also uses `vm.mockCall()` to simulate the Quacky Ducks NFT being an ERC2981-compliant token.

This is the output of the PoC:

```sh
Ran 1 test for test/LossOfRoyalties.t.sol:LossOfRoyaltiesTest
[PASS] test_loss_of_royalties_due_to_precision_loss() (gas: 235334)
Logs:
  Royalties received: expected=7.992e14 actual=7.2e14 loss=7.92e13
```
