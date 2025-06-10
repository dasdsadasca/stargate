# Vulnerability Analysis: Cross-Contract Interactions

This document outlines potential vulnerabilities arising from interactions between `StargateStaking.sol` and `StargateMultiRewarder.sol`, including their respective libraries.

## Methodology

The analysis focuses on specific interaction points: `onUpdate` calls, calls to `IStargateStaking` views, administrative desynchronization, `withdrawToAndCall` flow, and potential manipulation of shared state like `totalSupply`.

## Contracts Involved:

*   `StargateStaking.sol` (SS)
*   `StakingLib.sol` (SLib) - Used by SS
*   `StargateMultiRewarder.sol` (SMR)
*   `RewardLib.sol` (RLib) - Used by SMR
*   `RewardRegistryLib.sol` (RRLib) - Used by SMR

## Analysis of Cross-Contract Scenarios:

### 1. `StargateStaking` -> `StargateMultiRewarder.onUpdate` Call Sequence

*   **Context:**
    *   SS's `deposit()`, `withdraw()`, and `claim()` functions call `SLib.deposit()`, `SLib.withdraw()`, and `SLib.claim()` respectively.
    *   These SLib functions then call `SMR.onUpdate(token, user, oldStake, oldSupply, newStake/0)`.
    *   SS functions (`deposit`, `withdraw`, `claim`) are protected by a `nonReentrant` guard.
    *   SMR's `onUpdate` function is **not** protected by a reentrancy guard.
    *   SMR's `onUpdate` first calculates rewards for all relevant pools (calling `RLib.indexAndUpdate` which updates `rewardDebt` and `lastRewardTime`) and then iterates again to transfer these rewards via `SMR._transferToken()`.

