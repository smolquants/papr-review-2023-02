# Oracle Manipulation

Oracle manipulation analysis.

## TL;DR


## Manipulating the Uniswap Pool to Trigger Liquidations

The attack is relatively simple and straightforward. The goal is to move the mark price on the Uniswap pool
enough to alter the PAPR controller target price such that the vault closest to liquidation gets pushed past the max LTV
allowed by the protocol, enabling liquidation of the vault. As the controller considers
all debt *in PAPR terms* and uses the *target price* (i.e. internal controller price for the PAPR token) to
convert from quote to PAPR terms when calculating collateral value backing the loan.

### PAPR Math

Specifically, the PAPR vault verifies whether a vault is liquidatable prior to triggering the liquidation auction.
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
which the controller will then attempt to counter with significantly higher targets. Note, selling PAPR is a
relatively simple task as one could mint PAPR by taking out an overcollateralized loan from the protocol, then intentionally
dump the PAPR on the Uniswap pool.

Solving for the mark price that triggers a liquidation:

```math
M_{liq} = R(t - \Delta t) \cdot \bigg[ \frac{R(t - \Delta t) \cdot D}{C(t) \cdot \mathrm{LTV}_{max}} \bigg]^{F / \Delta t}
```

where $M(t) < M_{liq}$ satisfies the liquidation condition. This can be simplified to approximately

```math
M_{liq} \approx R(t - \Delta t) \cdot \bigg[ \frac{\mathrm{LTV}(t - \Delta t)}{\mathrm{LTV}_{max}} \bigg]^{F / \Delta t}
```

when assuming the collateral value in quote terms is approximately the same since the last funding update: $C(t) \approx C(t-\Delta t)$.


### Uniswap V3 TWAP Math

The PAPR controller [uses](https://github.com/with-backed/papr/blob/master/src/UniswapOracleFundingRateController.sol#L144) the Uniswap V3
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

Inverting for the relative difference the attacker needs to attain in terms of the mark price required from the PAPR calc:

```math
P_t = P_{t-\Delta t, t-\beta} \cdot \bigg(\frac{M(t)}{P_{t - \Delta t, t - \beta}} \bigg)^{\Delta t / \beta}
```

Taking $M(t) = M_{liq}$ in this expression gives the spot price required to liquidate the PAPR vault

```math
P_{liq} \approx P_{t-\Delta t, t-\beta} \cdot \bigg( \frac{R(t-\Delta t)}{P_{t-\Delta t, t-\beta}} \bigg)^{\Delta t / \beta} \cdot \bigg[ \frac{\mathrm{LTV}(t - \Delta t)}{\mathrm{LTV}_{max}} \bigg]^{F/\beta}
```

Notice the spot liquidation price has dependence on the LTV ratio to the power of $F / \beta$, which is great from a manipulation standpoint
as the PAPR controller can tune $F$ to as large as necessary to reduce manipulability (but the tradeoff is less sensitivity for changes in funding).
One does, however, need to worry about the ratio of target to prior TWAP value being raised to $\Delta t / \beta$, since as $\Delta t \to F$ in
cases when `target > mark`, the liquidation spot price becomes less extreme for the manipulator.

To avoid the latter issues, would consider making cron-like calls to the PAPR controller's `updateTarget()` function to keep $\Delta t$ relatively small
for target adjustments -- this issue is due to the fact that the PAPR controller update process is *separate* from the Uni V3 TWAP update
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
move the price below $P_{liq}$, where $\Delta x$ can be minted via an overcollateralized loan from PAPR. Within the tick range, the amount of real $x$
reserves left for the LP position as a function of current price $p$ is:

```math
x(p) = L \cdot \bigg[ \frac{1}{\sqrt{p}} - \frac{1}{\sqrt{p_b}} \bigg]
```

Therefore, to get through the *first* tick range $[p_{a_0}, p_{b_0}]$, the attacker must send

```math
\Delta x_{0 \to a_0} = L_{a_0, b_0} \cdot \bigg[ \frac{1}{\sqrt{p_{a_0}}} - \frac{1}{\sqrt{p_0}} \bigg]
```

taking the start price for the entire pool to be $p_0$. To get through each subsequent price range $[p_{a_i}, p_{b_i}]$ with liquidity $L_{a_i, b_i}$
on the way to a final spot price $p_f$, the attacker must sweep through providing additional capital

```math
\Delta x_{b_i \to a_i} = L_{a_i, b_i} \cdot \bigg[ \frac{1}{\sqrt{p_{a_i}}} - \frac{1}{\sqrt{p_{b_i}}} \bigg]
```

along each range step. To push through the final range to the ultimate price desired,

```math
\Delta x_{b_n \to f} = L_{a_n, b_n} \cdot \bigg[ \frac{1}{\sqrt{p_f}} - \frac{1}{\sqrt{p_{b_n}}} \bigg]
```

Total capital needed to sell into the pool to reach $p_f$ from a start of $p_0$ is simply the sum of all these terms

```math
\Delta x = \Delta x_{0 \to a_0} + \sum_{i=1}^{n-1} \Delta x_{b_i \to a_i} + \Delta x_{b_n \to f}  
```

where $i \in \[1, n-1\]$ is simply a counter for all the tick ranges in between.


