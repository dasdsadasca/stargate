# Analysis of Cross-Contract State Consistency and Calculation Impacts

This document analyzes potential vulnerabilities arising from state inconsistencies and timing differences between `StargateStaking.sol` (SS) and `StargateMultiRewarder.sol` (SMR), and their impact on reward calculations in `RewardLib.sol` (RLib).

## 1. Timing Difference in State Usage (`oldStake`/`oldSupply` vs. Live Reads)

There's a fundamental and intentional difference in how state from `StargateStaking` is consumed by `StargateMultiRewarder`:

*   **User-Triggered Operations (`SMR.onUpdate`)**:
    *   When a user deposits, withdraws, or claims via `StargateStaking`, SS calls `SMR.onUpdate(stakingToken, user, oldStake, oldSupply, newStake)`.
    *   `oldStake` is the user's balance in SS *before* the current operation.
    *   `oldSupply` is the `totalSupply` of the `stakingToken` in SS *before* the current operation.
    *   `RewardLib` uses these `oldStake` and `oldSupply` values to calculate the rewards accrued by the user up to the point of their current action. This ensures that users are rewarded fairly based on their contribution *prior* to the state change they are initiating. It prevents them from, for example, depositing and instantly claiming rewards on that new deposit within the same transaction based on a `totalSupply` that doesn't yet fully reflect their deposit.
    *   **This is a security measure to ensure fairness for the transacting user.**

*   **Admin-Triggered Re-indexing (`SMR._indexRewardTokenPools`)**:
    *   Functions like `SMR.setReward()` and `SMR.setAllocPoints()` call `_indexRewardTokenPools()`.
    *   This function iterates through relevant pools and calls `pool.index(rewardDetails, staking.totalSupply(pool.stakingToken))`.
    *   Here, `staking.totalSupply(pool.stakingToken)` is a **live call** to SS, fetching the current total supply of the LP token at that exact moment in the transaction.
    *   **This live read is the basis for the flash loan vulnerability detailed below.** It's used to update the global `pool.accRewardPerShare` for *all* participants in that pool.

*   **View Functions (`SMR.getRewards`)**:
    *   `SMR.getRewards(stakingToken, user)` also uses live calls: `staking.balanceOf(pool.stakingToken, user)` and `staking.totalSupply(pool.stakingToken)` to calculate and return the pending rewards for a user *at that moment*.
    *   This is generally acceptable for view functions, as they don't change state, but it means the value can fluctuate based on ongoing activity.

**Impact of Timing Difference:**
The use of `oldStake`/`oldSupply` in `onUpdate` is crucial for preventing users from exploiting their own transactions to gain unfair rewards. However, the use of live `totalSupply` in admin-triggered re-indexing creates an attack surface if that live value can be manipulated within the same transaction as the admin's action.

## 2. Flash Loan Attack Scenario: Theft of Unclaimed Yield via `_indexRewardTokenPools`

This scenario refines the previously identified flash loan attack, framing it as a "Theft of Unclaimed Yield." The attacker aims to artificially inflate `pool.accRewardPerShare` for a targeted pool, allowing them to claim a disproportionate share of the total reward budget before it's rightfully distributed to long-term stakers over time.

*   **Attacker's Goal**: Maximize `rewardsForUser = ((inflated_accRewardPerShare - attacker_rewardDebt) * attacker_stake) / PRECISION` by manipulating `inflated_accRewardPerShare`.

*   **Preconditions**:
    1.  SMR Admin is about to call a function (e.g., `setReward`, `setAllocPoints`) that triggers `_indexRewardTokenPools` for a specific `rewardToken`.
    2.  This `_indexRewardTokenPools` will update `pool.accRewardPerShare` for one or more staking pools (`pool`) using a live `staking.totalSupply(pool.stakingToken)` call.
    3.  The attacker can front-run this admin transaction.
    4.  The attacker has access to significant capital in the form of the target `pool.stakingToken` (LP token), or can flash-loan other assets to acquire it temporarily if needed for complex balance sheet maneuvers.

