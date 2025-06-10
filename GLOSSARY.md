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
