# Oracle Manipulation

Oracle manipulation analysis.

## TL;DR

- It is currently not possible within a "reasonable" timeframe to force a liquidation auction on the average paprMEME vault by manipulating the spot price on the paprMEME/WETH 1\% Uniswap V3 pool
- This is due to both the liquidation price being *outside* of the `MIN_TICK` price supported by the Uniswap pool implementation and the paprMEME `_targetMarkRatioMax` internal constant
being set well. Given current numbers, the attacker would need to wait for an instance when the `updateTarget()` function hasn't been called in $\Delta t > 44$ days
- Note that even if the attack were available, the attacker would still need to win the liquidation auction to make it worthwhile
- Still, the amount of capital required to force the Uniswap pool to the `MIN_TICK` is finite given the lack of full-range liquidity provision in the pool
- As backup protection against the extreme case where the cap can no longer protect paprMEME vaults from liquidation via manipulation, consider increasing the cost of attack to manipulate the underlying
paprMEME/WETH pool by incentivizing liquidity provision across the full tick range
- Should also consider making cron-like calls to the papr controller's `updateTarget()` function to keep $\Delta t$ relatively small, which helps eliminate the viability of this
manipulation attack given the controller-imposed `_targetMarkRatioMax` cap


## Manipulating the Uniswap Pool to Trigger Liquidations

The attack is relatively simple and straightforward. The goal is to move the mark price on the Uniswap pool
enough to alter the papr controller target price such that the vault closest to liquidation gets pushed past the max LTV
allowed by the protocol, enabling liquidation of the vault. As the controller considers
all debt in PAPR terms and uses the target price (i.e. internal controller price for the PAPR token) to
convert from quote to PAPR terms when calculating collateral value backing the loan.

### papr Math

