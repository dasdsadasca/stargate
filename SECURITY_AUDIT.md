# Stargate Staking Protocol Security Audit

## 1. Executive Summary

This security audit reviews the Stargate staking protocol, focusing on `StargateStaking.sol`, `StargateMultiRewarder.sol`, and their associated library contracts.

The overall security posture of the contracts is robust in many aspects, utilizing standard libraries like OpenZeppelin's `Ownable`, `ReentrancyGuard`, and `SafeERC20`. Arithmetic is generally safe due to Solidity version ^0.8.22.

However, several key areas of concern have been identified:

*   **Critical Concern - Owner Privileges in `StargateMultiRewarder.sol`**: The owner of `StargateMultiRewarder.sol` has the unilateral ability to withdraw all undistributed reward funds (both ERC20 and native ETH) via the `stopReward` function. This presents a significant centralization risk and a potential for rug pull of accumulated rewards.
*   **High Concern - Fund Freezing via Malicious/Faulty Rewarder**: The owner of `StargateStaking.sol` can set an arbitrary rewarder contract. If this rewarder is malicious or faulty, its `onUpdate()` or `connect()` methods could revert or exhaust gas, leading to deposits, withdrawals, and reward claims being blocked for specific LP pools.
*   **High Concern - Flash Loan Attack on Admin Pool Indexing**: An attacker could manipulate `staking.totalSupply()` via a flash loan during an admin's call to re-index reward pools (`setReward`, `setAllocPoints`). This can lead to an artificially inflated `accRewardPerShare`, allowing the attacker to claim excessive rewards (Theft of Unclaimed Yield).
*   **High Concern - "Poisonous Reward Token"**: If a configured reward token's transfer function consistently fails, it can block users from claiming *any* rewards for a staking pool and also prevent standard deposit/withdrawal operations, effectively causing a permanent DoS for affected users regarding those rewards and pool interactions.
*   **Medium Concern - Re-entrancy Vulnerability in `StargateMultiRewarder.onUpdate`**: The `onUpdate` function in `StargateMultiRewarder.sol` lacks a re-entrancy guard, potentially leading to gas griefing or complex state interactions if re-entered, although direct theft seems mitigated.
*   **Medium Concern - Administrative Desynchronization**: If `StargateStaking.sol` and `StargateMultiRewarder.sol` are managed by different, uncoordinated owners, misconfigurations can lead to reward distribution failures.

Other findings include potential DoS vectors of lesser severity and recommendations for stricter adherence to best practices.

The most critical recommendations involve addressing the centralization of power in `StargateMultiRewarder.sol`, mitigating the flash loan attack vector on admin functions, strategies for handling problematic reward tokens, and implementing a re-entrancy guard in `StargateMultiRewarder.onUpdate()`.

## 2. Scope

The following smart contracts and libraries were analyzed:

*   **`rewarder/StargateStaking.sol`**: Manages LP token staking (deposits, withdrawals).
*   **`stake/StakingLib.sol`**: Library used by `StargateStaking.sol` for core staking logic.
*   **`rewarder/StargateMultiRewarder.sol`**: Manages and distributes various reward tokens.
*   **`stake/RewardLib.sol`**: Library used by `StargateMultiRewarder.sol` for reward calculation logic.
*   **`stake/RewardRegistryLib.sol`**: Library used by `StargateMultiRewarder.sol` for managing reward pools and configurations.

Interfaces and OpenZeppelin contracts were reviewed for context but were not the primary subject of vulnerability analysis.

## 3. Findings

Findings are listed in descending order of severity.

---

### Finding 1: Owner Can Withdraw All Reward Funds from `StargateMultiRewarder`

