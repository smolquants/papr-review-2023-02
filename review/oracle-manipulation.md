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


### Uniswap V2 Math



### Uniswap V3 Math

[Uniswap V3 Pools](https://uniswap.org/whitepaper-v3.pdf) are extensions of the core V2
pool to incorporate the notion of concentrated liquidity.




