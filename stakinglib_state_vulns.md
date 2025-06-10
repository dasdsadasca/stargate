# Advanced Analysis of StakingLib.sol State Changes

This document analyzes the state change mechanisms within `StakingLib.sol`, specifically focusing on the integrity of `oldBal` (user's old balance) and `oldSupply` (pool's old total supply) as they are read and passed to the `rewarder.onUpdate` function.

## Contracts Involved:

*   `stake/StakingLib.sol` (SLib)
*   `rewarder/StargateStaking.sol` (SS) - The primary caller of SLib functions.
*   `rewarder/StargateMultiRewarder.sol` (SMR) - The typical `IRewarder` implementation whose `onUpdate` is called by SLib.

## Core Logic in `StakingLib.sol` for `deposit`, `withdraw`, `claim`

**1. `deposit(StakingPool storage self, IERC20 token, address from, address to, uint256 amount)`**
   *   `oldBal = self.balanceOf[to];` (User's balance before this deposit)
   *   `oldSupply = self.totalSupply;` (Total staked supply before this deposit)
   *   `newBal = oldBal + amount;`
   *   `self.balanceOf[to] = newBal;` (State update)
   *   `self.totalSupply = oldSupply + amount;` (State update)
   *   `self.rewarder.onUpdate(token, to, oldBal, oldSupply, newBal);` (External call with old values)
   *   `token.safeTransferFrom(from, address(this), amount);`

**2. `withdraw(StakingPool storage self, IERC20 token, address from, address to, uint256 amount, bool withUpdate)`**
   *   `oldBal = self.balanceOf[from];` (User's balance before this withdrawal)
   *   `oldSupply = self.totalSupply;` (Total staked supply before this withdrawal)
   *   `if (oldBal < amount) revert WithdrawalAmountExceedsBalance();`
   *   `newBal = oldBal - amount;`
   *   `self.balanceOf[from] = newBal;` (State update)
   *   `self.totalSupply = oldSupply - amount;` (State update)
   *   `if (withUpdate)`:
        *   `self.rewarder.onUpdate(token, from, oldBal, oldSupply, newBal);` (External call with old values)
   *   `token.safeTransfer(to, amount);`

**3. `claim(StakingPool storage self, IERC20 token, address user)`**
   *   This function is intended to trigger reward updates/payouts without changing stake.
   *   It calls `self.rewarder.onUpdate(token, user, self.balanceOf[user], self.totalSupply, 0);`
     *   `self.balanceOf[user]` is the current balance of the user.
     *   `self.totalSupply` is the current total supply.
     *   The `newStake` parameter for `onUpdate` is passed as `0`. This is a bit unusual. Typically, `onUpdate` might expect the *actual* new stake. If SMR's `onUpdate` uses the `newStake` parameter to calculate anything or update its internal representation of the user's stake, passing `0` here might be problematic if not specifically handled by the rewarder.
     *   Looking at SMR's `onUpdate`: `onUpdate(IERC20 stakingToken, address user, uint256 oldStake, uint256 oldSupply, uint256 /*newStake*/ )`. The `newStake` parameter is commented out (`/*newStake*/`), meaning SMR's current implementation **does not use the `newStake` value passed to it.** It only cares about `oldStake` and `oldSupply` to calculate rewards accrued up to that point. This makes passing `0` as `newStake` from `StakingLib.claim` benign for SMR.

## Analysis of `oldBal` and `oldSupply` Integrity

1.  **Source of Truth:**
    *   `oldBal` is read directly from `self.balanceOf[user]` (storage) before any modifications in the current call.
    *   `oldSupply` is read directly from `self.totalSupply` (storage) before any modifications in the current call.
    *   These represent the state of the pool as of the beginning of the current function's execution.

2.  **Manipulation Potential by Non-Privileged Users:**
    *   A user cannot directly set `self.balanceOf[user]` or `self.totalSupply` to arbitrary values. These are updated only through `deposit` and `withdraw` logic.
    *   When a user calls `SS.deposit()` or `SS.withdraw()`, which then calls the respective SLib functions, the `oldBal` and `oldSupply` are read *before* the effects of their current transaction (the amount being deposited/withdrawn) are applied to these storage variables.
    *   This is the correct behavior: rewards should be calculated based on the stake held *during* the reward accrual period, which ends when the user's stake changes. The `oldBal` and `oldSupply` correctly capture this pre-change state.

3.  **Sequence of Actions Leading to Exploitable `onUpdate` Calls:**

    *   **Single Transaction Context:** All operations within a single SLib function call (`deposit` or `withdraw`) occur atomically. An attacker cannot interleave actions *between* the reading of `oldBal`/`oldSupply` and the call to `rewarder.onUpdate` within that same top-level transaction.
    *   **Contract Calling SLib Functions Multiple Times (e.g., deposit then withdraw in one tx):**
        *   Suppose a contract `AttackerContract` calls `SS.deposit(...)` and then immediately `SS.withdraw(...)` for the same user and token within its own single transaction.
        *   **First call (deposit):**
            *   `oldBal_1`, `oldSupply_1` are read from storage.
            *   SLib updates `balanceOf`, `totalSupply`.
            *   `rewarder.onUpdate(token, user, oldBal_1, oldSupply_1, newBal_1)` is called.
            *   SMR calculates rewards based on `oldBal_1`, `oldSupply_1`.
        *   **Second call (withdraw):**
            *   `oldBal_2` is read from storage. This will be `newBal_1` from the deposit call (user's balance *after* the deposit).
            *   `oldSupply_2` is read from storage. This will be `totalSupply` *after* the deposit.
            *   SLib updates `balanceOf`, `totalSupply` again (for the withdrawal).
            *   `rewarder.onUpdate(token, user, oldBal_2, oldSupply_2, newBal_2)` is called.
            *   SMR calculates rewards based on `oldBal_2`, `oldSupply_2`.
        *   **Integrity:** This sequence is correct. The second call to `onUpdate` (for withdrawal) uses the state *after* the deposit completed and *before* the withdrawal is processed. This accurately reflects the (potentially very short) period the user held the `oldBal_2` amount.
        *   **No Exploitation Seen:** The values `oldBal` and `oldSupply` passed to `onUpdate` accurately reflect the state of the user's holdings and the total pool supply at the beginning of each discrete operation (`deposit` or `withdraw`). An attacker cannot make these values "stale" or "incorrect" for the period they represent. The atomicity of each `deposit`/`withdraw` call ensures this.

4.  **`claim()` Function Specifics:**
    *   `StakingLib.claim` calls `rewarder.onUpdate(token, user, currentBalance, currentSupply, 0)`.
    *   As noted, SMR's `onUpdate` doesn't use the `newStake` (fifth) parameter.
    *   The `currentBalance` (`self.balanceOf[user]`) and `currentSupply` (`self.totalSupply`) passed are the live values.
    *   SMR's `onUpdate` uses the passed `oldStake` (which is `currentBalance` here) and `oldSupply` (which is `currentSupply` here) to calculate rewards.
    *   This means `claim` effectively calculates rewards accrued up to the present moment based on the user's current stake and the pool's current total supply. This is the intended behavior for a claim function that doesn't alter stake.

5.  **Race Conditions / Atomicity Issues (Single Transaction Context):**
    *   Within a single transaction, Solidity execution is sequential. There are no race conditions in the traditional multi-threading sense.
    *   The relevant "atomicity" is that each call to `deposit` or `withdraw` from `StargateStaking` completes its storage reads, state updates, and the `onUpdate` call before the next external call from a user (or contract) can begin.
    *   If `rewarder.onUpdate` were to re-enter `StakingLib` (which is protected by `StargateStaking`'s `nonReentrant` guard for main entry points), this would be a re-entrancy issue, not a race condition. The `nonReentrant` guard in `StargateStaking` should prevent SLib functions from being re-entered in a way that corrupts these `oldBal`/`oldSupply` readings for a single operation.
    *   For example, if `rewarder.onUpdate` called back into `SS.deposit()`:
        *   The outer `SS.deposit()` call would be in progress, its `nonReentrant` lock active.
        *   The inner, re-entrant `SS.deposit()` call would be blocked by the `nonReentrant` guard.
        *   Thus, the `oldBal` and `oldSupply` for the outer call remain consistent for its `onUpdate` invocation.

## Conclusion on `StakingLib.sol` State Variable Integrity

*   The `oldBal` and `oldSupply` variables in `StakingLib.deposit` and `StakingLib.withdraw` are read from storage *before* any modifications from the current call are applied. This is correct and ensures that the `rewarder.onUpdate` function receives the state relevant to the period ending with the user's action.
*   Sequences of actions within a single transaction (e.g., a contract calling deposit then withdraw) will result in multiple calls to `onUpdate`, each with `oldBal` and `oldSupply` values that correctly reflect the state at the beginning of that specific sub-operation. This does not appear to create an exploit vector.
*   The `claim` function passes current balance and supply to `onUpdate` as `oldStake` and `oldSupply`, which is appropriate for calculating rewards up to that point without a stake change. SMR's non-use of the `newStake` parameter makes the `0` value passed by `claim` benign.
*   Atomicity within each function call and the `nonReentrant` guard in the calling `StargateStaking` contract protect the integrity of these values against same-transaction re-entrancy exploits that might aim to manipulate them between reading and usage.

No vulnerabilities related to the determination or usage of `oldBal` and `oldSupply` within `StakingLib.sol` itself were identified. Their integrity relies on correct storage reads and the atomicity of function executions, which seem to be properly handled. The key external dependency is the behavior of the `rewarder.onUpdate` function, but the inputs it receives from `StakingLib` are accurate reflections of the pre-action state.I have performed the advanced analysis of `StakingLib.sol` state changes, focusing on the integrity and potential manipulation of `oldBal` and `oldSupply` values passed to `rewarder.onUpdate`. The findings have been documented in `stakinglib_state_vulns.md`.

The analysis covered:
1.  Integrity and potential manipulation of `oldBal` and `oldSupply`.
2.  Sequences of actions (e.g., deposit then withdraw within a contract call) and their effect on values passed to `onUpdate`.
3.  Potential race conditions or atomicity issues within single transaction execution.

The conclusion is that `StakingLib.sol` correctly determines and uses `oldBal` and `oldSupply`, reflecting the state accurately before modifications from the current call. These values do not seem manipulatable by users to exploit reward calculations in `StargateMultiRewarder.onUpdate`, given SMR uses these as the basis for the period ending with the user's action. The `nonReentrant` guard in `StargateStaking.sol` further protects the integrity of these operations.

The subtask is complete. I will now submit the report.
