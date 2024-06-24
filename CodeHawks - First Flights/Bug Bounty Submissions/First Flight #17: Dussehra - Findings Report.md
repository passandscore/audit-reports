# First Flight #17: Dussehra - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. Weak randomness in `ChoosingRam:selectRamIfNotSelected` allows organizer to be the winner.](#L-01)
    - ### [L-02. Funds are locked in `Dussehra.sol` contract  when the `withdraw` function is called. ](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #17

### Dates: Jun 6th, 2024 - Jun 13th, 2024

[See more contest details here](https://www.codehawks.com/contests/clx1ufwjy006g3d8ddjdk3qfr)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 0
   - Low: 2



		


# Low Risk Findings

## <a id='L-01'></a>L-01. Weak randomness in `ChoosingRam:selectRamIfNotSelected` allows organizer to be the winner.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/ChoosingRam.sol#L90

https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/RamNFT.sol#L49

## Summary
The current implementation of randomness in the ChoosingRam:selectRamIfNotSelected function uses a combination of `block.timestamp` and `block.prevrandao` hashed together and then applies modulo `tokenCounter` to determine a random number. This method is predictable, allowing a malicious organizer to potentially manipulate the outcome or predict the result ahead of time. As the organizer may have influence over or foreknowledge of these block values, they can exploit this to ensure a specific participant, including themselves, wins.

## Vulnerability Details
The current random number generation method in ChoosingRam:selectRamIfNotSelected is:

```solidity
uint256 randomIndex = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % tokenCounter;
```
This approach is weak because:

1) block.timestamp can be influenced by the validator within certain bounds.
2) block.prevrandao is accessible and predictable for validators.
3) The tokenCounter can be manipulated through continued NFT minting with the `RamNFT:mintRamNFT` method after the event has closed.

These factors combined allow an organizer to predict or manipulate the outcome, especially when they have knowledge of or control over block parameters.

