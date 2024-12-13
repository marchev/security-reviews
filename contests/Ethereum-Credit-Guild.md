# [M-1] `AuctionHouse` Whales can win large auctions via block stuffing

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/AuctionHouse.sol#L144-L152

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/test/proposals/gips/GIP_0.sol#L175-L179
## Impact

Whales (accounts with significant cryptocurrency holdings) can exploit the current auction mechanics to guarantee profits through block stuffing.

This is particularly feasible due to the short duration of auctions, allowing these entities to manipulate auction outcomes in their favor.

## Proof of Concept

The protocol uses a Dutch auction format with two phases:

- **First Phase**: Bidders must pay the full debt amount. The collateral percentage starts at 0% and increases with each new block until the auction's midpoint.
- **Second Phase**: The protocol offers the full collateral and decreases the owed debt by a percentage in each new block, reaching 0% at auction's end. This implies that a bidder could eventually receive the collateral for free.

Bidders are disincentivized to participate in the first phase, as it generally results in a net loss unless there are force majeure market conditions.

Whales can use a block stuffing attack to win large auctions and acquire collateral at significantly reduced prices.

Example scenario:

1. Alice has the following bad debt which is auctioned off:
	- Collateral: 2,000,000 USDC
	- Debt: 1,000 WETH (1 WETH = 2,000 USDC)
2. There are no bidders in the first phase of the auction since this would result in a loss for the bidder.
2. The auction reaches its midpoint. Collateral cost reduces by ~1.14% every block. This is because the second phase is 1150 sec. (as per the `GIP_0.sol` deployment script), i.e. 88 blocks on Mainnet. The decay rate is thus 100% / 88 ~ 1.14%
3. At the auction's midpoint, Bob executes block stuffing attack. To make sure his attack would succeed, he uses a gas price of 250 Gwei.
4. After 88 blocks, Bob binds in the final block and wins 2,000,000 USDC at 0 ETH cost. The attack cost is: 
$$
88\ blocks \times 30M\ gas\ \times 250\ Gwei = 66,000,000,000\ Gwei = 660\ ETH
$$
	which is ~$1.5M at current ETH prices. Thus, Bob has made a profit of 500,000 USDC.

This strategy, while requiring substantial funds, is a feasible and potentially lucrative attack vector.

The severity is set to Medium given its low likelihood but high impact.
## Tools Used

Manual review
## Recommended Mitigation Steps

To mitigate this vulnerability, it is recommended to extend the auction duration. Longer auctions would increase the cost and complexity of block stuffing attacks, reducing the likelihood of such exploits.

# [M-2] `LendingTerm` Inconsistency between debt ceiling as calculated in `borrow()` and `debtCeiling()`
  
## Impact  
There is an inconsistency between how [borrow]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397)) calculates the `debtCeiling` and how it is calculated in [debtCeiling]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270)).  
  
## Proof of Concept  
  
Currently, there is an inconsistency in the way [borrow]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397)) calculates a smaller value for [debtCeiling]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270)) than the actual [debtCeiling]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270)) function. This renders [this check useless]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L225-L231](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/tokens/GuildToken.sol#L225-L231)) since borrow prevents issuance from going even close to the debt ceiling.  
  
```solidity  
uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));  
require(issuance <= debtCeilingAfterDecrement, "GuildToken: debt ceiling used");  
```  
  
For the example, I am going to use the parameters below, and we will follow with the two calculations to get the results.  
  
| **Prerequisites**      | **Values**                                |  
|------------------------|------------------------------------------|  
| Issuance               | 20,000                                   |  
| Total borrowed credit  | 70,000 (5k when we call borrow / 65k old)|  
| Total weight           | 100,000                                  |  
| Gauge weight           | 50,000                                   |  
| Gauge weight tolerance | 60% (1.2e18)                              |  
  
[borrow]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L394-L397)) calculates the debt `debtCeiling` with the following formula (found [here]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L383-L387)](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L383-L387))):  
  
$$  
\begin{align*}  
\text{debtCeiling} &= \frac{{(\text{getGaugeWeight} \cdot (\text{totalBorrowedCredit} + \text{borrowAmount}))}}{{\text{totalWeight}}} \cdot \frac{{1.2e18}}{{1e18}} \\  
&= \frac{{50000e18 \cdot (65000e18+5000e18)}}{{100000e18}} \cdot \frac{{1.2e18}}{{1e18}} \\  
&= \frac{{50000e18 \cdot 70000e18}}{{100000e18}} \cdot \frac{{1.2e18}}{{1e18}} \\  
&= \frac{{35000e18 \cdot 1.2e18}}{{1e18}} \\  
&= 42000e18  
\end{align*}  
$$  
  
At the same time, [debtCeiling]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270)) uses the complicated formula below, which we will break down for easier understanding.  
  
```solidity  
debtCeiling = ((((totalBorrowedCredit * (getGaugeWeight * 1.2e18)) / totalWeight) - issuance) * totalWeight / otherGaugesWeight) + issuance  
```  
  
Feel free to follow with the [code]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270-L331](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L270-L331)).  
  
