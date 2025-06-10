# Analysis of Re-entrancy Scenarios in StargateMultiRewarder.onUpdate

This document analyzes the potential for non-owners to exploit re-entrancy in the `StargateMultiRewarder.onUpdate` function, focusing on state changes, potential for theft, state corruption, Denial-of-Service (DoS), and interactions with other functions.

## 1. Modeling State Changes During a Re-entrant Call to `onUpdate`

The `StargateMultiRewarder.onUpdate` function is called by `StargateStaking.sol` (SS) when a user deposits, withdraws, or claims. SS itself is protected by a `nonReentrant` guard during these operations. `onUpdate` has an `onlyStaking` modifier, meaning `msg.sender` must be the `StargateStaking` contract.

**Structure of `onUpdate`:**
1.  **Loop 1 (Calculations):** For each reward pool associated with the input `stakingToken`:
    *   `pool.indexAndUpdate(...)` is called. This internally calls `RewardLib.index()` then `RewardLib.update()`.
    *   `RewardLib.index()`: Calculates `new_accRewardPerShare` based on current state, `block.timestamp`, and `totalSupply` (passed as `oldSupply`). Updates `pool.accRewardPerShare` and `pool.lastRewardTime` in storage.
    *   `RewardLib.update()`: Calculates `rewardsForUser` using `new_accRewardPerShare` and the *current* `pool.rewardDebt[user]`. It then updates `pool.rewardDebt[user] = new_accRewardPerShare` in storage.
    *   The calculated `rewardsForUser` is stored in a local `amounts` array.
2.  An event `RewardsClaimed` is emitted with the `amounts` array.
3.  **Loop 2 (Transfers):** For each reward pool, if `amounts[i] > 0`, `_transferToken(user, rewardToken, amounts[i])` is called.
    *   If `rewardToken` is native ETH, `_transferToken` uses `user.call{value: amount}("")`. This is the potential re-entrancy vector if `user` is a contract.

**Hypothetical Re-entrancy Path:**
A direct re-entrant call from the `user` contract (receiving ETH) back to `SMR.onUpdate` would fail the `onlyStaking` modifier.
Therefore, a re-entrancy exploit would require a more complex path:
`SS.deposit/withdraw` (User action) -> `SMR.onUpdate` (Outer call) -> `SMR._transferToken` -> `user.receive()` -> `SS.someVulnerableFunction()` (if SS allows re-entry and can be made to call `SMR.onUpdate` again) -> `SMR.onUpdate` (Inner call).

**State Changes Across Re-entrant Call (assuming the complex path above):**

Let `ARPS` be `accRewardPerShare` and `RD` be `rewardDebt`.

*   **Outer `onUpdate` - Loop 1 (Calculations for pool P):**
    1.  `RLib.index` calculates `ARPS_P_outer_new`. Storage: `P.accRewardPerShare = ARPS_P_outer_new`, `P.lastRewardTime = ts_outer`.
    2.  `RLib.update` calculates `rewards_P_outer` using `ARPS_P_outer_new` and `P.rewardDebt[user]_before_outer`. Storage: `P.rewardDebt[user] = ARPS_P_outer_new`.
    3.  `local_amounts_array` stores `rewards_P_outer`.

*   **Outer `onUpdate` - Loop 2 (Transfers for pool P, e.g., ETH):**
    4.  `_transferToken(user, ETH, rewards_P_outer)` initiates transfer. Re-entrancy occurs here.

*   **Inner `onUpdate` - Loop 1 (Calculations for pool P):**
    5.  `RLib.index` calculates `ARPS_P_inner_new`. This will be `ARPS_P_outer_new + increment_due_to_time_delta` (increment is likely zero or tiny if in same block). Storage: `P.accRewardPerShare = ARPS_P_inner_new`, `P.lastRewardTime = ts_inner`.
    6.  `RLib.update` calculates `rewards_P_inner` using `ARPS_P_inner_new` and `P.rewardDebt[user]` (which is currently `ARPS_P_outer_new` from step 2).
        *   `rewards_P_inner = ((ARPS_P_inner_new - ARPS_P_outer_new) * oldStake) / PRECISION`. This calculates only the marginal reward accrued during the re-entrancy.
    7.  Storage: `P.rewardDebt[user] = ARPS_P_inner_new`.
    8.  `local_amounts_array_inner` stores `rewards_P_inner`.

*   **Inner `onUpdate` - Loop 2 (Transfers):**
    9.  Transfers `rewards_P_inner` (the marginal amount).

*   **Outer `onUpdate` - Loop 2 (Resumes):**
    10. The transfer of `rewards_P_outer` (from step 4) completes.
    11. If there are other reward tokens, their transfers proceed using the amounts calculated in the initial Loop 1 of the outer call. The `rewardDebt` for those other tokens would have also been updated by the inner call's Loop 1 if they were processed there.

## 2. Impact Assessment of Re-entrancy

### 2.1. Theft of Unclaimed Yield

