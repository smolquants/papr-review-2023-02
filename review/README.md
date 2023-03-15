# Review

The PAPR economic review is broken into two sections:

1. Oracle manipulation analysis in [oracle-manipulation.md](./oracle-manipulation.md)
2. Funding rate analysis in [funding-rates.md](./funding-rates.md)

The first section deals with cost of attack calculations for manipulating the underlying Uniswap V3 pool to
intentionally trigger liquidation auctions on PAPR.

The second section focuses on the current funding rate mechanisms implemented by the protocol. It dives deeper into
ways to mitigate issues related to competing with the ETH staking rate, similar to issues experienced by RAI.

Background information summarizing our understanding of the core protocol is in [background.md](./background.md).

## Approach

Our approach to understanding papr, and specificially what each PAPR token represents from a financial perspective,
was to examine the protocol under the context of no arbitrage arguments. Meaning, take the perspective of ETH
natives and assume these natives seek to earn returns on their ETH at or above the ETH staking rate: i.e. assume
the ETH staking rate is the "risk-free" rate for the ETH economy. Then, we ask ourselves

- What "arbitrage" strategies can these ETH natives deploy using papr vaults to earn risk-free yield in excess of the risk-free rate?
- Under what conditions would these arbitrage premia on the risk-free rate go to zero?

The latter conditions enforce no-arbitrage and provide guidance for what one should expect the funding rates on each
PAPR token should trend toward over time, once markets become "efficient".

