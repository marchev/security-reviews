# [M-2] Less active nominees can be left without rewards after an year of inactvitiy

## Impact

The `VoteWeighting` within the protocol requires some sort of activity for nominees at least every 53 weeks to maintain its functionality properly. However, some nominees can be less active. Particularly when users lock their veOLAS for long periods (e.g., 4 years) and vote in a set-and-forget manner. Due to the current one-year lookbehind period, the `nomineeRelativeWeight()` function will break for less active nominees after one year (it will return `0`), causing users with longer lock periods to essentially lose their voting power. This undermines the fairness and intended functionality of the protocol, as these users will no longer be able to influence governance or earn rewards despite their significant long-term commitment.

An attempt to address this issue with keepers would not be effective. A malicious user can create an arbitrarily large number of nominees. This would make iterating over all nominees on a regular basis to ensure checkpoints are made extremely gas-expensive, rendering the keepers model impractical and ineffective.

## Proof of Concept

The vulnerability can be observed in the following scenario:

1. A user locks their veOLAS tokens for 4 years and votes for a nominee.
2. The user does not interact with the nominee for over 53 weeks. No other users interact with it either.
3. After this period, when attempting to call the `nomineeRelativeWeight()` function, it will return `0` even though the user has almost 3 years of lock period remaining.
4. As a result, the voting power associated with the user's locked veOLAS is lost.

This issue can be traced to the code handling the lookbehind period for nominees, which does not retain historical data beyond 53 weeks:

https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/governance/contracts/VoteWeighting.sol#L267-L286

The following coded PoC demonstrates the issue. Add the following test case to `VoteWeighting.js`:

```js
        it.only("Nominee can get bigger share", async function () {
            const oneMillionOLAS = ethers.utils.parseEther("1000000");

            // Add nominees
            const numNominees = 2;
            let nominees = [signers[1].address, signers[2].address];
            const chainIds = new Array(numNominees).fill(chainId);
            for (let i = 0; i < numNominees; i++) {
                await vw.addNomineeEVM(nominees[i], chainIds[i]);
            }

            nominees = [convertAddressToBytes32(nominees[0]), convertAddressToBytes32(nominees[1])];

            // Actors
            let charlie = signers[0];
            let alice = signers[10];
            let bob = signers[11];

            // Mint 1M OLAS to Alice & Bob
            await olas.mint(alice.address, oneMillionOLAS);
            await olas.mint(bob.address, oneMillionOLAS);

            // Approve token transfer for locks
            await olas.approve(ve.address, oneMillionOLAS);
            await olas.connect(alice).approve(ve.address, oneMillionOLAS);
            await olas.connect(bob).approve(ve.address, oneMillionOLAS);

            // Lock 1M OLAS into veOLAS for 4 years
            await ve.createLock(oneMillionOLAS, 4 * oneYear);
            await ve.connect(alice).createLock(oneMillionOLAS, 4 * oneYear);
            await ve.connect(bob).createLock(oneMillionOLAS, 4 * oneYear);

            // Charlie votes for nominees[0], whereas alice and bob - for nominees[1]
            await vw.voteForNomineeWeights(nominees[0], chainId, maxVoteWeight);
            await vw.connect(alice).voteForNomineeWeights(nominees[1], chainId, maxVoteWeight);
            await vw.connect(bob).voteForNomineeWeights(nominees[1], chainId, maxVoteWeight);

            // Vote for nominees
            // Get the next point timestamp where votes are written after voting
            const block = await ethers.provider.getBlock("latest");
            const nextTime = getNextTime(block.timestamp);

            // Check weights that must represent a half for each
            for (let i = 0; i < numNominees; i++) {
                const weight = await vw.nomineeRelativeWeight(nominees[i], chainIds[i], nextTime);
                console.log("Vote weights @ lock time: nominees[%s] = %s", i, Number(weight.relativeWeight) / E18);
            }

            // Advance 1 year
            await helpers.time.increase(oneWeek * 52);

            // Alice & Bob vote, Charlie - does not
            await vw.connect(alice).voteForNomineeWeights(nominees[1], chainId, maxVoteWeight);
            await vw.connect(bob).voteForNomineeWeights(nominees[1], chainId, maxVoteWeight);

            for (let i = 0; i < numNominees; i++) {
                const weight = await vw.nomineeRelativeWeight(nominees[i], chainIds[i], nextTime + (oneWeek * 52));
                console.log("Vote weights @ 1 year later: nominees[%s] weight = %s", i, Number(weight.relativeWeight) / E18);
            }

            // Advance 1 more year
            await helpers.time.increase(oneWeek * 2); // Beyond 53 weeks

            await vw.checkpointNominee(nominees[0], chainId); // Relative weight will be broken despite this checkpoint
            await vw.checkpointNominee(nominees[1], chainId);

            for (let i = 0; i < numNominees; i++) {
                const weight = await vw.nomineeRelativeWeight(nominees[i], chainIds[i], nextTime + (oneWeek * 54));
                console.log("Vote weights @ 2 years later: nominees[%s] weight = %s", i, Number(weight.relativeWeight) / E18);
            }
        });
```

## Tools Used

Manual Code Review

## Recommended Mitigation Steps

To mitigate this vulnerability, it is recommended to extend the lookbehind period to at least the maximum lock period plus one additional year. This approach is effectively implemented in Curve's VotingEscrow, where the lookbehind period is 5 years while the maximum lock period is 4 years.