*   **Direct Double Claim:** The model above shows that a re-entrant call to `onUpdate` (even if possible) would likely not result in the user claiming the main reward amount twice for the same triggering event. The `rewardDebt` is updated after the first calculation, so the inner call calculates rewards only on the marginal difference in `accRewardPerShare` accrued during the very short period of the re-entrancy.
*   **State Manipulation for Future Gain:** The critical state variables (`pool.accRewardPerShare`, `pool.rewardDebt[user]`, `pool.lastRewardTime`) are updated sequentially. The inner call updates them based on the state left by the outer call's calculation phase. While this interleaving is complex, it doesn't present an obvious path to setting `rewardDebt` to an artificially low value or `accRewardPerShare` to an unjustly high value *for the attacker's direct benefit* within this re-entrant flow. The `amounts` array for the outer call is fixed before re-entrancy begins.

**Conclusion on Theft**: Theft of unclaimed yield via this specific re-entrancy mechanism appears unlikely.

### 2.2. Corruption of State Affecting Other Users

*   The global state `pool.accRewardPerShare` and `pool.lastRewardTime` are updated by both outer and inner calls. The inner call's update is based on the outer call's updated state. This means these variables are moved forward in time/accumulation, which is not inherently corrupting for other users. Other users' rewards will be calculated based on these latest values.
*   Individual `pool.rewardDebt[user]` is specific to the user involved in the transaction.
*   The risk of broad state corruption affecting *other* users seems low from this vector alone, as the changes are consistent with how rewards accrue over time, albeit with potentially very small time increments during re-entrancy.

**Conclusion on State Corruption**: Significant harmful state corruption affecting other users is not an obvious outcome.

### 2.3. Denial-of-Service (DoS)

*   **Gas Exhaustion:** This is the most plausible impact. If the re-entrant call to `onUpdate` (and its full loop of calculations and potentially further, albeit tiny, transfers) consumes enough gas, the entire transaction (the user's original deposit/withdrawal in `StargateStaking`) could fail due to running out of gas. This would be a DoS for the user initiating the action.
*   **Revert due to other conditions:** If the re-entrant call leads to a state that causes a revert later in the execution (e.g., hitting a limit, though unlikely in `onUpdate` itself), it would also cause the main transaction to fail.

**Conclusion on DoS**: DoS through gas exhaustion is a credible risk if the complex re-entrancy path via `StargateStaking` is possible.

## 3. Re-entrant Calls to Other Functions in `StargateMultiRewarder`

If the `user` contract, upon receiving ETH from `_transferToken`, attempts to call other functions in `StargateMultiRewarder`:

*   **`connect()`**: This is `onlyStaking`. Not callable by the user contract.
*   **Admin Functions (`setReward`, `extendReward`, `setAllocPoints`, `stopReward`)**: These are `onlyOwner`. Not callable by the user contract.
*   **View Functions (`getRewards`, etc.)**: Calling view functions during re-entrancy is generally harmless from a state-change perspective. They would read the state as it was updated by the outer `onUpdate`'s Loop 1. This could contribute to gas usage.

Therefore, direct harmful re-entrancy into other `StargateMultiRewarder` functions by the `user` contract seems blocked by access controls.

## 4. Conclusion on Practical Exploitability and `nonReentrant` Guard

*   **Practical Exploitability for Theft/Corruption by Non-Owners**:
    *   The `onlyStaking` modifier on `onUpdate` is a strong first line of defense, preventing simple re-entrancy from the direct recipient of funds.
    *   If a more complex re-entrancy path via a vulnerability in `StargateStaking` were possible, the sequential update of `rewardDebt` in `RewardLib.update` appears to prevent straightforward double-claiming of the main reward amounts. The attacker would primarily get rewards for the tiny time delta of the re-entrant call.
    *   Thus, the direct financial exploitability by a non-owner through `onUpdate` re-entrancy is **low**.

*   **Practical Exploitability for DoS**:
    *   If the complex re-entrancy path is possible, DoS through gas exhaustion is a **medium** risk for the user whose transaction enables the re-entrancy.

*   **Recommendation for `nonReentrant` Guard**:
    *   **Reaffirm**: Despite the low direct theft risk, adding a `nonReentrant` guard to `StargateMultiRewarder.onUpdate` is **still strongly recommended**.
    *   **Rationale**:
        1.  **Defense-in-Depth**: It protects against unforeseen or highly complex re-entrancy pathways that may not be immediately obvious, especially in systems with multiple interacting contracts.
        2.  **Prevents DoS**: It would effectively block the re-entrant call, thus mitigating the gas exhaustion DoS risk described.
        3.  **Adherence to Best Practices**: Functions that modify state and make external calls (especially ETH transfers) should generally be protected against re-entrancy. `StakingLib` calls `onUpdate` *before* token transfers, making `onUpdate` an "interaction" from `StakingLib`'s perspective. While `StargateStaking` (caller of `StakingLib`) is guarded, adding protection to `onUpdate` itself hardens the system.
        4.  **Simplifies Reasoning**: It makes the state flow within `onUpdate` easier to reason about by ensuring it completes its execution atomically without interleaved calls.

In summary, while the current logic appears to resist trivial re-entrancy exploits for theft, the absence of a guard on `SMR.onUpdate` poses a DoS risk (if re-entrancy is possible via `StargateStaking`) and is a deviation from best practices for such a critical function.I have completed the detailed analysis of re-entrancy scenarios in `StargateMultiRewarder.onUpdate`, including modeling state changes and assessing potential impacts. The findings are ready to be saved to `reentrancy_onupdate_vulns.md`.
