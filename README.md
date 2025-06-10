# Stargate Staking Protocol Documentation

This document provides a comprehensive overview of the Stargate staking protocol, including its architecture, smart contract components, user flows, and a glossary of key terms.

---

## Architectural Overview

The Stargate protocol enables users to stake their Liquidity Provider (LP) tokens to earn rewards. This system is designed to incentivize liquidity provision by distributing various reward tokens to stakers over time.

The core of the protocol consists of several smart contracts and libraries that work together to manage staking and reward distribution:

### Main Contract Components:

*   **`StargateStaking.sol`**: This is the primary contract that users interact with to manage their staked LP tokens.
    *   **Role**: Handles user deposits and withdrawals of LP tokens. It keeps track of each user's staked balance and interacts with the `StargateMultiRewarder` to ensure rewards are accounted for when balances change.

*   **`StargateMultiRewarder.sol`**: This contract is responsible for managing and distributing multiple types of reward tokens to stakers.
    *   **Role**: It calculates, tracks, and distributes rewards based on the amount of LP tokens staked and the configured reward rates for each token. It interfaces with `StargateStaking` to update reward accruals when user stakes change.

*   **`stake/StakingLib.sol`**: This is a library contract that provides the core staking logic.
    *   **Role**: Used by `StargateStaking.sol` to perform fundamental staking operations, such as updating stake balances and managing user stake records. This modular approach keeps the main staking contract cleaner and focuses its responsibilities.

*   **`stake/RewardLib.sol`**: This library contains the logic for calculating reward amounts.
    *   **Role**: Used by `StargateMultiRewarder.sol` to compute the rewards accrued by each staker for each reward token. It likely implements algorithms to determine reward rates and distribution schedules.

*   **`stake/RewardRegistryLib.sol`**: This library is responsible for managing the configuration of reward pools.
    *   **Role**: Used by `StargateMultiRewarder.sol` to manage information about different reward tokens, their respective pools, emission rates, and other relevant parameters. It allows for flexible addition and management of various reward types.

### Interaction Flow:

The components interact in a coordinated manner to ensure seamless staking and reward distribution:

1.  **User Action (Deposit/Withdrawal)**: A user interacts with `StargateStaking.sol` to deposit or withdraw their LP tokens.
2.  **Update Stake**: `StargateStaking.sol` calls functions within `stake/StakingLib.sol` to update the user's staked LP token balance.
3.  **Notify Rewarder**: After the stake balance is updated, `StargateStaking.sol` notifies `StargateMultiRewarder.sol` about the change in the user's stake. This is crucial because rewards are typically calculated based on the amount staked and the duration.
4.  **Calculate and Track Rewards**: Upon notification, `StargateMultiRewarder.sol` utilizes `stake/RewardLib.sol` to calculate the updated reward accrual for the user. It also uses `stake/RewardRegistryLib.sol` to fetch the current reward rates and configurations for the various reward tokens being distributed.
5.  **Reward Distribution**: When a user decides to claim their rewards, or sometimes automatically during other operations like withdrawing stake, `StargateMultiRewarder.sol` handles the transfer of the accrued reward tokens to the user.

This architecture separates concerns, making the system more modular, auditable, and easier to maintain. `StargateStaking.sol` focuses on LP token management, while `StargateMultiRewarder.sol` and its associated libraries handle the complexities of multi-token reward calculations and distributions.

---

## Smart Contract Breakdown

This section provides a detailed look into the primary smart contracts and libraries within the Stargate staking protocol.

---

### 1. `StargateStaking.sol`

*   **Purpose**:
    `StargateStaking.sol` is the central contract for managing user LP token staking. It allows users to deposit their LP tokens to participate in the Stargate protocol and withdraw them when they choose. It also serves as the primary touchpoint for users to claim their accumulated rewards.

*   **Function**:
    *   **LP Token Management**: Handles the deposit (staking) and withdrawal (unstaking) of users' LP tokens.
    *   **Stake Tracking**: Maintains records of each user's staked balance and the total amount of LP tokens staked in the contract.
    *   **Reward Coordination**: When a user's stake changes (deposit, withdrawal), it notifies the `StargateMultiRewarder` contract to ensure reward calculations are updated accordingly.
    *   **Reward Claiming**: Provides a mechanism for users to claim their earned rewards, which are managed and distributed by `StargateMultiRewarder`.

