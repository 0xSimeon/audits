# First Flight #1: PasswordStore - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. [H-0] Missing access control in `PasswordStore::setPassword` allows anyone to set a new password](#H-01)
    - ### [H-02. [M-0] Incorrect assumption about privacy makes sensitive, private data publicly available](#H-02)

- ## Low Risk Findings
    - ### [L-01. [L-0] Redundant checks should be moved to a modifier](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. [H-0] Missing access control in `PasswordStore::setPassword` allows anyone to set a new password            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L26

## Summary
Upon deployment of the contract, there is an assumption that only the owner `s_owner` can set and retrieve a password. But the missing access control check in: https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L26 means anyone can call `setPassword` and update the password which is not expected by the owner. 

## Vulnerability Details
In `setPassword`, there is no check if the caller of the method is actually the owner's address stored in `s_owner` during deployment. This results in the `s_password` storage variable being updated by anyone. 

## Impact
Anyone can update the password at any time. all access pass. 

## POC 
Add `test_non_owner_can_update_password()` below to `passwordStore.t.sol`

  ```
function test_non_owner_can_update_password() public {
        address randomUser = makeAddr("randomUser");
        // call setPassword using a random address and update the stored password. 
        vm.startPrank(randomUser);
        passwordStore.setPassword("anyone_can_set_new_password");
        vm.stopPrank();

        // check if password was updated by non-owner. 
        vm.prank(owner);
        assertEq(passwordStore.getPassword(), "anyone_can_set_new_password");
  }
```

## Tools Used
Manual review.

## Recommendations
Add a check for the caller of the `setPassword()` function: 
```diff 
+   error PasswordStore__CallerIsNotOwner();
....
......

function setPassword(string memory newPassword) external {
+     if (msg.sender != s_owner) {
+       revert PasswordStore__CallerIsNotOwner();
+      }
        s_password = newPassword;
        emit SetNetPassword();
    }

```

## <a id='H-02'></a>H-02. [M-0] Incorrect assumption about privacy makes sensitive, private data publicly available            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L14C2-L14C31

## Summary
The developer of the PasswordStore smart contract has the assumption that using _private_ state variables will add an extra layer of privacy for password stored in the contract.

## Vulnerability Details
The state variable - `s_password` has the private visibility. This means it is only accessable via the **PasswordStore** contract and not in inherited contracts or externally but the all data stored in a public blockchain are publicly viewable. 

## Impact
Sensitive data stored in `s_password` is pulblicly viewable on-chain thus, anyone can access it and use it for an exploit directly affecting the deployer/owner. 

## Tools Used
Manual Review 

## POC
Not required. 

## Recommendations
Usage of encryption techniques or simply hashing the password off-chain using the _keccak256_ hashing algorithm and storing the hash output adds an extra layer of privacy for sensitive data but not the best solution as the keccak256 algo is deterministic (same input always produce same output). However, It should be helpful in this context. 
		


# Low Risk Findings

## <a id='L-01'></a>L-01. [L-0] Redundant checks should be moved to a modifier            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/856ed94bfcf1031bf9d13514cb21b591d88ed323/src/PasswordStore.sol#L36C2-L36C2

## Summary
Redundant checks should be converted into a modifier. 

## Vulnerability Details
N/A

## Impact
N/A 


## Tools Used
Manual review. 

## Recommendations

It is considered best practice for checks such as the one below to be converted to a modifier. Better yet, the modifier logic moved to a function that gets called in the modifier. 
```
if (msg.sender != s_owner) {
     revert PasswordStore__CallerIsNotOwner();
}
```




