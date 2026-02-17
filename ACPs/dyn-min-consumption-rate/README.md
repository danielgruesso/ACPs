| ACP | XXX |
| :--- | :--- |
| **Title** | Dynamic Minimum Consumption Rate |
| **Author(s)** | Martin Eckardt ([@martineckardt](https://github.com/martineckardt)), Daniel Gruesso ([@danielgruesso](https://github.com/danielgruesso)) |
| **Status** | Proposed |
| **Track** | Standards |

## Abstract

This ACP introduces two adjustable parameters to the Primary Network staking reward formula on the P-Chain: a **Reward Reduction Factor (α)** that uniformly scales all staking rewards, and revised bounds for **MinConsumptionRate** that widen the spread between short and long-duration staking yields. Together, these changes give the Avalanche Community fine-grained, governance-controlled levers to reduce token emissions and strengthen the economic incentive for longer staking commitments, without redesigning the reward formula or changing `MaximumSupply`.

The modified formula multiplies the existing `PotentialReward` calculation by α (bounded between 0.30 and 1.00). Independently, `MinConsumptionRate` is permitted to range from 0.05 to 0.10 (currently fixed at 0.10), while `MaxConsumptionRate` remains fixed at 0.12. At the proposed bounds, annualized yields range from approximately 1.94% to 6.45% for 1-year stakers and 0.81% to 6.45% for shortest-duration stakers, depending on governance-selected values. Both parameters are set via an on-chain governance mechanism with safety bounds enforced at the protocol level.

## Motivation

The current Primary Network reward formula produces annualized staking yields of approximately 6.45% (1-year duration) and 5.41% (2-week duration). This creates two issues:

1. **Limited emission control:** The community has no mechanism to reduce or increase the overall reward rate in response to changing market conditions, staking participation levels, or monetary policy goals. The only way to adjust rewards today is through a hard-coded parameter change requiring a network upgrade.

2. **Weak duration differentiation:** A 2-week staker earns roughly 84% of the annualized rate of a 1-year staker. This narrow spread provides insufficient incentive for validators to commit capital for longer periods, even though longer commitments improve network stability and reduce validator churn.

### Why Now

Recent discussions around reducing minimum staking duration (from 2 weeks to 2 days) and ongoing community interest in emission sustainability make this an appropriate time to introduce governance-adjustable reward parameters. These changes are complementary: shorter minimum durations increase accessibility, while this ACP ensures that reward economics can be tuned to maintain appropriate incentive gradients.

### Goals

- Introduce a governance-adjustable scalar (α) that controls the overall level of staking rewards.
- Allow `MinConsumptionRate` to be lowered below its current value, widening the yield gap between short and long staking durations.
- Keep both parameters within protocol-enforced safety bounds to prevent governance decisions from making staking either trivially unrewarding or excessively inflationary.
- Preserve the existing formula structure; the only structural change is multiplication by α.

### Non-Goals

- This ACP does **not** redesign the staking reward formula beyond the introduction of α.
- This ACP does **not** change `MaximumSupply` or the broader emission schedule beyond what α and `MinConsumptionRate` indirectly influence.
- This ACP does **not** modify delegation fee structures, though delegation rewards are affected proportionally (see [Delegation](#delegation)).

## Specification

### Modified Reward Formula

The current P-Chain reward formula calculates `PotentialReward` as:

```
PotentialReward = (MaximumSupply - Supply) × (Stake / Supply) × (StakingPeriod / MintingPeriod) × EffectiveConsumptionRate
```

This ACP modifies it to:

```
PotentialReward = α × (MaximumSupply - Supply) × (Stake / Supply) × (StakingPeriod / MintingPeriod) × EffectiveConsumptionRate
```

where:

```
EffectiveConsumptionRate = (MinConsumptionRate / PercentDenominator) × (1 - StakingPeriod / MintingPeriod)
                        + (MaxConsumptionRate / PercentDenominator) × (StakingPeriod / MintingPeriod)
```

The `EffectiveConsumptionRate` calculation is unchanged from the current implementation. The only structural addition is the multiplicative factor α.

### Parameter Definitions

| Parameter | Type | Current Value | Proposed Range | Default (at activation) |
| :--- | :--- | :--- | :--- | :--- |
| **α** (Reward Reduction Factor) | `uint64` scaled by 10⁴ (i.e., 10000 = 1.00) | 1.00 (implicit; not present in current code) | 0.30 – 1.00 | 1.00 |
| **MinConsumptionRate** | `uint64` (existing parameter) | 0.10 (i.e., 10% of `PercentDenominator`) | 0.05 – 0.10 | 0.10 |
| **MaxConsumptionRate** | `uint64` (existing parameter) | 0.12 | Fixed at 0.12 | 0.12 (unchanged) |

**Activation default:** Both α and `MinConsumptionRate` default to their current-equivalent values at activation. This means the upgrade is reward-neutral on day one. Changes require explicit governance action.

### Parameter Bounds and Rationale

**α (Reward Reduction Factor): 0.30 – 1.00**

| Bound | Value | Effect | Rationale |
| :--- | :--- | :--- | :--- |
| Floor | 0.30 | 1-year APR ≈ 1.94% | Keeps staking competitive with alternative yield sources (e.g., DeFi lending, L2 staking). Below ~2%, rational capital would exit staking, degrading network security. |
| Ceiling | 1.00 | 1-year APR ≈ 6.45% | Equivalent to today's rate. No mechanism to increase rewards beyond the current formula's output. |

**MinConsumptionRate: 0.05 – 0.10**

| Bound | Value | Effect | Rationale |
| :--- | :--- | :--- | :--- |
| Floor | 0.05 | At α = 1.0, 2-week APR ≈ 2.83% (44% of 1-year rate) | Maintains a meaningful reward for short-duration stakers. Below 0.05, short-duration rewards approach zero, which could deter new validators. |
| Ceiling | 0.10 | Current behavior | At 0.10, the 2-week rate is already ~84% of the 1-year rate; the ceiling preserves backward compatibility. |

**MaxConsumptionRate: Fixed at 0.12**

`MaxConsumptionRate` is not made adjustable in this ACP. It anchors the maximum achievable rate for the longest staking duration and provides a stable reference point. Making it adjustable alongside α would create redundant controls (α already scales all rates uniformly). A future ACP could propose adjustability if needed.

### What α Controls vs. What MinConsumptionRate Controls

**α controls the overall reward level.** It is a uniform scalar applied to all staking durations equally. Changing α from 1.0 to 0.5 halves every staker's annualized yield, regardless of duration.

**MinConsumptionRate controls the spread between short and long durations.** It determines the `EffectiveConsumptionRate` at the shortest staking duration. Lowering `MinConsumptionRate` reduces short-duration yields relative to long-duration yields, creating a steeper incentive curve that rewards commitment.

These two parameters are orthogonal by design: one sets the level, the other sets the slope.

### Impact Analysis

All figures assume current supply of ~468M AVAX, giving a remaining emission ratio `(MaximumSupply − Supply) / Supply ≈ 0.5375`.

#### 1-Year Staking APR

The 1-year rate depends only on α because at `StakingPeriod = MintingPeriod` (ratio = 1), `EffectiveConsumptionRate = MaxConsumptionRate` regardless of `MinConsumptionRate`.

| | α = 0.30 | α = 1.00 |
| :--- | :--- | :--- |
| **1-Year APR** | 1.94% | 6.45% |

#### 2-Week Staking APR

| | α = 0.30 | α = 1.00 |
| :--- | :--- | :--- |
| **MinConsumptionRate = 0.05** | 0.85% | 2.83% |
| **MinConsumptionRate = 0.10** | 1.63% | 5.41% |
| **MinConsumptionRate = 0.12** | 1.94% | 6.45% |

#### Combined View (1-Year / 2-Week APR)

| | α = 0.30 | α = 1.00 |
| :--- | :--- | :--- |
| **MinConsumptionRate = 0.05** | 1.94% / 0.85% | 6.45% / 2.83% |
| **MinConsumptionRate = 0.12** | 1.94% / 1.94% | 6.45% / 6.45% |

At `MinConsumptionRate = MaxConsumptionRate = 0.12`, staking duration has zero effect on annualized yield. At `MinConsumptionRate = 0.05`, a 2-week staker earns roughly 44% of the annualized rate of a 1-year staker.

#### 2-Day Minimum Staking Duration

If the minimum staking duration is reduced from 2 weeks to 2 days (as discussed in [ACP-273](https://github.com/avalanche-foundation/ACPs/pull/273)), the effect on reward dynamics is modest:

| | α = 0.30 | α = 1.00 |
| :--- | :--- | :--- |
| **MinConsumptionRate = 0.05** | 1.94% / 0.81% | 6.45% / 2.71% |
| **MinConsumptionRate = 0.12** | 1.94% / 1.94% | 6.45% / 6.45% |

The 2-day shortest-duration APR at `MinConsumptionRate = 0.05` is 2.71% vs. 2.83% for 2-week, a difference of 0.12pp. This is because `EffectiveConsumptionRate` interpolates linearly and `MinConsumptionRate` already dominates at short durations (where `StakingPeriod / MintingPeriod` is near zero). The practical implication is that reducing minimum staking duration does not meaningfully change reward dynamics; the parameter bounds proposed here remain appropriate regardless of minimum duration policy.

### Governance and Control Mechanism

#### Who Sets α and MinConsumptionRate

Both parameters are set via the existing Avalanche network parameter governance mechanism. Validators signal their preferred values through their node configuration (analogous to existing AvalancheGo config flags for ACP support signaling). The network computes the effective value as the **stake-weighted median** of all active validators' signaled preferences.

#### Update Frequency and Rate Limits

- Parameters are recalculated at each **reward epoch boundary** (to be defined; suggested: every 8,192 blocks on the P-Chain, approximately every ~3 hours at current block times).
- A **maximum change rate** of ±5% of the current value per epoch applies to both α and `MinConsumptionRate`. This prevents abrupt reward shocks from rapid governance swings.
- Example: If α is currently 0.80, it can move to at most 0.84 or at least 0.76 in a single epoch.

#### Default / Fallback Behavior

- If fewer than 80% of active stake has signaled a preference for a parameter, the parameter retains its current value (no change).
- At activation, both parameters default to current-equivalent values (α = 1.00, `MinConsumptionRate` = 0.10).
- Validators that do not explicitly configure a preference are treated as voting for the current value (status quo bias).

#### Activation Delay

After the stake-weighted median is computed, the new value takes effect after a **1-epoch delay** to allow tooling, dashboards, and staking calculators to update.

### Delegation

Delegation rewards are computed as a fraction of the validator's total reward, which itself is derived from `PotentialReward`. Because α multiplies `PotentialReward` uniformly, delegation rewards scale proportionally. No changes to delegation fee parameters or delegation reward splitting logic are required.

Delegators staking for shorter durations will see the same relative yield reduction as validators when `MinConsumptionRate` is lowered. This is intentional: the economic signal (longer commitment = higher yield) should apply consistently to all participants.

## Rationale

### Why a Multiplicative Factor Instead of Adjusting MinConsumptionRate Alone

Adjusting `MinConsumptionRate` only changes the short end of the yield curve. To reduce all emissions uniformly (e.g., in response to high staking participation or a desire to slow supply growth), the community would need to lower both `MinConsumptionRate` and `MaxConsumptionRate` in lockstep. Introducing α provides a single knob for uniform scaling, keeping the two concerns (level vs. spread) cleanly separated.

### Why Not Make MaxConsumptionRate Adjustable

Adding a third adjustable parameter increases governance complexity without clear benefit. α already provides uniform scaling. If `MaxConsumptionRate` were also adjustable, the parameter space would have degenerate configurations (e.g., lowering `MaxConsumptionRate` below `MinConsumptionRate`). A future ACP can revisit this if needed.

### Why Stake-Weighted Median

The median is robust to outliers: a small number of validators cannot push the parameter to an extreme value. Stake-weighting ensures that the economic majority (by capital committed) drives the outcome.

## Backwards Compatibility

### Breaking Changes

This ACP modifies the P-Chain reward calculation, which is a **consensus-critical** change. All Avalanche Network Clients must implement the updated formula to remain in consensus after activation.

### What Does Not Change

- Transaction formats on the P-Chain are unaffected.
- Staking transaction types (`AddValidatorTx`, `AddDelegatorTx`, etc.) are unchanged.
- Existing staking periods in progress at activation time complete under the **new formula** (see Migration below).
- `MaximumSupply` is unchanged.
- `MaxConsumptionRate` is unchanged.

### Migration / Rollout

- **In-progress staking periods:** Rewards for validators and delegators whose staking period spans the activation boundary are calculated using the new formula for the entire period. Because the default values at activation are current-equivalent (α = 1.00, `MinConsumptionRate` = 0.10), there is no reward discontinuity at activation.
- **No state migration required.** The change is purely computational — no P-Chain state schema changes are needed.

## Reference Implementation

### Code Areas Affected

1. **Reward calculator** (`vms/platformvm/reward/calculator.go` in AvalancheGo): Multiply `PotentialReward` by α. Add α as a parameter input alongside existing consumption rate parameters.
2. **Network parameter storage**: Add α as a tracked network parameter (similar to how `MinConsumptionRate` and `MaxConsumptionRate` are stored). Enforce bounds [0.30, 1.00] at the protocol level.
3. **Governance signal processing**: Extend the validator signaling mechanism to accept α and `MinConsumptionRate` preferences. Implement stake-weighted median aggregation and rate limiting.
4. **Configuration**: Add AvalancheGo config flags for validators to express governance preferences (e.g., `--staking-reward-alpha`, `--staking-min-consumption-rate`).
5. **API surface**: Expose current effective α and `MinConsumptionRate` via the `platform.getStakingParameters` or equivalent API endpoint.

### Test Expectations

- At activation defaults (α = 1.00, `MinConsumptionRate` = 0.10), reward outputs must be identical to the current implementation (bit-for-bit).
- At α = 0.30, `MinConsumptionRate` = 0.05: verify 1-year APR ≈ 1.94%, 2-week APR ≈ 0.85%.
- Bounds enforcement: attempts to set α < 0.30 or > 1.00 must be rejected.
- Rate limiting: parameter changes exceeding ±5% per epoch must be clamped.
- Quorum: if < 80% of stake signals, parameter must not change.

## Security Considerations

- **Griefing via extreme parameter votes.** Mitigated by protocol-enforced bounds and the ±5% per-epoch rate limit. Even if a majority of stake colludes to minimize rewards, α cannot go below 0.30.
- **Validator incentive misalignment.** Large validators may prefer lower α to suppress competition from smaller validators who are more yield-sensitive. The stake-weighted median makes this difficult unless a supermajority of stake coordinates.
- **Reward predictability for delegators.** Dynamic parameters mean that expected yields at delegation time may differ from realized yields. Staking UIs should display the current parameters and their recent trajectory. This is an informational concern, not a security issue.

## Economic / Tokenomics Considerations

- **Emission reduction.** At α = 0.30, annual emissions from staking rewards drop by ~70% relative to current rates. This slows the approach to `MaximumSupply` and may be desirable if the community judges current emission too high relative to network usage fees.
- **Staking participation.** Lower rewards may reduce staking participation. The α floor of 0.30 (≈2% 1-year APR) is designed to remain competitive with risk-free-rate proxies. Monitoring staked percentage post-activation is critical.
- **Duration incentive.** At `MinConsumptionRate` = 0.05, short-duration stakers earn ~44% of the long-duration rate. This creates meaningful economic pressure toward longer commitments, potentially reducing validator churn and improving network stability.

## Deployment Plan

### Activation

This change activates as part of **Helicon** network upgrade. The upgrade includes:

1. The modified reward formula (multiplication by α).
2. New bounds for `MinConsumptionRate`.
3. Governance signaling infrastructure for α and `MinConsumptionRate`.

## Open Questions

1. **Epoch length.** The suggested epoch of ~8,192 P-Chain blocks (~3 hours) is a placeholder. What is the optimal frequency for parameter recalculation given the tradeoff between responsiveness and stability?

2. **Rate limit calibration.** Is ±5% per epoch the right maximum change rate? Too tight, and governance becomes sluggish. Too loose, and reward volatility increases.

3. **Quorum threshold.** Is 80% of active stake the right threshold for requiring signaling before parameters can change? Lower thresholds increase responsiveness but reduce the bar for coordinated manipulation.

4. **In-progress staking period handling.** Should validators/delegators who began staking before a parameter change receive rewards calculated at the parameters in effect at their start time, or at the parameters in effect at reward distribution time? This ACP specifies the latter (reward-time parameters). The alternative provides more predictability but adds implementation complexity (snapshotting parameters per staking period).

5. **Interaction with future staking changes.** If minimum staking duration is reduced to 2 days (as proposed in [ACP-273](https://github.com/avalanche-foundation/ACPs/pull/273)), does this ACP's specification require any amendment? (Analysis suggests no, see [2-Day Minimum Staking Duration](#2-day-minimum-staking-duration).)

6. **Fixed-point precision for α.** This ACP proposes `uint64` scaled by 10⁴. Is this sufficient precision, or should a higher denominator (e.g., 10⁶) be used to allow finer-grained adjustments?

7. **Validator UX for governance signaling.** What is the minimum viable UX for validators to set and update their parameter preferences? Should this be a CLI flag, a config file entry, or an on-chain transaction?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
