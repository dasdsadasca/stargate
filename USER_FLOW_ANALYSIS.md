## User Flow Analysis

This section details the typical user interaction flows within the Stargate staking protocol, outlining the sequence of contract calls and key interactions involved in common operations.

---

### 1. Staking LP Tokens

*   **Description**:
    A user decides to stake their LP tokens to earn rewards. They interact with the `StargateStaking` contract, which then updates their stake using `StakingLib` and notifies the `StargateMultiRewarder` to ensure their reward calculations begin or are adjusted.

*   **Flow**:
    1.  **User** initiates a `deposit` transaction to `StargateStaking.sol`, specifying the amount of LP tokens to stake.
    2.  `StargateStaking.sol` receives the LP tokens.
    3.  `StargateStaking.sol` calls `StakingLib` to update the user's staked balance and the total staked supply.
    4.  `StakingLib` updates the relevant storage variables.
    5.  `StargateStaking.sol` then calls `StargateMultiRewarder.sol`'s `onUpdate` (or a similar function) to inform it of the user's new or increased stake. This step is crucial for the `StargateMultiRewarder` to accurately calculate and accrue rewards for the user.
    6.  `StargateMultiRewarder.sol` updates its internal accounting for user rewards, potentially using `RewardLib` and `RewardRegistryLib`.

*   **Mermaid Diagram**:

    ```mermaid
    sequenceDiagram
        actor User
        participant StargateStaking
        participant StakingLib
        participant StargateMultiRewarder
        participant RewardLib
        participant RewardRegistryLib

        User->>+StargateStaking: deposit(amountLP)
        StargateStaking->>+StakingLib: updateUserStake(user, amountLP, isDeposit=true)
        StakingLib-->>-StargateStaking: success
        StargateStaking->>+StargateMultiRewarder: onUpdate(user, newStakeBalance)
        StargateMultiRewarder->>RewardRegistryLib: getPoolInfo()
        StargateMultiRewarder->>+RewardLib: calculateUserRewards(user, poolInfo)
        RewardLib-->>-StargateMultiRewarder: updatedRewardState
        StargateMultiRewarder-->>-StargateStaking: success
        StargateStaking-->>-User: StakingSuccessfulEvent
    ```

---

### 2. Claiming Rewards

*   **Description**:
    A user wishes to claim the rewards they have accumulated from staking their LP tokens. The process typically involves the `StargateStaking` contract, which coordinates with the `StargateMultiRewarder` to calculate and transfer the owed rewards.

*   **Flow**:
    1.  **User** initiates a `claimRewards` (or similar, often part of `deposit` or `withdraw` or a separate `getReward` function) transaction to `StargateStaking.sol`.
    2.  `StargateStaking.sol` may first call `StakingLib` to ensure the user's current stake is accurately reflected, or directly proceed to interact with the rewarder.
    3.  `StargateStaking.sol` calls a function on `StargateMultiRewarder.sol` (e.g., `getReward` or as part of `onUpdate` if claiming is combined with other actions) to process the reward claim.
    4.  `StargateMultiRewarder.sol` calculates the pending rewards for the user. This involves:
        *   Fetching reward pool configurations using `RewardRegistryLib`.
        *   Calculating the precise reward amounts using `RewardLib`.
    5.  `StargateMultiRewarder.sol` transfers the calculated reward tokens to the user.
    6.  `StargateMultiRewarder.sol` updates its internal state to reflect that rewards have been paid out.

*   **Mermaid Diagram**:
    *(Note: Claiming can sometimes be bundled with `deposit` or `withdraw`. This diagram assumes a dedicated claim or a flow where rewards are processed first.)*

    ```mermaid
    sequenceDiagram
        actor User
        participant StargateStaking
        participant StargateMultiRewarder
        participant RewardLib
        participant RewardRegistryLib
        participant RewardTokenA
        participant RewardTokenB

        User->>+StargateStaking: claimRewards() / getReward()
        StargateStaking->>+StargateMultiRewarder: processRewards(user) / onUpdate(user, currentStake)
        Note over StargateMultiRewarder: May first update rewards based on current stake
        StargateMultiRewarder->>RewardRegistryLib: getActivePools()
        StargateMultiRewarder->>+RewardLib: calculatePendingRewards(user, allPoolInfo)
        RewardLib-->>-StargateMultiRewarder: pendingRewardsAmounts
        StargateMultiRewarder->>RewardTokenA: transfer(user, amountA)
        StargateMultiRewarder->>RewardTokenB: transfer(user, amountB)
        StargateMultiRewarder-->>-StargateStaking: RewardsClaimedEvent
        StargateStaking-->>-User: RewardsTransferred
    ```