Specifically, the papr vault verifies whether a vault is liquidatable prior to triggering the liquidation auction.
The [liquidation condition](https://github.com/with-backed/papr/blob/master/src/PaprController.sol#L337)
checks whether the debt in PAPR terms (static) exceeds the max debt allowed for the vault
given the current collateral value in quote and the target price set by the controller
(dynamic). The vault can be liquidated if:

```math
D > \frac{C(t) \cdot \mathrm{LTV}_{max}}{R(t)}
```

where $D$ is the vault's PAPR debt, $C(t)$ is the collateral value in quote terms, $\mathrm{LTV}_{max}$ is the
max loan to value param (50\% in practice) and $R(t)$ is the controller's current target price
for PAPR against the quote token. The target price the controller targets (and updates each write to controller)
follows the update equation

```math
R(t) = R(t - \Delta t) \cdot \bigg[ \frac{R(t-\Delta t)}{M(t)} \bigg]^{\Delta t / F}
```

where $\Delta t$ is the time elapsed since the last target update (i.e. funding payment),
$F$ is the funding period gov param (initially set to 90 days), and $M(t)$ is the mark price
from the Uniswap pool for the PAPR token vs quote.

Therefore, the attacker's goal would be to force the controller to target $R(t) \to \infty$ to make the collateral
worth significantly less in internal PAPR terms. To accomplish this, the attacker must sell PAPR into the pool so $M(t) \to 0$,
which the controller will then attempt to counter with significantly higher targets. With the caveat that the papr controller
imposes [bounds](https://github.com/with-backed/papr/blob/master/src/UniswapOracleFundingRateController.sol#L24) $B^{+}_{R/M}$ on the maximum
target-to-mark ratio it will acknowledge as valid. Selling PAPR is a relatively simple task as one could mint PAPR by taking
out an overcollateralized loan from the protocol, then intentionally dump the PAPR on the Uniswap pool.

Solving for the mark price that triggers a liquidation:

```math
M_{liq} = R(t - \Delta t) \cdot \bigg[ \frac{R(t - \Delta t) \cdot D}{C(t) \cdot \mathrm{LTV}_{max}} \bigg]^{F / \Delta t}
```

where $M(t) < M_{liq}$ satisfies the liquidation condition. This can be simplified to approximately

```math
M_{liq} \approx R(t - \Delta t) \cdot \bigg[ \frac{\mathrm{LTV}(t - \Delta t)}{\mathrm{LTV}_{max}} \bigg]^{F / \Delta t}
```

when assuming the collateral value in quote terms is approximately the same since the last funding update: $C(t) \approx C(t-\Delta t)$.
Liquidations via manipulation are not possible if $R(t - \Delta t) / M_{liq} > B^{+}_{R/M}$ due to the cap enforced by the papr controller.


### Uniswap V3 TWAP Math

The papr controller [uses](https://github.com/with-backed/papr/blob/master/src/UniswapOracleFundingRateController.sol#L144) the Uniswap V3
geometric mean time-weighted average price over the time since the last call to `updateTarget()` as the mark price $M(t)$ in the target calculation
from the prior section. Using the geometric mean TWAP makes the mark price more difficult to manipulate, requiring a much lower
instantaneous spot price to be reached when the attacker sells PAPR to the Uniswap pool.

The mark price used is given by the expression for the geometric TWAP:

```math
M(t) = P_{t-\Delta t, t} = \bigg( \prod_{i = t - \Delta t}^{t} P_i \bigg)^{1/\Delta t}   
```

Assume the attacker only manipulates over the last block in the TWAP calculation, with block times of $\beta$. Can rearrange the above
in terms of the TWAP from $\[t- \Delta t, t-\beta \]$ and the relative difference with this and the spot price $P_t$ realized post swap
over the last $\beta$ seconds of block time:

```math
M(t) = P_{t-\Delta t, t-\beta} \cdot \bigg( \frac{P_t}{P_{t-\Delta t, t-\beta}} \bigg)^{\beta / \Delta t}
```

Inverting for the relative difference the attacker needs to attain in terms of the mark price required from the papr calc:

```math
P_t = P_{t-\Delta t, t-\beta} \cdot \bigg(\frac{M(t)}{P_{t - \Delta t, t - \beta}} \bigg)^{\Delta t / \beta}
```

Taking $M(t) = M_{liq}$ in this expression gives the spot price required to liquidate the papr vault

```math
P_{liq} \approx P_{t-\Delta t, t-\beta} \cdot \bigg( \frac{R(t-\Delta t)}{P_{t-\Delta t, t-\beta}} \bigg)^{\Delta t / \beta} \cdot \bigg[ \frac{\mathrm{LTV}(t - \Delta t)}{\mathrm{LTV}_{max}} \bigg]^{F/\beta}
```

Notice the spot liquidation price has dependence on the LTV ratio to the power of $F / \beta$, which is great from a manipulation standpoint
as the papr controller can tune $F$ to as large as necessary to reduce manipulability (but the tradeoff is less sensitivity for changes in funding).
One does, however, need to worry about the ratio of target to prior TWAP value being raised to $\Delta t / \beta$, since as $\Delta t \to F$ in
cases when `target > mark`, the liquidation spot price becomes less extreme for the manipulator.

To avoid the latter issues, would consider making cron-like calls to the papr controller's `updateTarget()` function to keep $\Delta t$ relatively small
for target adjustments -- this issue is due to the fact that the papr controller update process is *separate* from the Uni V3 TWAP update
process (i.e. can't get triggered when users swap on Uniswap and move the mark).


### Uniswap V3 Liquidity Math

Between ticks [Uni V3](https://uniswap.org/whitepaper-v3.pdf) obeys the $x \cdot y = L^2$ invariant, but introduces the concept of virtual liquidity
to enable liquidity provision over a finite price range chosen by the LP. The real reserves $(x, y)$ within a given price range $[p_a, p_b]$ follow

```math
(x + \frac{L}{\sqrt{p_b}}) \cdot (y + L \sqrt{p_a}) = L^2
```

where $x_v = x + \frac{L}{\sqrt{p_b}}$, $y_v = y + L \sqrt{p_a}$ are the virtual reserves (following the original V2 invariant) at the
current price $p = \frac{y_v}{x_v}$. V3 reduces to V2 for an LP range covering the entire price range: i.e. $p_a \to 0$, $p_b \to \infty$.

Goal for the attacker is to calculate the amount of capital $\Delta x$ (i.e. quantity of PAPR token) required to send to the pool in order to
move the price below $P_{liq}$, where $\Delta x$ can be minted via an overcollateralized loan from papr. Within the tick range, the amount of real $x$
reserves left for the LP position as a function of current price $p$ is:

```math
x(p) = L \cdot \bigg[ \frac{1}{\sqrt{p}} - \frac{1}{\sqrt{p_b}} \bigg]
```

Therefore, to get through the first tick range $[p_{a_0}, p_{b_0}]$, the attacker must send

```math
\Delta x_{0 \to a_0} = L_{a_0, b_0} \cdot \bigg[ \frac{1}{\sqrt{p_{a_0}}} - \frac{1}{\sqrt{p_0}} \bigg]
```

taking the start price for the entire pool to be $p_0$. To get through each subsequent price range $[p_{a_i}, p_{b_i}]$ with
liquidity $L_{a_i, b_i}$ on the way to a final spot price $p_f$, the attacker must sweep through providing additional capital

```math
\Delta x_{b_i \to a_i} = L_{a_i, b_i} \cdot \bigg[ \frac{1}{\sqrt{p_{a_i}}} - \frac{1}{\sqrt{p_{b_i}}} \bigg]
```

along each range step. To push through the final range to the ultimate price desired,

```math
\Delta x_{b_n \to f} = L_{a_n, b_n} \cdot \bigg[ \frac{1}{\sqrt{p_f}} - \frac{1}{\sqrt{p_{b_n}}} \bigg]
```

Total capital needed to sell into the pool to reach $p_f$ from a start of $p_0$ is the sum of all these terms

```math
\Delta x = \Delta x_{0 \to a_0} + \sum_{i=1}^{n-1} \Delta x_{b_i \to a_i} + \Delta x_{b_n \to f}  
```

where $i \in \[1, n-1\]$ is a counter for all the tick ranges in between. $p_f = P_{liq}$ enables the attacker to trigger
the liquidation on the papr vault.

From a manipulation standpoint, the [downside to Uni V3 v.s. V2](https://cmichel.io/replaying-ethereum-hacks-rari-fuse-vusd-price-manipulation/)
is it can take a *finite* amount of capital to reach the minimum tick range for the pool (i.e. go to effectively 0), depending on
the existing liquidity distribution of the pool. Whereas V2 forces liquidity to be spread across the entire price range, V3 with
concentrated liquidity does *not* enforce this, although allows for it if an LP desires. By enforcing provision of liquidity over the
full price range, V2 requires an infinite amount of capital to reach a price of zero for the pool.

With regard to the oracle manipulation attack above, the finite capital required to reach a price of effectively zero on Uni V3 means that
the attacker does not necessarily have to worry about selling through intermediate liquidity in ranges down to $p_f$ *if* the liquidity LPs
are providing ends prior to $p_f$. Meaning, even if the $P_{liq}$ required to liquidate on papr is near zero (robust from papr mechanism standpoint),
the actual capital to get there could be significantly less than anticipated as the liquidity profile on the V3 PAPR pool could end far higher
than the liquidation spot price the attacker needs to reach. This effectively *increases* the liquidation price to

```math
P'_{liq} = \max (P_{liq}, p_{l})
```

where $p_l$ is the lowest price at which LPs are currently providing liquidity on the V3 pool, as long as $P_{liq}$ from papr is within
the price range supported by Uni V3 (i.e. above the min tick). As once $p_l$ is passed, the pool moves to the [tick range min](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L656).
LPs for the PAPR pool should consider replicating the robustness (to manipulation) of V2 by providing liquidity over the full tick range,
which forces the capital requirements for manipulating the pool to a price of zero to be infinite.


### Some Numbers

Referencing info on 2023-03-01 from [papr.wtf](https://papr.wtf):

| Total    | Amount (PAPR) |  Avg LTV  |   Max LTV   |  Last Update |  Last Target     |   Last Mark      |  Funding Period |  Target-to-Mark Bounds  |
| ------   | ------------- | --------- | ----------- | ------------ | ---------------  | ---------------- | --------------- | ----------------------  |
| 36 loans |   21.958      |  29.21\%  |   50\%      |   12 h ago   |  0.995 WETH/PAPR |  0.975 WETH/PAPR |   90 d          |       (0.5, 3.0)        |

Take block time to be 12 seconds. If an attacker were to manipulate the price only over 1 block (i.e. $\beta = 12$ s), the spot
liquidation tick they'd need to achieve would be -3481790298, which is less than the [`MIN_TICK`](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol#L9)
supported by Uni V3 (i.e. outside the Uni pool price range). Which means the attack is not possible, even though given the current
liquidity conditions, to get to the pool's min tick takes a finite amount of capital. The spot liquidation price only comes
within the Uniswap price range if spot remains at a `MIN_TICK` value for ~ 12 hours, ignoring the cap the controller places
on the target-to-mark ratio.

While this is a good thing in terms of the attack not being possible within a "reasonable" timeframe, it's not great that the inability to
perform the attack rests on an implementation detail for the Uniswap pool. Particularly, since it would be relatively cheap for the attacker
to push the price down to the minimum tick as the minimum price with liquidity on the pool $p_l$ is only 0.3329 ETH/PAPR and the pool
has a TVL of only $55.363K. An [oracle manipulation notebook](../notebook/oracle-manipulation.ipynb) is provided that can be rerun for
up to date numbers.

The cap $B^{+}_{R/M}$ on the target-to-mark ratio does a very good job at eliminating the viablity of this manipulation attack
for most "reasonable" time frames of $\Delta t$. For the current average LTV on PAPR at 90 day funding period, the attacker would have to
wait $\Delta t \approx 44$ days for the manipulation to even become possible. Assuming the shortest funding period allowed of
[28 days](https://github.com/with-backed/papr/blob/master/src/UniswapOracleFundingRateController.sol#L123), the attacker would
need to wait at least $\Delta t \approx 14$ days.
