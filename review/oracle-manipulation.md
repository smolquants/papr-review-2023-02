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


### Uniswap V3 Liquidity Math

Between ticks [Uni V3](https://uniswap.org/whitepaper-v3.pdf) obeys the $x \cdot y = L^2$ invariant, but introduces the concept of virtual liquidity
to enable liquidity provision over a finite price range chosen by the LP. The real reserves $(x, y)$ within a given price range $[p_a, p_b]$ follow

```math
(x + \frac{L}{\sqrt{p_b}}) \cdot (y + L \sqrt{p_a}) = L^2
```

where $x_v = x + \frac{L}{\sqrt{p_b}}$, $y_v = y + L \sqrt{p_a}$ are the virtual reserves (following the original V2 invariant) at the
current price $P = \frac{y_v}{x_v}$. V3 reduces to V2 for an LP range covering the entire price range: i.e. $p_a \to 0$, $p_b \to \infty$.


