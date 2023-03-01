# Oracle Manipulation

Oracle manipulation analysis.

## TL;DR


## Manipulating the Uniswap Pool to Trigger Liquidations

The attack is relatively simple and straightforward. The goal is to move the mark price on the Uniswap pool
enough to alter the PAPR controller target price such that the vault closest to liquidation gets pushed past the max LTV
allowed by the protocol, enabling liquidation of the vault. As the controller considers
all debt *in PAPR terms* and uses the *target price* (i.e. internal controller price for the PAPR token) to
convert from quote to PAPR terms when calculating collateral value backing the loan.

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

If we define the ratio of LTVs in the form

```math
\frac{\mathrm{LTV}(t-\Delta t)}{\mathrm{LTV}_{max}} = e^{- l(t-\Delta t)}
```

where $l(t)$ is the log difference in max vs vault LTV at time $t$, we get a nice Taylor series expansion to first order

```math
M_{liq} \approx R(t - \Delta t) \cdot \bigg[ 1 - \frac{F}{\Delta t} \cdot l(t - \Delta t) + \ldots \bigg]
```

to work with. Also gives us some more clarity on the effect differences in LTV have on the mark liquidation price. Further,
on the importance of having significantly longer funding periods vs funding update intervals $F \gg \Delta t$, as this forces
the attack to reach a much smaller price to trigger liquidations.

Given changes in mark price do *not* trigger updates to the target price on the PAPR controller, the protocol is taking the
risk that users will interact with the controller frequently (relative to $F$ time period). While currently $F = 90$ days does seem
relatively safe, the protocol should aim to trigger updates frequently (even if via cron-like calls to the controller)
to avoid a situation where e.g. the controller for a PAPR token hasn't been called in 10+ days. Otherwise, in the 10+ day example,
an attacker would only need to decrease mark by ~36% to trigger on a near-liquidation vault that has LTV / LTV_max = 96%, which seems plausible
for the only [$50K of liquidity](https://papr.wtf/tokens/paprMeme/lp) currently in paprMEME.


### Uniswap V3 Math

The PAPR controller [uses](https://github.com/with-backed/papr/blob/master/src/UniswapOracleFundingRateController.sol#L144) the Uniswap V3
geometric mean time-weighted average price over the time since the last call to `updateTarget()` as the mark price $M(t)$ in the target calculation
from the prior section. Using the geometric mean TWAP makes the mark price more difficult to manipulate, requiring a much lower
instantaneous spot price to be reached when the attacker sells PAPR to the Uniswap pool.

The mark price used is given by the expression for the geometric TWAP:

```math
M(t) &=& P_{t-\Delta t, t}
     &=& \bigg( \prod_{i = t - \Delta t}^{t} P_i )^{1/\Delta t}   
```



