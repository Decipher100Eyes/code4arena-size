# Audit Report: Liquidator cannot achieve proper liquidator reward as protocol promised

## Root cause and summary

In executeLiquidate, if liquidation is profitable, liquidatorReward is assignedCollateral(collateral amount assigned to target debtPosition) - debtInCollateralToken(debt position futureValue as collateral token).
This calculated reward capped to liquidationRewardPercent(5%) on future value of debt position with below code.
https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L96-L99

However, 5% of debt position future value is not converted to collateral token amount.
In the current code, the future value of debtPosition is unit of USDC but calculated reward should be unit of CollateralToken(ETH).
The decimal of USDC is 1e6, and the decimal of ETH is 1e18. In this wrong comparison, most of time liquidation reward should be miscalculated in favor against liquidator.
The future value (decimal of 6) will always be less than collateral token (decilmal of 18).
When it adds liquidate reward to liquidator at the end of function, liquidatorReward is considered as ETH unit, as if the comparison was correct.
So the liquidator receives much less liquidation reward than that of the procotocl promised.

## Proof of Concept

Actually, test code for this section exists and it is passed, as the test code is wrongly designed.
https://github.com/code-423n4/2024-06-size/blob/main/test/local/actions/Liquidate.t.sol#L163-L220
https://github.com/code-423n4/2024-06-size/blob/main/test/local/actions/Liquidate.t.sol#L184-L187

If using correctly converted one like below,

```solidity
uint256 debtInCollateralToken = size.debtTokenAmountToCollateralTokenAmount(futureValue);

uint256 liquidatorReward = Math.min(
  _state().bob.collateralTokenBalance - debtInCollateralToken,
Math.mulDivUp(debtInCollateralToken, size.feeConfig().liquidationRewardPercent, PERCENT)
);
```

It reverts at

```
assertEq(
  _after.liquidator.collateralTokenBalance,
  _before.liquidator.collateralTokenBalance + liquidatorProfitCollateralToken
);
```

## Recommended Mitigation Steps

The future value of the Debt position should be converted to collateral token unit when choosing proper liquidator reward.
As debtInCollateralToken already represents the value of debtposition in collateral token with proper decimal, it should be used instead of debtPosition.futureValue.

```
uint256 liquidatorReward = Math.min(
  assignedCollateral - debtInCollateralToken,
  Math.mulDivUp(debtInCollateralToken, state.feeConfig.liquidationRewardPercent,PERCENT)
);
```

## Tools Used

Foundry
