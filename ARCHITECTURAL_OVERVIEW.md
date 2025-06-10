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
