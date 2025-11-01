### Incorrect State upddating in adjust Maker leads stale liquidity value
**Description -** The protocol fee calculation mechanism relies on the value of the aaset.liq to determine user share of the accrued fee. however after a liquidity adjustment via adjustMaker(), the asset.liq storage variable is not updated to reflect the new liquidity, which result in all subsequent fee calculations using the outdated liquidity value leading to incorrect fee payout and accounting issues. In the adjustMaker(), the asst.liq is not updated with the new value, and user liquidity is passed into this struct and used throughout the tree walk to calculate the user shares of fees. This situation can lead to both over and underpayment.
https://github.com/sherlock-audit/2025-09-ammplify/blob/main/Ammplify/src/facets/Maker.sol#L55-L97

**Impact -** Under and over payment of the fees to the user.

**Tool Used -** Mannula Review

**Recommendation -** Update the asset.liq with the new adjusted liquidity.