The organizer if a validator or working with one can know ahead of time the `block.timestamp` and `block.prevrandao` and use that knowledge to predict when to select the winner. See the [solidity blog on prevrando](https://soliditydeveloper.com/prevrandao) here. `block.difficulty` was recently replaced with `prevrandao`. Although the hashed result of these two values then applies modulo `tokenCounter`, the organizer can still manipulate the `tokenCounter` value to result in their index being the winner because the 'RamNFT:mintRamNFT` function is not restricted after the event has ended.

## Impact
If the organizer or an accomplice is a participant, they can manipulate the randomness to select themselves as the winner, leading to unfair outcomes and potential financial loss for other participants. This predictability undermines trust in the event's fairness.

## Tools Used
- Manual Review

## Recommendations
Start by preventing the minting of NFTs using the `RamNFT:mintRamNFT` method once the event has ended. Consider using an oracle for your randomness like [Chainlink VRF](https://docs.chain.link/vrf/v2/introduction). It is capable of providing a secure and verifiable source of randomness on-chain for a number found within a range.
## <a id='L-02'></a>L-02. Funds are locked in `Dussehra.sol` contract  when the `withdraw` function is called.             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/Dussehra.sol#L76

https://github.com/Cyfrin/2024-06-Dussehra/blob/main/src/Dussehra.sol#L46

## Summary
When the `killRavana` function is called in the `Dussehra.sol` contract, the funds are not divided correctly. This results in a remainder of the division that is not accounted for. This remainder will be locked in the contract and not be able to be withdrawn by the organizer. 

## Vulnerability Details
Performing `totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;` in the `Dussehra.sol:killRavana` will not always result in a number that is truly 50% of the total amount. This happens when an entry fee, passed to the contracts constructor in Wei is not divisible by 2 when multiplied by the number of participants. 

## Impact
After the organizer and the selected Ram have received their funds, the remainder of the division will be locked in the contract. This will result in the organizer not being able to withdraw the funds.

## Tools Used
Stateless Fuzz Testing.

**Proof of Concept:** 

1. `Dussehra.sol` is deployed with an entry fee in wei (fuzzed) that is not divisible by 2 when multiplied by the number of participants.
2. Enter participants with the fuzzed entrance fee.
3. Warp to the time when the event is finished
4. Organizer calls `selectRamIfNotSelected` and `killRavana`
5. The funds are divided and the remainder is locked in the contract.
6. Organizer recieves half of the funds during the `killRavana` call.
7. The selected ram calls 'withdraw` and  recieves the other half, while the remainder is locked in the contract.

**Steps to Reproduce:**
1. Create a new test file called `LockedFundsAfterWithDraw.t.sol`
2. Add the code below to the file.
3. Run the test with `forge test --match-path test/LockedFundsAfterWithDraw.t.sol -vvv`



**Proof of Code:** 

<details>
<summary>Code</summary>

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Dussehra} from "../src/Dussehra.sol";
import {ChoosingRam} from "../src/ChoosingRam.sol";
import {RamNFT} from "../src/RamNFT.sol";

contract LockedFundsAfterWithDraw is Test {
    error Dussehra__NotEqualToEntranceFee();
    error Dussehra__AlreadyClaimedAmount();
    error ChoosingRam__TimeToBeLikeRamIsNotFinish();
    error ChoosingRam__EventIsFinished();

    Dussehra public dussehra;
    RamNFT public ramNFT;
    ChoosingRam public choosingRam;

    address public organiser = makeAddr("organiser");
    address public player1 = makeAddr("player1");
    address public player2 = makeAddr("player2");
    address public player3 = makeAddr("player3");

    function deploy(uint256 entranceFee) public {

        vm.startPrank(organiser);
        ramNFT = new RamNFT();
        choosingRam = new ChoosingRam(address(ramNFT));
        dussehra = new Dussehra(entranceFee, address(choosingRam), address(ramNFT));

        ramNFT.setChoosingRamContract(address(choosingRam));
        vm.stopPrank();
    }

    function enterParticipants(uint256 entranceFee) internal {
        vm.startPrank(player1);
        vm.deal(player1, entranceFee);
        dussehra.enterPeopleWhoLikeRam{value: entranceFee}();
        vm.stopPrank();

        vm.startPrank(player2);
        vm.deal(player2, entranceFee);
        dussehra.enterPeopleWhoLikeRam{value: entranceFee}();
        vm.stopPrank();

        vm.startPrank(player3);
        vm.deal(player3, entranceFee);
        dussehra.enterPeopleWhoLikeRam{value: entranceFee}();
        vm.stopPrank();
    }

    function test_withdraw_locks_funds(uint256 entranceFee) public {
        // Set up the contracts with the fuzzed entrance fee
        entranceFee = bound(entranceFee, 0.01 ether, 1 ether);
        deploy(entranceFee);

        // Enter participants with the fuzzed entrance fee
        enterParticipants(entranceFee);

        // Warp to the time when the event is finished
        vm.warp(1728691200 + 1);

        // Select Ram as a winner
        vm.startPrank(organiser);
        choosingRam.selectRamIfNotSelected();
        vm.stopPrank();

        // Determine the winner
        address winner = choosingRam.selectedRam() == player1
            ? player1
            : choosingRam.selectedRam() == player2
                ? player2
                : player3;

        vm.startPrank(winner);
        dussehra.killRavana();

        uint256 RamWinningAmount = dussehra.totalAmountGivenToRam();

        // Check the balance of the organiser
        assertEq(organiser.balance, RamWinningAmount);

        dussehra.withdraw();
        vm.stopPrank();

        // check the balance of the winner
        assertEq(winner.balance, RamWinningAmount);

        // check that the balance of the winner and the organiser is the same
        assertEq(winner.balance, organiser.balance);

        // check that the balance of the contract is 0
        assertEq(address(dussehra).balance, 0 ether);
    }
}
```

</details>

## Recommendations
Performing `totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;` is already a more accurate way to calculate the 50% of the total amount. However, the `killRavana` function should be updated to ensure that after the 50% is calculated, the remainder of contract funds are sent to the organizer. This will prevent any funds from being locked in the contract.

```diff
 function killRavana() public RamIsSelected {
    if (block.timestamp < 1728691069) {
        revert Dussehra__MahuratIsNotStart();
    }
    if (block.timestamp > 1728777669) {
        revert Dussehra__MahuratIsFinished();
    }
    IsRavanKilled = true;
    uint256 totalAmountByThePeople = WantToBeLikeRam.length * entranceFee;
    totalAmountGivenToRam = (totalAmountByThePeople * 50) / 100;

+   uint256 remainder = totalAmountByThePeople - totalAmountGivenToRam;
        
-   (bool success, ) = organiser.call{value: totalAmountGivenToRam}("");
+  (bool success, ) = organiser.call{value: remainder}("");
    require(success, "Failed to send money to organiser");
    }
```