#### [toleratedGaugeWeight]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L305](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L305))  
  
$$  
\begin{align*}  
\text{toleratedGaugeWeight} &= \frac{{\text{gaugeWeight} \cdot \text{gaugeWeightTolerance}}}{{1e18}} \\  
&= \frac{{50000e18 \cdot 1.2e18}}{{1e18}} \\  
&= 60000e18  
\end{align*}  
$$  
  
#### [debtCeilingBefore]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L307-L308](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L307-L308))  
  
$$  
\begin{align*}  
\text{debtCeilingBefore} &= \frac{{\text{totalBorrowedCredit} \cdot \text{toleratedGaugeWeight}}}{{\text{totalWeight}}} \\  
&= \frac{{70000e18 \cdot 60000e18}}{{100000e18}} \\  
&= 42000e18  
\end{align*}  
$$  
  
#### [remainingDebtCeiling]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L312](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L312))  
  
$$  
\begin{align*}  
\text{remainingDebtCeiling} &= \text{debtCeilingBefore} - \text{issuance} \\  
&= 42000e18 - 20000e18 \\  
&= 22000e18  
\end{align*}  
$$  
  
#### [otherGaugesWeight]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L319](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L319))  
  
$$  
\begin{align*}  
\text{otherGaugesWeight} &= \text{totalWeight} - \text{toleratedGaugeWeight} \\  
&= 100000e18 - 60000e18 \\  
&= 40000e18  
\end{align*}  
$$  
  
#### [maxBorrow]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L320-L321](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L320-L321))  
  
$$  
\begin{align*}  
\text{maxBorrow} &= \frac{{\text{remainingDebtCeiling} \cdot \text{totalWeight}}}{{\text{otherGaugesWeight}}} \\  
&= \frac{{22000e18 \cdot 100000e18}}{{40000e18}} \\  
&= 55000e18  
\end{align*}  
$$  
  
#### [debtCeiling]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L322](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/LendingTerm.sol#L322))  
  
$$  
\begin{align*}  
\text{debtCeiling} &= \text{issuance} + \text{maxBorrow} \\  
&= 55000e18 + 20000e18 \\  
&= 75000e18  
\end{align*}  
$$  
  
Finally, after the long pursuit, we have come to our answer. However, this answer (75k) differs from what we can max borrow (42k).  
  
## Tools Used  
Manual review.  
  
## Recommended Mitigation Steps  
Implement one function to calculate `debtCeiling`.  
  
  
## Assessed type  
  
Error

# [M-3] Use of `block.number` will not work on Base

[https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L36](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L36)

## Impact  
The use of `block.number` will not work on Base.  
  
## Proof of Concept  
  
Developers clarified that ECG will be deployed on the ETH mainnet and Layer2s , such as ARB, OP and Base.  

> @Eswak, which other L2 chains do you plan to use?  

> Probably Optimism, Base, etc.  
  
However, in the Base [docs]([https://www.coinbase.com/en-gb/cloud/discover/protocol-guides/guide-to-base](https://www.coinbase.com/en-gb/cloud/discover/protocol-guides/guide-to-base)), it is explained that blocks are produced every 2 seconds, not 13 like mainnet.  
  
> Base blocks are produced every two seconds, regardless of whether they are empty (no transactions), filled up to the block gas limit with transactions, or somewhere in the middle.  
  
Considering this information, [LendingTermOffboarding]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol)) will not function correctly, as [proposeOffboard]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L89-L102](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L89-L102)) and [supportOffboard]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L116-L148](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L116-L148)) depend on [POLL_DURATION_BLOCKS]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L36](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L36)), which is set to 46523 blocks (equivalent to ~7 days with 13-second blocks).  
  
[supportOffboard]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L116-L148](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L116-L148)) will expire on the 25th hour (not day 7), significantly reducing the time users have to vote.  
  
```solidity  
require(  
   block.number <= snapshotBlock + POLL_DURATION_BLOCKS,  
   "LendingTermOffboarding: poll expired"  
);  
```  
  
While [proposeOffboard]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L89-L113](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L89-L113)) can propose a new offBoarding every ~25 hours, which is not the expected or desired behavior.  
  
```solidity  
require(  
   block.number > lastPollBlock[term] + POLL_DURATION_BLOCKS,  
   "LendingTermOffboarding: poll active"  
);  
```  
  
## Tools Used  
Manual review.  
  
## Recommended Mitigation Steps  
Make [POLL_DURATION_BLOCKS]([https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L36](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/LendingTermOffboarding.sol#L36)) immutable rather than constant.  
  
```diff  
-    uint256 public constant POLL_DURATION_BLOCKS = 46523;  
+    uint256 public immutable pollDurationBlocks;  
  
-   constructor(...) CoreRef(_core) {  
+   constructor(..., uint256 pollDuration) CoreRef(_core) {  
       guildToken = _guildToken;  
       psm = _psm;  
       quorum = _quorum;  
+       pollDurationBlocks = pollDuration;  
   }  
```  
  
  
## Assessed type  
  
Error