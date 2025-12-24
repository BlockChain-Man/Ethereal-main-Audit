# Ethereal Contract – Security Review Findings

## Summary
This review covers the **Ethereal NFTfi protocol**, focusing on minting, redemption, fee accounting, and external asset handling (ETH / wstETH).  
Several **critical logic flaws** were identified that can break core functionality or permanently lock funds.

Below is a link for the **Ethereal NFTfi protocol** repo:

https://github.com/owenThurm/Ethereal

---

## Critical Severity

### C-01: Incorrect wstETH Minting Logic
**Location:** `_mintWstEth`

**Description**  
The contract attempts to mint wstETH by sending native ETH directly to the `wstETH` contract:

```solidity
(bool success,) = wstETH.call{value: msg.value}("");
require(success, "Failed to deposit Ether");
```

However, `wstETH` is an **ERC20 token contract**, not a payable deposit wrapper. Sending ETH to it will revert or have no effect.

**Impact**  
- Minting wstETH-backed Gems **always fails**  
- Entire wstETH-backed minting path is non-functional

**Recommendation**  
Implement the correct Lido flow:
- Convert ETH → stETH
- Wrap stETH → wstETH
- Use the proper interfaces instead of raw ETH transfer

---

### C-02: Fee Accounting Breaks for wstETH Redemptions
**Location:** `_redeemWstEth`, `withdrawFees`

**Description**  
Fees from both ETH-backed and wstETH-backed redemptions are accumulated into a single variable:

```solidity
uint256 public fees;
```

During wstETH redemption:
```solidity
fees += redeemFee;
IwstETH(wstETH).transfer(msg.sender, amount);
```

But `withdrawFees()` attempts to withdraw **ETH only**:

```solidity
payout.call{value: fees}("");
```

**Impact**  
- wstETH-denominated fees become **irrecoverable**  
- `fees` mixes asset types and loses meaning

**Recommendation**  
Separate accounting:
```solidity
uint256 public ethFees;
uint256 public wstEthFees;
```
Provide a dedicated `withdrawWstEthFees()`.

---

### C-03: Redeem Fee Calculation Does Not Match Documentation
**Location:** `_redeemEth`, `_redeemWstEth`

**Description**  
The comment states:
```solidity
// % reward (3 decimals: 100 = 1%)
```

But the calculation uses a **basis-point divisor**:
```solidity
(balance * redeemFee) / 1e4;
```

**Impact**  
- Fees may be **10× higher or lower** than intended  
- Economic behavior is inconsistent and misleading

**Recommendation**  
Choose one standard:
- **Basis points (1e4):** update comment
- **3 decimals (1e3):** update divisor

---

## High Severity

### H-01: Missing Bounds Validation on IDs
**Affected Functions**
- `createGem`
- `updateCollection`
- `updateGem`
- `mint`

**Description**  
No validation ensures `_id` or `_collection` indices exist.

**Impact**  
- Out-of-bounds access causes reverts  
- Owner can accidentally create invalid Gem → Collection links

**Recommendation**  
Add explicit index checks with clear error messages.

---

### H-02: Incorrect `IwstETH.balanceOf` Interface
**Description**
```solidity
function balanceOf(address account) external returns (uint256);
```

Should be:
```solidity
function balanceOf(address account) external view returns (uint256);
```

**Impact**  
- Interface mismatch  
- Tooling and static analysis assumptions break

**Recommendation**  
Mark `balanceOf` as `view`.

---

### H-03: OpenZeppelin Version Assumptions
**Description**  
The contract uses OZ v5 features:
- `Ownable(msg.sender)`
- `_checkAuthorized(...)`

If OZ v4 is imported, this will **fail to compile or behave incorrectly**.

**Impact**  
- Deployment or runtime breakage

**Recommendation**  
Pin OpenZeppelin version explicitly and ensure API compatibility.

---

## Medium Severity / Design Issues

### M-01: Fee Withdrawal Updates State After External Call
**Location:** `withdrawFees`

**Description**
```solidity
payout.call{value: fees}("");
fees = 0;
```

**Impact**  
- Violates Checks-Effects-Interactions pattern  
- Risk increases if ownership changes to a contract

**Recommendation**
```solidity
uint256 amount = fees;
fees = 0;
payout.call{value: amount}("");
```

---

### M-02: Validator Can Brick Minting
**Description**  
A collection can be created with:
```solidity
validator == true && validatorAddress == address(0)
```

**Impact**  
- Minting becomes permanently impossible

**Recommendation**  
Require non-zero validator address when validator mode is enabled.

---

### M-03: `ERC721URIStorage` Is Effectively Unused
**Description**  
`tokenURI()` is fully overridden, ignoring stored URIs.

**Impact**  
- Extra inheritance cost with no benefit  
- Confusing behavior for integrators

**Recommendation**
- Remove `ERC721URIStorage`, or  
- Respect stored token URIs

---

### M-04: Redundant `onERC721Received` Override
**Description**  
Contract inherits `ERC721Holder` but reimplements the receiver hook.

**Impact**  
- No direct vulnerability  
- Unnecessary complexity

**Recommendation**  
Remove the manual override and rely on `ERC721Holder`.

---

### M-05: ETH Transfer Uses Non-Empty Data and Empty Error
**Location:** `_redeemEth`

```solidity
msg.sender.call{value: amount}(" ");
require(success, " ");
```

**Issues**
- Data payload should be empty
- Error message is uninformative

**Recommendation**
```solidity
msg.sender.call{value: amount}("");
require(success, "ETH transfer failed");
```

---

## Low Severity / Hygiene

- `Pausable` inherited but no `pause()` / `unpause()` functions exposed
- `approveWstEth()` grants infinite allowance (acceptable but risky)
- `circulatingGems` could be `public` for transparency

---

## Key Recommendations (Priority Order)

1. **Fix wstETH minting logic**
2. **Separate ETH and wstETH fee accounting**
3. **Correct redeem fee scaling**
4. **Add bounds validation for IDs**
5. **Ensure OpenZeppelin version consistency**
6. **Clean up redundant logic and improve CEI compliance**

---

### Final Note
From an audit perspective, the **wstETH flow and fee accounting issues alone justify a Critical severity rating**. These must be resolved before mainnet deployment.