*   **Mermaid Diagram**:

    ```mermaid
    graph TD
        User -->|deposit(amount)| StargateStaking
        User -->|withdraw(amount)| StargateStaking
        User -->|claimRewards()| StargateStaking
        StargateStaking -->|onUpdate(user, newStake)| StargateMultiRewarder
        StargateStaking -->|LPToken| LPTokenContract((LP Token Contract))
    ```

---

### 2. `StargateMultiRewarder.sol`

*   **Purpose**:
    `StargateMultiRewarder.sol` is responsible for managing and distributing multiple types of reward tokens to users who have staked their LP tokens in `StargateStaking.sol`. It handles the complexities of different reward schedules, rates, and token types.

*   **Function**:
    *   **Reward Management**: Configures and manages various reward tokens, including their emission rates and distribution periods.
    *   **Reward Calculation**: Calculates the amount of each reward token accrued by each staker based on their stake in `StargateStaking.sol`.
    *   **Reward Distribution**: Handles the actual transfer of reward tokens to users when they claim them.
    *   **Stake Updates**: Receives updates from `StargateStaking.sol` (via the `onUpdate` function or similar mechanism) whenever a user's staked balance changes, triggering recalculations of reward accruals.
    *   **Admin Functions**: Provides administrative functions to add new reward tokens, update reward rates, and manage the overall reward system.

*   **Mermaid Diagram**:

    ```mermaid
    graph TD
        subgraph "StargateMultiRewarder Interactions"
            StargateStaking -->|onUpdate(user, newStake)| StargateMultiRewarder
            Admin -->|configureReward(token, rate)| StargateMultiRewarder
            Admin -->|addRewardPool(...)| StargateMultiRewarder
            StargateMultiRewarder -->|calculateRewards()| RewardLib
            StargateMultiRewarder -->|getPoolInfo()| RewardRegistryLib
            StargateMultiRewarder -->|RewardToken| RewardTokenContract((Reward Token Contract))
            User -->|claimRewards()| StargateStaking -- triggers reward processing --> StargateMultiRewarder
        end
    ```

---

### 3. `stake/RewardLib.sol`

*   **Purpose**:
    `stake/RewardLib.sol` is a library contract that provides the core logic for calculating rewards. It encapsulates complex calculations to keep the `StargateMultiRewarder` contract cleaner and more focused on state management and external interactions.

*   **Function**:
    *   **Reward Accrual Calculation**: Contains functions to calculate the amount of rewards a user has earned over a period, based on their staked amount, the duration of the stake, and the reward rate.
    *   **Per-Token Calculation**: Can handle calculations for multiple reward tokens, potentially with different rates and rules.
    *   **Update Reward State**: Provides utility functions to update reward-related storage variables, such as `rewardPerTokenStored` and `userRewardPaid`.

*   **Mermaid Diagram**:

    ```mermaid
    graph TD
        StargateMultiRewarder -->|uses| RewardLib
        subgraph "RewardLib"
            direction LR
            CalculateUserRewards["calculateUserRewards(user, poolInfo)"]
            CalculateRewardPerToken["calculateRewardPerToken(poolInfo, totalStaked)"]
        end
        RewardLib --> CalculateUserRewards
        RewardLib --> CalculateRewardPerToken
    ```

---

### 4. `stake/RewardRegistryLib.sol`

*   **Purpose**:
    `stake/RewardRegistryLib.sol` is a library used by `StargateMultiRewarder.sol` to manage the configurations and parameters of different reward pools and reward tokens. It acts as a registry for reward-related information.

*   **Function**:
    *   **Reward Pool Management**: Stores and retrieves information about each reward pool, such as the reward token address, emission rate, start and end times for rewards, and total reward amount allocated to the pool.
    *   **Reward Token Information**: Manages details specific to each reward token.
    *   **Configuration Validation**: May include functions to validate new reward pool configurations before they are added.
    *   **Query Functions**: Provides functions for `StargateMultiRewarder` to easily query active reward pools and their parameters.