*   **Title**: Owner Unilaterally Withdraws All Pending Rewards
*   **Description**: The `owner` of `StargateMultiRewarder.sol` can call the `stopReward(rewardToken, receiver, pullTokens=true)` function. If `pullTokens` is true, this function transfers the entire balance of the specified `rewardToken` (including native ETH if `rewardToken == address(0)`) held by the `StargateMultiRewarder` contract to an arbitrary `receiver` address. This allows the owner to confiscate all undistributed rewards at any time.
*   **Affected Contracts**: `rewarder/StargateMultiRewarder.sol`
*   **Impact**: Complete loss of all currently undistributed rewards for users if the owner acts maliciously or is compromised.
*   **Severity**: Critical
*   **Recommendation**:
    *   Implement a timelock mechanism for the `stopReward` function, especially when `pullTokens` is true.
    *   Require multi-signature approval for such critical actions.
    *   Clearly and prominently document this owner privilege to users, emphasizing the trust assumption.

---

### Finding 2: Fund Freezing via Malicious/Faulty Rewarder in `StargateStaking`

*   **Title**: Malicious or Faulty Rewarder Can Freeze Staking Operations
*   **Description**: The `StargateStaking.setPool()` function allows the owner to set an arbitrary contract address as the `IRewarder` for an LP token pool.
    1.  If the `newRewarder.connect(token)` call (made during `setPool`) reverts or has a gas bomb, it can prevent the owner from successfully setting or updating the rewarder for a pool.
    2.  More critically, the `deposit()`, `withdraw()`, and `claim()` operations in `StargateStaking` (via `StakingLib`) call `rewarder.onUpdate()`. If this `onUpdate()` call in the configured rewarder contract reverts or consumes excessive gas, all staking operations (deposits, withdrawals) and reward claims for that pool will fail, effectively freezing user funds or access to rewards for that pool.
*   **Affected Contracts**: `rewarder/StargateStaking.sol`, `stake/StakingLib.sol`
*   **Impact**: Users would be unable to deposit, withdraw LP tokens, or claim rewards from the affected pool(s). This can lead to permanent or temporary freezing of user assets or access.
*   **Severity**: High
*   **Recommendation**:
    *   The owner of `StargateStaking.sol` must exercise extreme caution and due diligence when setting rewarder addresses.
    *   Consider implementing an emergency mechanism in `StargateStaking.sol` allowing the owner to disable or replace a clearly faulty rewarder in a way that bypasses the problematic `connect()` or `onUpdate()` calls (e.g., a function to set a zero-address or a pass-through dummy rewarder if the current one is consistently failing).
    *   Document the high degree of trust placed in the configured rewarder contracts.

---

### Finding 3: Flash Loan Attack on Admin Pool Indexing (Theft of Unclaimed Yield)

*   **Title**: Flash Loan Attack on Admin Pool Indexing Leading to Theft of Unclaimed Yield
*   **Description**: An attacker can front-run an admin's call to `setReward` or `setAllocPoints` in `StargateMultiRewarder.sol`. These admin functions trigger `_indexRewardTokenPools`, which uses a live `staking.totalSupply()` call to update `pool.accRewardPerShare`. By using a flash loan to temporarily withdraw a large amount of the target LP token, the attacker significantly deflates `staking.totalSupply()` just before the admin's transaction executes. The subsequent execution of `RewardLib._index` uses this deflated supply, resulting in an artificially inflated `pool.accRewardPerShare`. The attacker then re-deposits LP tokens (or stakes new ones) and, upon the next reward update (e.g., their own deposit), claims a disproportionate amount of rewards based on the inflated `accRewardPerShare`.
*   **Affected Contracts**: `rewarder/StargateMultiRewarder.sol`, `stake/RewardLib.sol`, `rewarder/StargateStaking.sol` (as source of `totalSupply`).
*   **Impact**: Theft of unclaimed yield from the reward pool. The attacker unfairly benefits at the expense of other legitimate stakers and the overall reward budget for the pool. The pool's `accRewardPerShare` remains tainted until corrected.
*   **Severity**: High
*   **Recommendation**:
    *   **Primary Mitigation**: Modify `StargateMultiRewarder._indexRewardTokenPools` (and functions that call it) to avoid using a live spot `staking.totalSupply()`. Instead, consider maintaining an internal, more stable view of `totalSupply` within `StargateMultiRewarder`, updated only during `SMR.onUpdate` (which uses the safer `oldSupply` from `StargateStaking`). Alternatively, allow admins to provide a recent, trusted `totalSupply` value directly to functions that trigger re-indexing.
    *   **Secondary Mitigation (Operational)**: SMR Admins should exercise extreme caution when calling functions like `setReward` or `setAllocPoints`. Avoid these operations during high market volatility or when large, unusual `totalSupply` fluctuations are observed. Consider announcing such maintenance to discourage front-running.
    *   Investigate the feasibility of using a Time-Weighted Average Price (TWAP) oracle for `totalSupply` readings for these critical admin functions, though this adds complexity.