*   **Re-entrancy from `SMR._transferToken` back into `SMR.onUpdate` or other SMR functions:**
    *   If `SMR._transferToken` (specifically the native ETH transfer `to.call{value: amount}("")`) allows the recipient `user` to call back into SMR:
        *   **Target: `SMR.onUpdate()` again:**
            *   State of SS: Still locked by its `nonReentrant` guard from the initial call. A re-entrant call from SMR back to SS's `deposit/withdraw/claim` would be blocked.
            *   State of SMR:
                *   For the *outer* `onUpdate` call: The first loop (reward calculation and `rewardDebt` update via `RLib.indexAndUpdate`) for all reward pools associated with the `stakingToken` would have completed.
                *   The re-entrant call to `onUpdate` would execute with the same `stakingToken`, `user`, `oldStake`, `oldSupply`, `newStake` parameters passed by the original SS call.
                *   `RLib.indexAndUpdate` would be called again. `RLib.update` calculates `rewardsForUser = ((accRewardPerShare - pool.rewardDebt[user]) * oldStake) / PRECISION;`. Since `pool.rewardDebt[user]` was already updated to the current `accRewardPerShare` in the outer call's first loop, the `rewardsForUser` in the re-entrant call should be zero (or very close to zero, accounting for potential minimal time change if `accRewardPerShare` is re-indexed).
                *   **Impact Assessment:** Direct double-claiming of rewards for the *same triggering event* seems unlikely due to `rewardDebt` being updated before transfers. However, re-entrancy could still be problematic:
                    *   **Gas Griefing:** A re-entrant call could consume remaining gas, causing the outer transaction to fail.
                    *   **Integrity of other SMR functions:** If the re-entrant call targets other SMR admin functions (if they weren't `onlyOwner`) or view functions, it might observe inconsistent states, though direct state corruption is less obvious if those functions are also robust.
                    *   **Complexity:** Re-entrancy makes reasoning about contract state significantly harder.
        *   **Target: Other SMR state-changing functions (e.g., admin functions if not `onlyOwner`):** This is less likely as most are `onlyOwner`.
    *   **Recommendation:** It is highly recommended to add a `nonReentrant` guard to `SMR.onUpdate()` to prevent these re-entrancy scenarios, ensuring cleaner state management and defense-in-depth.

### 2. Calls from `StargateMultiRewarder` to `IStargateStaking` View Functions

*   **Context:**
    *   `SMR.onUpdate()` calls `RLib.indexAndUpdate()`.
    *   `RLib.index()` and `RLib._index()` are called, which then may (indirectly, though not explicitly shown in the provided `RewardLib` snippets for `onUpdate` itself but rather in `getRewards`) use `staking.balanceOf(pool.stakingToken, user)` and `staking.totalSupply(pool.stakingToken)`. The `onUpdate` in SMR passes `oldStake` and `oldSupply` directly, but `_indexRewardTokenPools` (called by admin functions) does call `staking.totalSupply()`. The `getRewards` view function in SMR also calls these SS views.
*   **DoS Risks:**
    *   The `staking` address is `immutable` in SMR, set at construction. This means it cannot be changed to a malicious contract by SMR's owner.
    *   If the `StargateStaking` contract instance at that address were to be self-destructed (if it had such a function and was not properly secured) or if these view functions could be made to revert (e.g., due to an internal error or extreme out-of-gas condition within SS itself), then:
        *   Calls from SMR requiring these values (e.g., `_indexRewardTokenPools` during admin operations, or `getRewards` view) would fail.
        *   This could disrupt reward calculations or prevent admins from properly updating reward parameters if those updates rely on fresh `totalSupply` readings.
    *   **Impact Assessment:** Low to Medium. The `StargateStaking` contract is central; its failure would have wider implications. The specific risk here is SMR's functions failing due to SS view issues. `onUpdate` itself uses passed-in values for supply/stake, mitigating this for core reward distribution.
    *   **Recommendation:** This is an inherent dependency. Ensure `StargateStaking` view functions are robust and low-cost.

### 3. Admin Functions and Potential for Desynchronization

*   **Context:** `StargateStaking` (SS) and `StargateMultiRewarder` (SMR) both have owners. It's possible they are different.
*   **Scenarios & Risks:**
    *   **SS Owner Actions:**
        *   If SS owner calls `setPool(lp_token, new_rewarder_address)`:
            *   This calls `new_rewarder_address.connect(lp_token)`. If `new_rewarder_address` is an SMR instance, SMR's `connect()` is called, marking the `lp_token` as connected. This is fine.
            *   If SS owner sets a rewarder that is *not* the intended SMR, or an SMR with a misconfiguration, SS will call this incorrect rewarder.
    *   **SMR Owner Actions:**
        *   SMR owner configures rewards (`setReward`, `setAllocPoints`) for specific `rewardToken` and `stakingToken` (LP token) pairs.
        *   If SMR owner configures rewards for an LP token that is not actually managed by or correctly wired up in SS (i.e., SS does not have a pool for that LP token, or SS's pool for that LP token points to a different rewarder), then:
            *   SMR will track rewards for this LP token.
            *   However, `onUpdate` calls will never come from SS for this specific LP token if SS doesn't know about it or uses a different rewarder. Rewards would accrue internally in SMR's view but might not be claimable if claims are only triggered via SS's `onUpdate`.
            *   The `claim()` function in SS iterates `lpTokens` provided by the user. If a user tries to claim for an LP token that SS knows but SMR doesn't (or vice-versa due to desync), it might lead to partial failures or no rewards for that token.
    *   **Impact Assessment:** Medium. Desynchronization can lead to rewards not accruing correctly, users being unable to claim, or calls going to incorrect contracts.
    *   **Recommendation:** Ideally, the ownership of SS and SMR should be closely managed, possibly by the same entity or a multi-sig with clear operational procedures. Strong documentation and off-chain monitoring are needed if owners are different. Consider events to signal important reconfigurations that the other contract's admin might need to react to.

### 4. `StargateStaking.withdrawToAndCall` Flow

*   **Context:**
    1.  User calls `SS.withdrawToAndCall(token, receiver_contract, amount, data)`.
    2.  SS (via SLib) calls `SMR.onUpdate(token, user, oldBal, oldSupply, newBal)` to process rewards.
    3.  SS (via SLib) transfers LP tokens to `receiver_contract`.
    4.  SS calls `receiver_contract.onWithdrawReceived(token, user, amount, data)`.
*   **Risk: `IStakingReceiver.onWithdrawReceived` calling into `StargateMultiRewarder`:**
    *   The `receiver_contract` is an arbitrary contract. It could attempt to call functions on SMR.
    *   **Target: `SMR.onUpdate()`:** SMR's `onUpdate` is `onlyStaking`. The `receiver_contract` cannot call this directly.
    *   **Target: `SMR.getRewards()` (view function):** This is fine, just a view.
    *   **Target: SMR Admin Functions:** These are `onlyOwner`, so `receiver_contract` cannot call them.
    *   **Target: Other hypothetical SMR functions:** If SMR had other unprotected state-changing functions, this could be a vector. Currently, it doesn't seem to have such functions that would be impactful here.
    *   Since `SMR.onUpdate` (which handles reward payout) has already been called and completed *before* `onWithdrawReceived` is invoked, the user's rewards for the withdrawal event are already processed. A call from `onWithdrawReceived` to SMR functions related to reward calculation for *that specific withdrawal event* would likely see the state as already updated.
    *   **Impact Assessment:** Low. The `onlyStaking` guard on `onUpdate` and `onlyOwner` on admin functions in SMR significantly limit what `receiver_contract` can do to SMR. The sequence of operations (rewards processed before external call to receiver) is generally good.
    *   **Recommendation:** Current design seems relatively robust for this specific flow due to SMR's access controls.

### 5. Manipulation of `staking.totalSupply()` for Reward Calculation

*   **Context:** Some SMR calculations, particularly during admin configuration (`_indexRewardTokenPools` called by `setReward`, `setAllocPoints`) or in `getRewards` view, use `staking.totalSupply(pool.stakingToken)`. The core `onUpdate` flow uses `oldSupply` passed by SS.
*   **Scenario:**
    *   An attacker wants to manipulate `accRewardPerShare` which depends on `totalSupply`.
    *   The `onUpdate` function, which is critical for actual reward payouts, uses `oldSupply` passed by `StargateStaking`. This `oldSupply` is read by SS *before* the user's current deposit/withdrawal action affects SS's `totalSupply`. So, a flash loan manipulating `totalSupply` in SS *during* a deposit/withdrawal that triggers `onUpdate` would not directly affect the `oldSupply` parameter used in that `onUpdate` call for that user's rewards.
    *   However, if an attacker could:
        1.  Manipulate `SS.totalSupply()` (e.g., flash loan deposit of many LP tokens).
        2.  *Then*, in the same transaction, trigger an SMR admin action (e.g., if they were also the owner) like `setReward` or `setAllocPoints` which calls `_indexRewardTokenPools`. This function *would* use the manipulated `staking.totalSupply()`.
        3.  This would affect the `pool.accRewardPerShare` calculation for that specific admin-triggered update.
        4.  Subsequent user reward calculations (via `onUpdate` or `getRewards`) would use this manipulated `accRewardPerShare`.
*   **Impact Assessment:** Medium. This is a complex scenario requiring specific conditions (attacker influences `totalSupply` AND triggers SMR admin functions that re-index based on that `totalSupply` in the same transaction). The primary reward distribution in `onUpdate` is shielded by using `oldSupply`. The main risk is to the global `accRewardPerShare` if admin actions happen concurrently with `totalSupply` manipulation.
*   **Recommendation:**
    *   Admin actions that cause re-indexing of SMR pools (`setReward`, `setAllocPoints`) should be done with caution, ideally not in highly volatile periods or complex transactions.
    *   Using a time-weighted average for `totalSupply` (TWAP) is a common defense if this were a more direct issue, but it's complex to implement. Given `onUpdate` uses `oldSupply`, the current risk is more about the base `accRewardPerShare` being set during admin ops. The owner should ensure `totalSupply` is stable/representative during such maintenance.

## Summary of Cross-Contract Concerns:

1.  **Re-entrancy in `SMR.onUpdate`:** Absence of a re-entrancy guard in `SMR.onUpdate` is a notable risk, even if direct exploitation for double reward claim seems difficult. Adding a guard is recommended.
2.  **Admin Desynchronization:** Different owners or uncoordinated actions between SS and SMR admins can lead to misconfigured reward flows or non-functional pools.
3.  **Integrity of `StargateStaking` Views:** SMR relies on SS view functions. While SS is immutable in SMR, if SS itself had issues, it would impact SMR.
4.  **Potential for `totalSupply` Manipulation to Affect `accRewardPerShare` (during admin re-indexing):** While core user reward flow is protected, admin-triggered re-indexing could use a manipulated `totalSupply`.

Overall, the separation of concerns (SS for staking, SMR for rewards) is logical, but the interaction points, especially `onUpdate` and administrative alignment, require careful consideration and robust handling.