*   **Mermaid Diagram**:

    ```mermaid
    graph TD
        StargateMultiRewarder -->|uses| RewardRegistryLib
        subgraph "RewardRegistryLib"
            direction LR
            AddPool["addRewardPool(config)"]
            GetPoolInfo["getPoolInfo(poolId)"]
            UpdatePool["updatePoolConfig(poolId, newConfig)"]
            IsPoolActive["isPoolActive(poolId, timestamp)"]
        end
        RewardRegistryLib --> AddPool
        RewardRegistryLib --> GetPoolInfo
        RewardRegistryLib --> UpdatePool
        RewardRegistryLib --> IsPoolActive
    ```

---

### 5. `stake/StakingLib.sol`

*   **Purpose**:
    `stake/StakingLib.sol` is a library contract that provides the core logic for staking operations. It is used by `StargateStaking.sol` to handle the mechanics of users depositing and withdrawing LP tokens.

*   **Function**:
    *   **Stake Management**: Contains functions to update a user's staked balance when they deposit or withdraw LP tokens.
    *   **Total Supply Tracking**: Manages the total supply of staked LP tokens.
    *   **Balance Updates**: Provides safe and efficient mechanisms for updating stake balances, potentially including checks and balances.
    *   **User Stake Data**: May manage data structures that store information about each user's stake.

*   **Mermaid Diagram**:

    ```mermaid
    graph TD
        StargateStaking -->|uses| StakingLib
        subgraph "StakingLib"
            direction LR
            UpdateUserStake["updateUserStake(user, amount, isDeposit)"]
            GetUserStake["getUserStake(user)"]
            GetTotalStaked["getTotalStaked()"]
        end
        StakingLib --> UpdateUserStake
        StakingLib --> GetUserStake
        StakingLib --> GetTotalStaked
    ```
---

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

## Glossary

This glossary defines key terms used within the Stargate staking protocol documentation.

| Term                      | Definition                                                                                                                                                              |
| :------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **LP Token**              | Liquidity Provider Token. Represents a user's share in a liquidity pool (e.g., from a decentralized exchange). These are the tokens users stake in `StargateStaking.sol`. |
| **Reward Token**          | A token distributed as an incentive to users for staking their LP tokens. The `StargateMultiRewarder.sol` can manage and distribute multiple types of reward tokens.     |
| **Staking Pool (within `StargateStaking`)** | Refers to the specific pool within `StargateStaking.sol` where users deposit a particular type of LP token. Each Staking Pool is associated with a specific LP token address. |
| **Reward Pool (within `StargateMultiRewarder`)** | Refers to a configuration within `StargateMultiRewarder.sol` for a specific reward token. It defines parameters like the reward token address, emission rate (`rewardPerSec`), total allocation points (`allocPoints`), and accumulated rewards per share (`accRewardPerShare`). |
| **`allocPoints` (Allocation Points)** | Allocation Points. A weighting system used in `StargateMultiRewarder.sol` to determine the proportion of total rewards distributed to a specific Reward Pool (and thus to stakers of the corresponding LP token). Higher `allocPoints` mean a larger share of rewards. |
| **`accRewardPerShare` (Accumulated Reward Per Share)** | Accumulated Reward Per Share. A value maintained by `StargateMultiRewarder.sol` for each Reward Pool, representing the total rewards distributed per share of staked LP tokens up to a certain point in time. It's used to calculate individual user rewards. |
| **`rewardDebt`**          | The amount of reward a user has already been accounted for or paid out. When calculating pending rewards, the `rewardDebt` is subtracted from the total calculated rewards based on their current share and `accRewardPerShare` to find the actual claimable amount. |
| **`rewardPerSec`**        | Reward Per Second. The rate at which a specific reward token is emitted or distributed to a Reward Pool within `StargateMultiRewarder.sol`. This determines the flow of rewards into the pool over time. |
| **Staking**               | The act of depositing LP tokens into the `StargateStaking.sol` contract to earn rewards.                                                                                 |
| **Claiming**              | The act of a user requesting the withdrawal of their accumulated reward tokens from the `StargateMultiRewarder.sol` contract, typically initiated via `StargateStaking.sol`. |
| **`onUpdate`**            | A function name (or conceptual function) typically found in `StargateMultiRewarder.sol`, which is called by `StargateStaking.sol` whenever a user's staked LP token balance changes (due to deposit, withdrawal). This triggers reward recalculations. |
| **`emergencyWithdraw`**   | A function in `StargateStaking.sol` that allows users to withdraw their staked LP tokens immediately, bypassing standard reward calculation and potentially forfeiting any unclaimed rewards for the current period. A safety mechanism. |

---
