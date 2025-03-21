### [H-1] Missing constructor in the contracts.

**Description:** The `Blueprint` contracts are missing constructor with `_disableInitializers` function, as written In the openzeppelin docs

```
An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke the _disableInitializers function in the constructor to automatically lock it when it is deployed:  
```


**Impact:**  The contract lacks proper initialization protection, allowing an attacker to reinitialize it

**Recommended Mitigation:** 

```solidity 
constructor() {  
    _disableInitializers();  
}  
```

### [M-1] Unauthorized Project Data Reset Vulnerability

**Description:** The `BlueprintCore::upgradeProject()` function allows anyone to reset a project's details, including `requestProposalID`, `requestDeploymentID`, and `proposedSolverAddr`, to predefined values. This function lacks an ownership check, enabling any user to modify project data, even if they do not own the project.

**Impact:** An attacker can arbitrarily reset the details of any project, disrupting operations and potentially leading to data corruption or loss of critical information.

**Proof of Concept:**
```solidity 

    function upgradeProject(bytes32 projectId) public hasProject(projectId) {
        // reset project info
        projects[projectId].requestProposalID = 0;
        projects[projectId].requestDeploymentID = 0;
        projects[projectId].proposedSolverAddr = dummyAddress;
    }

```
https://github.com/sherlock-audit/2025-03-crestal-network/blob/27a3c28155702b3a68f29347efedffb048010e33/crestal-omni-contracts/src/BlueprintCore.sol#L198C1-L204C1

**Recommended Mitigation:** Introduce an ownership validation check by ensuring `msg.sender` matches the project owner before allowing project data to be reset.