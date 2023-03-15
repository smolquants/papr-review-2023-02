# Funding Rates

Funding rate mechanism analysis.

## TL;DR

- papr should be aware of the issues [RAI is currently experiencing](https://community.reflexer.finance/t/can-oracles-double-as-co-stakers-how-rai-like-systems-might-safely-support-staked-eth/397),
given papr uses similar funding rate mechanisms
- Meaning, from no arbitrage / interest rate parity arguments, should expect the funding rate on a hypothetical ETH-backed
papr vault to likely trend negative due to overcollateralization via *unstaked* ETH: i.e. there's an opportunity cost to using
ETH to overcollateralize the loans as the vault collateral will not earn the "risk-free" ETH staking rate
- The example of an unstaked ETH-backed papr vault gives a framework for analyzing all papr vaults: i.e. no-arbitrage arguments for what
to expect the funding rate should trend toward
- Interestingly, a simple replication strategy shows passively holding the papr token for a particular NFT collection appears equivalent to
playing the basis trade on a hypothetical perpetual market on that same NFT collection.


## No-Arbitrage Funding Rates

### ETH Collateral

One way to look at the papr funding rate mechanism (and expectations for what the rate might be) would be to apply no-arbitrage arguments to the system.
Take the super simplified case of ETH as the collateral users are borrowing PAPR against (i.e. replicating RAI).
By no-arbitrage / [interest rate parity](https://en.wikipedia.org/wiki/Interest_rate_parity), I could:

- Take out a loan by minting paprETH against my ETH
- Sell the paprETH for ETH on secondary (ignore slippage and trading fees)
- Stake the ETH to earn the "risk-free" rate

The yield on the *loan principle* for this strategy would be approximately the ETH staking rate minus the funding rate needed
to pay papr holders for the loan. But this strategy would represent earning a risk-free premium on top of the ETH staking rate
if papr funding != risk free rate (ignoring liquidations and collateral). So by no arb arguments, IF you ignore the capital locked
up in the vault to mint the loan, would expect the papr funding rate to be equal to the ETH staking rate on this vault.

*However*, as mentioned above, this completely ignores the ETH required to back the papr loan (overcollateralized). These same
borrowers employing this no-arb strategy are comparing all of their ETH capital locked up in this strategy relative to the same
amount of ETH capital simply staked earning the protocol risk-free rate, assuming for the ETH economy that staked ETH yields (consensus portion)
represent the "risk-free" rate (or as close as we'll get to one). So when employing the papr no-arb strategy, they're really
*losing* yield from the ETH staking rate `r` on the capital needed to back the papr loan, and therefore need to compensate for that
lost yield with a higher yield on the loan principle itself. So really should expect the no-arb funding rate `f` to be satisfied by

```
0 = -r + LTV * (r - f)
```

to first-order or

```
f = r * (1 - 1 / LTV) 
```

where the overcollateralization ratio `LTV` really hurts here -- borrowers need to overcompensate for the lost risk free rate on the
ETH collateral used to back the loan, which means one should anticipate the funding rate to be persistently negative on ETH-backed papr
as `LTV < 1`. In the papr case, implies `mark > target` consistently, which is bad for papr token holders; i.e. not enough demand for leverage
given the interest rate parity arguments above.

### NFT collateral

You can apply the same logic to the papr token loans backed by NFT collateral to find a similar strategy that gives a rough estimate for
the no-arb funding rate IF you assume the existence of an NFT perpetual that users of papr can short to hedge their exposure to the NFT
collateral locked in papr vaults. No arb strategy would be:

- Buy the NFT collateral with your ETH
- Sell the perp on the collateral to hedge NFT exposure
- Mint paprNFT using NFT collateral
- Sell paprNFT for ETH on secondary
- Stake ETH to earn "risk-free" rate

The yield for this strategy relative to the risk free ETH rate is to first-order:

```
y = (-r + f_perp) + LTV * (r - f_papr)
```

where `f_perp > 0` would mean getting paid funding to short the perp.

No arb means this `y` should go to zero over time, so would expect the NFT-backed papr funding rate to be

```
f_papr = f_perp / LTV + r * (1 - 1/LTV)
```

This is only > 0 if the hypothetical `f_perp` funding rate short hedge is getting paid out greater than `r * (1 - LTV)`. I would expect this to be
the case most of the time, particularly in a bull market, but when everyone's bearish NFTs (or mildly bearish/complacent for that matter), this def
won't be. Which means papr holders backed by these NFTs should expect negative funding rates in these scenarios and are less likely to hold their papr
tokens.

Notice, if `f_perp` also incorporates the interest rate component, then when using a non-rebasing LSD as the quote for targeting (instead of ETH),
should expect papr funding rates to be the funding rate on the perp adjusted for risk-free (i.e. appetite for NFT leverage thru the perp) divided by `LTV`,
since the no arb strategy would produce

```
f'_papr = (f_perp - r) / LTV
        = f'_perp / LTV
```

where `f = f' + r` in both papr and perp cases. `f'` can be interpreted as the mark "premium" component of the funding rate and `r` the "interest rate"
component when using the [BitMEX analogy](https://www.bitmex.com/app/perpetualContractsGuide#Funding-Rate-Calculations).

Interestingly, passively holding the papr token for an NFT collection appears equivalent to playing the basis trade on a hypothetical perp (i.e. NFT/ETH perp market)
for that same NFT collection, through the replication strategy above. Therefore, price appreciation on the respective papr token should produce
similar returns to that of the basis trade on a hypothetical NFT perp.
