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
