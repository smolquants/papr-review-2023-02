# Funding Rates

Funding rate mechanism analysis.

## TL;DR

- Using ETH as the quote token for a given PAPR vault setup causes similar issues to what RAI is currently experiencing
- Meaning, from no arbitrage / interest rate parity arguments, should expect the funding rate on the associated PAPR vault
to likely trend negative as the "risk-free" rate the vault needs to compete with (and compare to) is the ETH staking rate
- Consider using wstETH or one of the other non-rebase LSDs as the quote token to target PAPR rates against. This would make PAPR funding rates
reflect the actual "premium" component (i.e. mark vs target price), since the "interest rate" component is built into the LSD token itself
- Interestingly, a simple replication strategy shows passively holding the PAPR token for an NFT collection appears equivalent to
playing the basis trade on a hypothetical perpetual market for that same NFT collection.


## No-Arbitrage Funding Rates

Should consider making the numeraire (i.e. quote token) wstETH such that each Uniswap pool querying mark from
has pairing wstETH vs PAPR, otherwise likely encounter similar issues to [RAI](https://community.reflexer.finance/t/can-oracles-double-as-co-stakers-how-rai-like-systems-might-safely-support-staked-eth/397).

Using wstETH as the quote in adjusting PAPR token targets would effectively have the funding rate mechanism on PAPR represent solely
the appetite for leverage against collateral types supported by PAPR. This is analogous to the "premium" component of the funding
rate on e.g. BitMEX perps. Bringing in wstETH as the quote to adjust targets against would be analogous to including the "interest rate"
component of the BitMEX funding rate, assuming for the ETH economy that staked ETH yields (consensus portion) represent the "risk-free"
rate (or as close as we'll get to one).

### ETH Collateral

Another way to look at this is from no-arbitrage arguments, using the super simplified case of using ETH as the collateral
users are borrowing PAPR against (i.e. replicating RAI). By no-arbitrage / [interest rate parity](https://en.wikipedia.org/wiki/Interest_rate_parity),
I could:

- Take out a loan by minting PAPR against my ETH
- Sell the PAPR for ETH on secondary (ignore slippage and trading fees)
- Stake the ETH to earn the "risk-free" rate

The yield on the *loan principle* for this strategy would be approximately the ETH staking rate minus the funding rate needed
to pay PAPR holders for the loan. But this strategy would represent earning a risk-free premium on top of the ETH staking rate
if PAPR funding != risk free rate (ignoring liquidations and collateral). So by no arb arguments, IF you ignore the capital locked
up in the vault to mint the loan, would expect the PAPR funding rate to be equal to the ETH staking rate on this vault.

*However*, as mentioned above, this completely ignores the ETH required to back the PAPR loan (overcollateralized). These same
borrowers employing this no-arb strategy are comparing all of their ETH capital locked up in this strategy relative to the same
amount of ETH capital simply staked earning the protocol risk-free rate. So when employing the PAPR no-arb strategy, they're really
*losing* yield from the ETH staking rate `r` on the capital needed to back the PAPR loan, and therefore need to compensate for that
lost yield with a higher yield on the loan principle itself. So really should expect the no-arb funding rate `f` to be satisfied by

```
0 = -r + LTV * (r - f)
```

or

```
f = r * (1 - 1 / LTV) 
```

where the overcollateralization ratio `LTV` really hurts here -- borrowers need to overcompensate for the lost risk free rate on the
ETH collateral used to back the loan, which means one should anticipate the funding rate to be persistently negative on ETH backed PAPR
as `LTV < 1`. In the PAPR case, implies `mark > target` consistently, which is bad for PAPR token holders; i.e. not enough demand for leverage
given the interest rate parity arguments above.

### NFT collateral

You can apply the same logic to the PAPR token loans backed by NFT collateral to find a similar strategy that gives a rough estimate for
the no-arb funding rate IF you assume the existence of an NFT perpetual that users of PAPR can short to hedge their exposure to the NFT
collateral locked in PAPR vaults. No arb strategy would be:

- Buy the NFT collateral with your ETH
- Sell the perp on the collateral to hedge NFT exposure
- Mint PAPR using NFT collateral
- Sell PAPR for ETH on secondary
- Stake ETH to earn "risk-free" rate

The yield for this strategy *relative to the risk free ETH rate* is:

```
y = (-r + f_perp) + LTV * (r - f_papr)
```

where `f_perp > 0` would mean getting paid funding to short the perp.

No arb means this `y` should go to zero over time, so would expect the PAPR funding rate to be

```
f_papr = f_perp / LTV + r * (1 - 1/LTV)
```

This is only > 0 if the hypothetical `f_perp` funding rate short hedge is getting paid out greater than `r * (1 - LTV)`. I would expect this to be
the case most of the time, particularly in a bull market, but when everyone's bearish NFTs (or mildly bearish/complacent for that matter), this def
won't be. Which means PAPR holders backed by these NFTs should expect negative funding rates in these scenarios and are less likely to hold their PAPR
tokens.

This is essentially the same reason (but for NFT collateral) to consider adding the "interest rate" premium into the PAPR mechanism funding rate by
using wstETH or one of its non-rebase LSD equivalents as the quote token in the underlying PAPR liquidity pool on Uniswap, as the funding rate would then
implicitly incorporate the risk free rate in the no arb strategy above.

Notice, if `f_perp` also incorporates the interest rate component, then when using the LSD, should expect PAPR funding rates to
be the funding rate on the perp adjusted for risk-free (i.e. appetite for NFT leverage thru the perp) divided by `LTV`,
since the no arb strategy would produce

```
f'_papr = (f_perp - r) / LTV
        = f'_perp / LTV
```

where `f = f' + r` in both PAPR and perp cases. `f'` can be interpreted as the mark "premium" component of the funding rate and `r` the interest
rate component when using the BitMEX analogy.

Interestingly, passively holding the PAPR token for an NFT collection appears equivalent to playing the basis trade on a hypothetical perp (i.e. NFT/ETH perp market)
for that same NFT collection, through the replication strategy above. Therefore, price appreciation on the respective PAPR token should produce
similar returns to that of the basis trade on a hypothetical NFT perp.
