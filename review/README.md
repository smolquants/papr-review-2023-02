# Review

## Intro

Papr is an innovative protocol that offers an avenue to access decentralized lending with illiquid collateral types, that likely
cannot be supported by existing lending protocols. If Papr scales successfully it means borrowers would be able to borrow against many more
collateral types in a decentralized manner and lenders have a separate avenue to earn sustainable yield via simply holding various
PAPR tokens.

SmolQuants is excited to present part 1 of our economic review of Papr below.


## Breakdown

The PAPR economic review is broken into two sections:

1. Funding rate analysis in [funding-rates.md](./funding-rates.md)
2. Oracle manipulation analysis in [oracle-manipulation.md](./oracle-manipulation.md)

The first section focuses on the current funding rate mechanisms implemented by the protocol. It dives deeper into
ways to mitigate issues related to competing with the ETH staking rate, similar to issues experienced by RAI.

The second section deals with cost of attack calculations for manipulating the underlying Uniswap V3 pool to
intentionally trigger liquidation auctions on PAPR.

Background information summarizing our understanding of the core protocol is in [background.md](./background.md).

## Approach

Our approach to understanding papr, and specifically what each PAPR token represents from a financial perspective,
was to examine the protocol under the context of no-arbitrage arguments. Meaning, take the perspective of an ETH
native and assume these natives seek to earn returns on their ETH at or above the ETH staking rate: i.e. assume
the ETH staking rate is the "risk-free" rate for the ETH economy. Then, we ask ourselves

- What "arbitrage" strategies can these ETH natives deploy using papr vaults to earn risk-free yield in excess of the risk-free rate?
- Under what conditions would these arbitrage premia on the risk-free rate go to zero?

The latter conditions enforce no-arbitrage and provide guidance for what one should expect the funding rates on each
PAPR token should trend toward over time, once markets become "efficient" and arbitrage premia trend to zero.