---

### Finding 4: "Poisonous Reward Token" Leading to Permanent DoS

*   **Title**: Problematic Reward Token Can Cause Permanent DoS for Users
*   **Description**: If a reward token, after being configured by an admin in `StargateMultiRewarder.sol`, starts to behave problematically (e.g., its `transfer` function always reverts, consumes excessive gas, or is subject to blacklisting affecting the rewarder contract or user), it will cause the `_transferToken` call within `StargateMultiRewarder.onUpdate` to fail for any user eligible for that reward.
*   **Affected Contracts**: `rewarder/StargateMultiRewarder.sol`. Indirectly impacts usability of `rewarder/StargateStaking.sol`.
*   **Impact**:
    *   Any user eligible for the "poisonous" reward token will have their `onUpdate` call revert.
    *   This blocks them from claiming *any* rewards (even other, non-poisonous rewards) from the staking pool(s) where this reward token is active.
    *   It also blocks them from performing standard `deposit` or `withdraw` operations on the associated `StargateStaking` pool, as these actions trigger `onUpdate`.
    *   Affected users can only retrieve their principal LP tokens via `StargateStaking.emergencyWithdraw()`, which forfeits all pending rewards because it bypasses `onUpdate`. This constitutes a permanent DoS for reward claiming and normal staking operations for affected users until an admin intervenes.
*   **Severity**: High
*   **Recommendation**:
    *   **Admin Diligence**: Admins must thoroughly vet reward tokens before adding them, including their upgradeability, pausability, blacklist functions, and gas costs on transfer.
    *   **Emergency Admin Action**: Admins should be prepared to promptly use `StargateMultiRewarder.stopReward(poisonousTokenAddress, adminAddress, false)` to halt further reward calculations and transfer attempts for the problematic token. The `pullTokens=false` parameter is crucial if the token's transfer is failing.
    *   **Architectural Consideration (Long-term)**: Investigate mechanisms for users to selectively claim specific reward tokens, bypassing problematic ones. This is a complex change but would offer greater resilience. Another approach could be for `onUpdate` to attempt all transfers but not revert the entire transaction if only some non-critical transfers fail, instead emitting events for failed transfers (this also has trade-offs regarding atomicity).
    *   **User Guidance**: Clearly inform users about the `emergencyWithdraw` function as a last resort, explaining the consequence of forfeiting rewards.

---

### Finding 5: Re-entrancy Vulnerability in `StargateMultiRewarder.onUpdate`

*   **Title**: `StargateMultiRewarder.onUpdate` Lacks Re-entrancy Guard
*   **Description**: The `StargateMultiRewarder.onUpdate()` function, which is called by `StargateStaking.sol` during deposits, withdrawals, and claims, is not protected by a re-entrancy guard. This function first calculates rewards (updating `rewardDebt`) and then, in a separate loop, transfers reward tokens via `_transferToken()`. If `_transferToken()` (especially for native ETH transfers using `to.call{value: amount}("")`) allows the recipient to call back into `StargateMultiRewarder.onUpdate()` or other functions (potentially via a complex path involving `StargateStaking`), it could lead to unexpected behavior.
    While direct double-claiming of rewards for the same triggering event appears unlikely due to `rewardDebt` being updated before transfers and the `onlyStaking` modifier, re-entrancy could:
    *   Lead to gas griefing if the re-entrant call consumes remaining gas, causing the original transaction to fail (DoS).
    *   Complicate reasoning about contract execution and state, potentially enabling more subtle exploits if other system components have vulnerabilities.
