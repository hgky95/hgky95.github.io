---
title: Role Management In Solidity
categories: [blockchain]
date: 2024-10-12 17:00:00
tags: [smart-contract, solidity, blockchain]
image: "/assets/img/role_mgmt/poster.png"
---
In Solidity, managing permissions across multiple smart contracts can be challenging, especially when multiple contracts need to share role-based access control (RBAC). A common misconception when working with role management is misunderstanding how permissions are applied across different contracts and how the **msg.sender** behaves when calls are made through other contracts.

In this post, I’ll share a problem I encountered when trying to assign roles across two contracts (`MilestoneManager` and `ProposalManager`), and how I solved it using role delegation with an `AdminManager` contract.

## The Problem: Misunderstanding Role Assignment Across Contracts
![Image](/assets/img/role_mgmt/role_mgmt.png)

Initially, I had two contracts `MilestoneManager.sol` and `ProposalManager.sol` both inheriting from a parent contract `RoleManager.sol` that used Solidity’s AccessControl mechanism. The RoleManager contract defined roles such as `STUDENT_ROLE` and `DEFAULT_ADMIN_ROLE`. Both child contracts had methods to add students and committee members via their own versions of the addStudent and addCommittee functions.
```
contract RoleManager is AccessControl {
    bytes32 public constant STUDENT_ROLE = keccak256("STUDENT_ROLE");
    bytes32 public constant COMMITTEE_ROLE = keccak256("COMMITTEE_ROLE");

    constructor(address initialAdmin) {
        _grantRole(DEFAULT_ADMIN_ROLE, initialAdmin);
    }

    function addStudent(address student) public onlyRole(DEFAULT_ADMIN_ROLE) {
        _grantRole(STUDENT_ROLE, student);
    }

    function addCommittee(address member) public onlyRole(DEFAULT_ADMIN_ROLE) {
        _grantRole(COMMITTEE_ROLE, member);
    }
}

contract MilestoneManager is RoleManager {
    // Milestone-specific logic
}

contract ProposalManager is RoleManager {
    // Proposal-specific logic
}
```

I initially assumed that calling `addStudent` in `ProposalManager` would automatically assign the `STUDENT_ROLE` in both `ProposalManager` and `MilestoneManager`. I believed that because both contracts inherited from `RoleManager`, the student role would be applied across both.

However, this was incorrect for two reasons:
- **Role Assignments Are Contract-Specific**: In Solidity’s AccessControl, roles are specific to the contract where they are granted. So, assigning a role in `ProposalManager` does not automatically assign that role in `MilestoneManager`, even though they share the same `RoleManager` base contract.

- **Sender Context (msg.sender)**: When I called the addStudent method on `ProposalManager` to assign a role in `MilestoneManager`, I overlooked the fact that `msg.sender` in `MilestoneManager` would be `ProposalManager` (the contract making the call), not the account that initiated the transaction. Since `ProposalManager` did not have admin permissions in `MilestoneManager`, the call failed.

## The Solution: Role Delegation Using AdminManager
To resolve this issue, I needed a way to assign roles in both `ProposalManager` and `MilestoneManager` with a single call. The solution was to create a separate `AdminManager` contract that could manage roles across both contracts. By delegating the role management to `AdminManager`, I ensured that roles could be granted in both contracts simultaneously without running into `msg.sender` permission issues.

### AdminManager Contract
The `AdminManager` contract is responsible for handling role assignments for both `ProposalManager` and `MilestoneManager`. This contract holds the `DEFAULT_ADMIN_ROLE` for both contracts and can manage student and committee roles as needed.

Here’s the implementation of the AdminManager contract:
```
contract AdminManager {
    MilestoneManager public milestoneManager;
    ProposalManager public proposalManager;

    constructor(address _milestoneManager, address _proposalManager) {
        milestoneManager = MilestoneManager(_milestoneManager);
        proposalManager = ProposalManager(_proposalManager);
    }

    function addStudentToBoth(address student) public {
        milestoneManager.addStudent(student); // Assign student role in MilestoneManager
        proposalManager.addStudent(student); // Assign student role in ProposalManager
    }

    function addCommitteeToBoth(address committeeMember) public {
        milestoneManager.addCommittee(committeeMember); // Assign committee role in MilestoneManager
        proposalManager.addCommittee(committeeMember); // Assign committee role in ProposalManager
    }
}

```

### Granting Permissions
To enable `AdminManager` to manage roles in both contracts, I needed to grant the `DEFAULT_ADMIN_ROLE` to the `AdminManager` contract in both `MilestoneManager` and `ProposalManager`.
```
// Deployer grants admin role to AdminManager in both contracts
milestoneManager.grantRole(DEFAULT_ADMIN_ROLE, address(adminManager));
proposalManager.grantRole(DEFAULT_ADMIN_ROLE, address(adminManager));
```

### The Result
Now, when I call `addStudentToBoth` from `AdminManager`, the student role is applied in both `MilestoneManager` and `ProposalManager` simultaneously. This avoids the issue of contract-specific roles and bypasses the `msg.sender` issue by centralizing role assignment logic in a single contract.