---

### 3. Withdrawing LP Tokens

*   **Description**:
    A user decides to withdraw their staked LP tokens from the protocol. This action also typically triggers a reward update and payout.

*   **Flow**:
    1.  **User** initiates a `withdraw` transaction to `StargateStaking.sol`, specifying the amount of LP tokens to unstake.
    2.  `StargateStaking.sol` first interacts with `StargateMultiRewarder.sol` (via `onUpdate` or similar) to ensure any pending rewards are calculated and paid out based on the stake *before* it's reduced.
    3.  `StargateMultiRewarder.sol` calculates and distributes any pending rewards to the user, using `RewardLib` and `RewardRegistryLib` as needed.
    4.  `StargateStaking.sol` then calls `StakingLib` to update the user's staked balance (decreasing it) and the total staked supply.
    5.  `StakingLib` updates the relevant storage variables.
    6.  `StargateStaking.sol` transfers the specified amount of LP tokens back to the user.
    7.  `StargateStaking.sol` may call `StargateMultiRewarder.sol`'s `onUpdate` again if the reward system needs to be informed about the zeroing out or reduction of stake *after* rewards for the previous period were paid.

*   **Mermaid Diagram**:

    ```mermaid
    sequenceDiagram
        actor User
        participant StargateStaking
        participant StakingLib
        participant StargateMultiRewarder
        participant RewardLib
        participant RewardRegistryLib
        participant LPTokenContract

        User->>+StargateStaking: withdraw(amountLP)
        %% Optional: Claim rewards first if not bundled
        StargateStaking->>+StargateMultiRewarder: onUpdate(user, currentStake)  // Update rewards before withdrawal
        StargateMultiRewarder->>RewardRegistryLib: getPoolInfo()
        StargateMultiRewarder->>+RewardLib: calculateUserRewardsAndPay(user, poolInfo) // Calculates and pays rewards
        RewardLib-->>-StargateMultiRewarder: rewardsPaidConfirmation
        StargateMultiRewarder-->>-StargateStaking: successOrRewardsPaidEvent

        StargateStaking->>+StakingLib: updateUserStake(user, amountLP, isDeposit=false)
        StakingLib-->>-StargateStaking: success
        StargateStaking->>LPTokenContract: transfer(user, amountLP)
        %% Optional: Notify rewarder of final state if necessary
        StargateStaking->>StargateMultiRewarder: onUpdate(user, newStakeBalance) // Inform of new (lower) balance
        StargateMultiRewarder-->>StargateStaking: ack
        StargateStaking-->>-User: WithdrawalSuccessfulEvent
    ```

---

### 4. Emergency Withdrawal

*   **Description**:
    An emergency withdrawal function allows users to retrieve their staked LP tokens quickly, typically bypassing the standard reward calculation and distribution mechanisms. This is a safety feature, often forfeiting any pending rewards for the current period.

*   **Flow**:
    1.  **User** initiates an `emergencyWithdraw` transaction to `StargateStaking.sol`.
    2.  `StargateStaking.sol` calls `StakingLib` to immediately update the user's staked balance (setting it to zero or reducing it by the withdrawn amount) and the total staked supply.
    3.  `StakingLib` updates the relevant storage variables.
    4.  `StargateStaking.sol` transfers the user's LP tokens back to them.
    5.  **Crucially**, in a typical emergency withdrawal, `StargateStaking.sol` **does not** call `StargateMultiRewarder.sol`'s `onUpdate` function for reward processing. The user forfeits any rewards that were not already claimed.

*   **Mermaid Diagram**:

    ```mermaid
    sequenceDiagram
        actor User
        participant StargateStaking
        participant StakingLib
        participant LPTokenContract

        User->>+StargateStaking: emergencyWithdraw()
        StargateStaking->>+StakingLib: updateUserStake(user, fullStakedAmount, isDeposit=false) // Or similar to zero out stake
        StakingLib-->>-StargateStaking: success
        StargateStaking->>LPTokenContract: transfer(user, fullStakedAmount)
        Note right of StargateStaking: No call to StargateMultiRewarder.onUpdate() for rewards.
        StargateStaking-->>-User: EmergencyWithdrawalEvent
    ```

---