*   **Affected Contracts**: `rewarder/StargateMultiRewarder.sol`
*   **Impact**: Potential for user transactions to be forced to fail (gas exhaustion DoS). Low risk of direct theft but increases system complexity and risk of unforeseen interactions.
*   **Severity**: Medium
*   **Recommendation**: Add an OpenZeppelin `nonReentrant` guard to the `StargateMultiRewarder.onUpdate()` function as a defense-in-depth measure to prevent all re-entrancy scenarios, including DoS through gas griefing.

---

### Finding 6: Administrative Desynchronization Between Staking and Rewarder Contracts

*   **Title**: Potential for Operational Issues from Uncoordinated Admin Actions
*   **Description**: `StargateStaking.sol` (SS) and `StargateMultiRewarder.sol` (SMR) are distinct contracts, potentially with different owners. Lack of coordination can lead to issues:
    *   The SS owner might set a rewarder address in `SS.setPool()` that is not the intended SMR instance or is misconfigured.
    *   The SMR owner might configure rewards (e.g., `setReward`, `setAllocPoints`) for an LP token that SS does not manage or for which SS points to a different rewarder.
    This can result in `onUpdate` calls not reaching the intended SMR, rewards accruing in SMR but not being claimable via SS, or general failure of the reward system for specific pools.
*   **Affected Contracts**: `rewarder/StargateStaking.sol`, `rewarder/StargateMultiRewarder.sol`
*   **Impact**: Users may not receive rewards as expected, or be unable to claim them. Specific staking pools might become non-functional from a rewards perspective.
*   **Severity**: Medium
*   **Recommendation**:
    *   Ideally, the ownership and administration of SS and SMR should be closely aligned, managed by the same entity or a tightly coordinated multi-sig.
    *   Implement robust off-chain monitoring and communication protocols if administration is separate.
    *   Consider emitting events upon critical reconfigurations in one contract that might necessitate changes in the other, to aid administrators.

---

### Finding 7: Potential `totalSupply` Manipulation Affecting `accRewardPerShare` During Admin Re-indexing (Context for Flash Loan Finding)

*   **Title**: Admin-Triggered Re-indexing May Use Manipulated `totalSupply` (Note: This is now part of Finding 3 with higher severity)
*   **Description**: SMR admin functions like `setReward` and `setAllocPoints` call `_indexRewardTokenPools`, which in turn uses `staking.totalSupply()` to update `accRewardPerShare` for reward pools. The core user reward flow in `onUpdate` is protected as it uses `oldSupply` passed from `StargateStaking`. However, if an attacker could manipulate `StargateStaking.totalSupply()` (e.g., via a flash loan) and an SMR admin executes a function like `setReward` or `setAllocPoints` within the same transaction, the `accRewardPerShare` could be calculated based on this temporarily skewed `totalSupply`. This would affect all subsequent reward calculations for users in those pools.
*   **Affected Contracts**: `rewarder/StargateMultiRewarder.sol`, `stake/RewardLib.sol`
*   **Impact**: Incorrect calculation of `accRewardPerShare` leading to unfair distribution of rewards. This is a component of the "Flash Loan Attack on Admin Pool Indexing" (Finding 3).
*   **Severity**: Medium (Elevated to High as part of Finding 3)
*   **Recommendation**: See recommendations for Finding 3.

---

### Finding 8: `withdrawToAndCall` External Call Failure Risk