*   **Attack Steps**:
    1.  **Preparation**: Attacker may already have some stake in the target pool, or is prepared to stake. Their current `rewardDebt` for this pool is based on the legitimate, lower `accRewardPerShare`.
    2.  **Front-Run & `totalSupply` Deflation**:
        *   Attacker monitors the mempool for the admin's transaction.
        *   Before the admin's transaction executes, the attacker:
            *   Withdraws a very large amount of `pool.stakingToken` from `StargateStaking.sol`. This drastically *reduces* the live `staking.totalSupply(pool.stakingToken)`.
            *   (This step assumes the attacker has a large legitimate stake to withdraw. If using a flash loan to borrow the LP tokens *to* withdraw, it's more complex as they'd need to already have them staked or find a way to borrow them from a source other than the Stargate pool itself if the goal is to reduce Stargate's `totalSupply`).
    3.  **Admin Transaction Execution**:
        *   The SMR admin's transaction calls `_indexRewardTokenPools`.
        *   For the targeted pool, `RLib._index` is called. The `totalSupply` parameter it receives is now the artificially *low* value due to the attacker's withdrawal.
        *   The calculation `accRewardPerShare_increment = (rewardDetails.rewardPerSec * (end - start) * pool.allocPoints * PRECISION) / rewardDetails.totalAllocPoints / deflated_totalSupply` results in a much *larger* `accRewardPerShare_increment`.
        *   The pool's global `pool.accRewardPerShare` is updated to this new, artificially inflated value.
    4.  **Exploitation - Claiming Excessive Rewards**:
        *   In a subsequent transaction (ideally in the same block, immediately after the admin's tx), the attacker:
            *   Deposits a chosen amount of `pool.stakingToken` back into `StargateStaking.sol` (or uses their remaining stake if they didn't withdraw all of it).
            *   This deposit triggers `SMR.onUpdate`.
            *   `RLib.update` calculates their rewards: `rewardsForUser = ((inflated_accRewardPerShare - attacker_previous_rewardDebt) * attacker_stake) / PRECISION;`.
            *   Since `inflated_accRewardPerShare` is now much higher than their `attacker_previous_rewardDebt` (which was based on the pre-attack `accRewardPerShare`), they receive a significantly larger amount of reward tokens than they are legitimately entitled to for their actual contribution over time.
        *   This is effectively a theft of yield that was meant to be distributed over a longer period or to other honest stakers.
    5.  **Cleanup**: Attacker restores their LP token balances if necessary (e.g., repays any operational flash loans if used for parts of the capital movement, though the primary capital is their own large stake of the LP token).

*   **Impact**:
    *   **Theft of Unclaimed Yield**: The attacker unjustly claims a larger portion of the available reward tokens from the `StargateMultiRewarder` contract balance for that specific reward token. This depletes the funds available for future distribution to honest stakers.
    *   **Distortion of Pool State**: The `pool.accRewardPerShare` remains artificially high, potentially causing overpayment to subsequent users until the value is naturally diluted or another admin action corrects it (which might involve using a normal `totalSupply`).
*   **Severity**: High (Potential for significant value extraction from the reward pool).
*   **Recommendation Refinement**:
    *   **Primary:** The most direct mitigation is for SMR admin functions that call `_indexRewardTokenPools` to *not* use a live `staking.totalSupply()`. Instead, `StargateMultiRewarder` could maintain its own view or snapshot of `totalSupply` for each staking token, updated only during `SMR.onUpdate` (which uses the safe `oldSupply`). Admin functions would then use this internal, more stable `totalSupply` view. This breaks the direct dependency on live `totalSupply` during admin ops.
    *   **Secondary (Operational):** If the above is not feasible, extreme caution during admin calls, avoiding volatile periods, and potentially announcing admin actions to discourage front-running. TWAP for `totalSupply` is also a theoretical but complex alternative.

## 3. Other Timing/State Inconsistencies Exploitable by Non-Owners

Excluding the flash loan scenario targeting admin actions:

*   **`SMR.onUpdate` as the Synchronizer**: The `SMR.onUpdate` function, callable only by `StargateStaking`, is the primary mechanism for synchronizing a user's state (their stake changes) between SS and SMR. When `onUpdate` is called with `oldStake` and `oldSupply`, it calculates rewards based on that past state and then SMR becomes aware of the user's action (implicitly, as their rewards are paid out based on that change).
*   **Direct Calls to SMR by Non-Owners**: Non-owners cannot call `SMR.onUpdate` or SMR admin functions. They can call `SMR.getRewards()` (a view function) and `SMR.connect()` (but `connect` is also `onlyStaking`).
*   **Exploiting `getRewards` View?**:
    *   `getRewards` uses live `staking.balanceOf()` and `staking.totalSupply()`.
    *   A user could deposit into SS. `SMR.onUpdate` runs, paying out rewards based on `oldStake`.
    *   If the user then immediately calls `SMR.getRewards()`, it will show pending rewards based on their *new* stake and current `totalSupply`. This is not an exploit but expected behavior of the view function – it shows what would be claimable *if* an update happened now. The actual payout only happens when `onUpdate` is triggered by a state-changing SS action.
*   **Rapid Deposit/Withdrawal Cycles**:
    *   Consider an attacker repeatedly depositing and withdrawing small amounts from SS very quickly. Each action triggers `SMR.onUpdate`.
    *   Rewards are calculated based on `oldStake`. `rewardDebt` is updated.
    *   Due to precision losses (dust), the attacker might lose tiny fractions of rewards on each cycle. This is not beneficial to them.
    *   There's no obvious way to cause `rewardDebt` to become misaligned with `accRewardPerShare` to their advantage through such rapid cycles, as `accRewardPerShare` is cumulative and `rewardDebt` tracks it per user.
*   **No `StakingLib.claim()` Race Condition**:
    *   `StakingLib.claim()` calls `self.rewarder.onUpdate(token, user, self.balanceOf[user], self.totalSupply, 0)`. The `0` for `newBal` effectively signals that this is a "claim what's due without changing stake" type of operation, or it processes rewards before a balance becomes zero (like in `emergencyWithdraw` if `withUpdate` was true, though it's false there). The `balanceOf[user]` and `totalSupply` are current state from `StakingLib`'s perspective.
    *   Since `StakingLib.claim` is called by `SS.claim`, which is `nonReentrant`, and then calls `SMR.onUpdate`, the state consistency for that claim operation itself is maintained within that transaction boundary.

**Conclusion on Other Inconsistencies for Non-Owners**:
Barring the specific flash loan attack on admin functions, the design where `SMR.onUpdate` uses `oldStake` and `oldSupply` seems to effectively prevent non-owners from exploiting timing or state inconsistencies between SS and SMR for their direct staking, withdrawal, or claiming operations to gain undue rewards. The main vector for state-related exploits lies in manipulating inputs to admin-triggered global updates.

This analysis reinforces the criticality of the flash loan vector against admin functions that use live `totalSupply` and suggests that other user-level exploits based purely on state timing between the two main contracts are less likely due to the design of `onUpdate`.I have completed the analysis focusing on cross-contract state consistency, timing differences, and refined the flash loan attack scenario, documenting these findings in the `cross_contract_state_vulns.md` content.

I will now use the `create_file_with_block` tool to save this content.
