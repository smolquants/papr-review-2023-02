# Background

Background notes on [PAPR](https://papr.wtf).

## Summary

PAPR seems like a quanto version of RAI: three tokens involved in the loan.
  - Debt token: PAPR
  - Collateral token: NFT
  - Quote token: USDC or ETH

Where the collateral is priced in quote, debt remains constant in PAPR terms, and interest payments come through funding in kind (Squeeth like)
approach -- Target for the debt token price relative to quote token gets adjusted up and down such that the debt each loan owes in quote token terms
increases/decreases based on prevailing demand for leverage/loans.

Target mechanism akin to funding rate based on price of debt token PAPR relative to quote token. Analogous to RAI interest rate mechanism on loans as
dampens any change in demand/supply for leverage. Meaning:
  - if Mark < Target, more debtors than previously had
    - Protocol increases "interest rate" by log difference in mark v.s. target price for PAPR vs quote to account for deviation in mark away from target.
      Makes the debt borrowers take on (or have taken on) more expensive in quote terms. Incentivizes borrowers that took out leverage on collateral to repay loans
    - Debt holders (i.e. passive holders of PAPR) rewarded higher "interest payments" through increase in target price 
  - if Mark > Target, less debtors than previously had
    - Protocol decreases "interest rate", making debt borrowers take on (or have taken on) less expensive in quote terms. Incentivizes borrowers to take out more loans.

What makes this super interesting is both the quanto nature of the loans AND that the debt token PAPR looks like something close to a perpetual bond, with holders
earning "interest payments" through changes in the target rate. So holding PAPR is effectively speculating on the anticipated perpetual "coupon" payment stream protocol
earns from borrowers taking out loans -- similar type of asset to stETH (bond) but without the rebase.

Makes for really interesting exercise in attempting to model the value of PAPR at any given time. See [funding-rates.md](./funding-rates.md) for no-arbitrage arguments
supporting these initial takes on what each PAPR token is.