*   **Title**: `withdrawToAndCall` Recipient Can Cause Transaction Failure
*   **Description**: The `StargateStaking.withdrawToAndCall()` function makes an external call to `to.onWithdrawReceived()`. If the recipient contract `to` is malicious or buggy, it can cause this call to revert or consume all available gas. This would lead to the entire `withdrawToAndCall` transaction failing. Users can still use the standard `withdraw()` function. The `StargateStaking` contract itself is protected by a `nonReentrant` guard.
*   **Affected Contracts**: `rewarder/StargateStaking.sol`
*   **Impact**: Users attempting to use `withdrawToAndCall` with a problematic recipient contract will have their transactions fail. This does not risk staked funds but affects the utility of this specific function.
*   **Severity**: Low (Medium for the specific function's utility)
*   **Recommendation**: Document clearly to users that the success of `withdrawToAndCall` is dependent on the recipient contract's behavior. The current design (reverting ambiguously to save gas on checks) is an accepted trade-off.

---

### Finding 9: Order of Operations in `StakingLib` (Interaction Before Effect)

*   **Title**: External Call to Rewarder Before Token Transfer
*   **Description**: In `StakingLib.deposit()`, the `rewarder.onUpdate()` call is made before `token.safeTransferFrom()`. Similarly, in `StakingLib.withdraw()`, `rewarder.onUpdate()` is called before `token.safeTransfer()`. This is a deviation from the Checks-Effects-Interactions pattern. While `StargateStaking` functions are `nonReentrant`, this ordering relies more heavily on the behavior of the rewarder and the reentrancy guard.
*   **Affected Contracts**: `stake/StakingLib.sol`
*   **Impact**: If the `nonReentrant` guard in `StargateStaking` were bypassed or if re-entrancy affected the rewarder itself, this pattern could lead to vulnerabilities.
*   **Severity**: Low (due to existing `nonReentrant` guard in caller)
*   **Recommendation**: For maximal adherence to security best practices and defense-in-depth, refactor to perform all local state changes and token transfers *before* the external `rewarder.onUpdate()` call.

---

### Finding 10: DoS Potential in View Functions and Registry Limits

*   **Title**: Minor DoS Vectors in View Functions and Configuration Limits
*   **Description**:
    1.  View functions in `StargateStaking.sol` (`tokens()`) and `StargateMultiRewarder.sol` (`getRewards`, `allocPointsByReward`, etc.) iterate over arrays. If these arrays become extremely large (e.g., many token pools), calls to these functions could consume excessive gas and fail.
    2.  `RewardRegistryLib.sol` imposes limits (`MAX_ACTIVE_POOLS_PER_REWARD = 100`, `MAX_ACTIVE_REWARD_TOKENS = 100`). Reaching these hardcoded limits will prevent registration of new reward tokens or new pools for a reward, which is a form of DoS for new configurations.
*   **Affected Contracts**: `rewarder/StargateStaking.sol`, `rewarder/StargateMultiRewarder.sol`, `stake/RewardRegistryLib.sol`
*   **Impact**: Off-chain applications or other contracts relying on these views might fail. New reward configurations might be blocked if limits are met.
*   **Severity**: Low / Informational
*   **Recommendation**:
    *   Encourage clients to use paginated versions of view functions where available.
    *   Document the fixed limits in `RewardRegistryLib`. If these limits might be too restrictive for future growth, consider making them configurable by the owner (with appropriate safeguards).

---

## 4. General Recommendations

*   **Defense in Depth**: While specific vulnerabilities have mitigations, continue to apply security best practices across all contracts. This includes:
    *   Strict adherence to the Checks-Effects-Interactions pattern.
    *   Comprehensive use of re-entrancy guards on all functions involving external calls and state changes (specifically `StargateMultiRewarder.onUpdate`).
    *   Use of `SafeERC20` and latest Solidity versions.
*   **Owner Responsibilities & Trust Assumptions**:
    *   The current system places significant trust in the owner(s) of `StargateStaking.sol` and especially `StargateMultiRewarder.sol`. These trust assumptions should be transparently and clearly communicated to users of the protocol.
    *   Consider strengthening governance mechanisms for highly privileged operations (e.g., timelocks, multi-signature wallets for `stopReward` and admin functions that update reward indexing) to reduce centralization risks.
*   **Testing**: Implement comprehensive unit and integration tests, including tests for edge cases, flash loan scenarios, poisonous token interactions, and potential re-entrancy scenarios identified.
*   **Monitoring**: Actively monitor contract activity, especially admin function calls, `totalSupply` fluctuations during admin operations, and large fund movements, to detect any suspicious behavior quickly.
*   **Documentation**: Ensure all function behaviors, especially those with security implications (like `setPool`, `stopReward`, `onUpdate` interactions), are thoroughly documented for developers and users. Explain the rationale behind specific design choices, like the comment in `StargateStaking.setPool` regarding "prevents re-adding of an old rewarder".

---
