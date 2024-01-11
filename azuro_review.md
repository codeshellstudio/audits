# Azuro - Security Review by CodeShell

## About the report

This report is the result of the security assessment made by Codeshell for Azuro.

Dates: 
04 December 2023 - 18 December 2023

Azuro is a betting protocol that enables peer-to-pool betting without intermediaries. It's built on EVM-compatible blockchains and consists of several smart contracts that handle different aspects of the betting process, such as liquidity pools, betting conditions, live betting, and payouts. 
https://azuro.org/
https://gem.azuro.org/concepts/overview

CodeShell is a team of blockchain security experts.

## Scope

https://github.com/Azuro-protocol/Azuro-v2

The following smart contracts were in the scope of the audit:

```
Off-chain live betting contracts scope:
- contracts/ClientCoreBase.sol 
- contracts/HostCore.sol 
- contracts/LiveCore.sol
- contracts/Relayer.sol
- contracts/interface/IClientCondition.sol
- contracts/interface/IClientCoreBase.sol 
- contracts/interface/IConditionState.sol
- contracts/interface/IHostCondition.sol 
- contracts/interface/IHostCore.sol 
- contracts/interface/ILiveCore.sol
- contracts/interface/ILP.sol

Vault contracts scope:
- contracts/Vault.sol
- contracts/LP.sol
- contracts/Factory.sol
- contracts/utils/LiquidityTree.sol
- contracts/interface/ILiquidityTree.sol
- contracts/interface/IVault.sol
```

audit commit hash:
- [813b4aa0afbfd4b5b73c539b336291fdd13b229a](https://github.com/Azuro-protocol/Azuro-v2/commit/813b4aa0afbfd4b5b73c539b336291fdd13b229a)

reaudit commit hash:
- [f541648c1eb786c8a451fe32c230f24be32930ba](https://github.com/Azuro-protocol/Azuro-v2/commit/f541648c1eb786c8a451fe32c230f24be32930ba)


---

## Findings


The following number of issues were found:

- Critical: no issues
- High: no issues
- Medium: 2 issues
- Low: 4 issues

|  Numeration     | Title       | Severity      | status |
| ------ | ---------------- | ------------- | ---
| [M-01] | Relayer approval lost if LP is changed | Medium | Fixed
| [M-02] | Winning outcomes is not checked for duplicates when resolving | Medium | Acknowledged
| [L-01] | winningOutcomesCount should be below or equal to conditionData.outcomes | Low | Fixed
| [L-02] | conditionData.outcomes is not checked for duplicates when creating a condition | Low | Fixed
| [L-03] | minOdds duplicate in betData | Low | Acknowledged
| [L-04] | Sum of fees of 100% is likely too much | Low | Acknowledged


## [M-01] Relayer approval lost if LP is changed

### Description

When `Relayer` is initialized it gives the following approvals:
```
        // Max approve LP spending
        TransferHelper.safeApprove(lp.token(), address(lp), type(uint256).max);
```
This approval is required to call `lp.betFor()`.
But LP can be changed:
```
    function changeLP(address newLP) external onlyOwner {
        lp = ILP(newLP);
        emit LPChanged(newLP);
    }
```
If it happens, the Relayer is left with no approval for the new LP. Thus, `betFor()` will not work.


### Recommendations

When changing LP make sure that:
- new approval is given
- previous approval is revoked

---

## [M-02] Winning outcomes is not checked for duplicates when resolving

### Description

`LiveCore.resolveConditions()` introduces `winningOutcomes`.
It only checks the length (which should be equal to `condition.winningOutcomesCount`)
But having duplicates among `winningOutcomes` is also a bad scenario.

The updated `payout` will not be correct as it would sum up `condition.payouts[]` for the same indexes.
https://github.com/Azuro-protocol/Azuro-v2/blob/200ebfec987201cfdfe3fc3e8672ba78d3e58c7e/contracts/LiveCore.sol#L65-L72

### Recommendations

Consider looking for duplicates in `winningOutcomes` and revert if found any

---

## [L-01] winningOutcomesCount should be below or equal to conditionData.outcomes

### Description

`_createCondition()` registers a condition, but does not check that:
`conditionData.winningOutcomesCount <= conditionData.outcomes`
If it is not true, it will not be possible to provide the correct `winningOutcomes` in the future.

### Recommendations

Consider making this additional check.

---

## [L-02] conditionData.outcomes is not checked for duplicates when creating a condition

### Description

When LiveCore creates conditions it is expected that outcome Ids are unique, but it is not checked.
https://github.com/Azuro-protocol/Azuro-v2/blob/200ebfec987201cfdfe3fc3e8672ba78d3e58c7e/contracts/ClientCoreBase.sol#L189-L212

In case of duplicates provided in `conditionData.outcomes`, `outcomeNumbers` will write a duplicate outcomeId a few times, keeping in the storage only the last incremented index.

### Recommendations

Consider checking for duplicates of outcomes in `ClientCoreBase._createCondition()`.

---

## [L-03] minOdds duplicate in betData

### Description


During the bet operation the are two `minOdds` arguments in inputs.
The first one - `BetData.minOdds`
https://github.com/Azuro-protocol/Azuro-v2/blob/813b4aa0afbfd4b5b73c539b336291fdd13b229a/contracts/LiveCore.sol#L90C37-L90C37

The second one - inside `BetData.data` decoded to `OrderData.minOdds`
https://github.com/Azuro-protocol/Azuro-v2/blob/813b4aa0afbfd4b5b73c539b336291fdd13b229a/contracts/LiveCore.sol#L93

In fact, only the second one is taken in the following logic:
https://github.com/Azuro-protocol/Azuro-v2/blob/813b4aa0afbfd4b5b73c539b336291fdd13b229a/contracts/LiveCore.sol#L111

But inputted `BetData.minOdds` is not used.

### Recommendations

Consider removing the first input `BetData.minOdds`.
OR
If you wish minimal changes, you make additional checks, that `order.clientBetData.minOdds == betData.minOdds`
OR
If `betData.minOdds` will be left ignored - you can check that it is equal to zero

---

## [L-04] Sum of fees of 100% is likely too much

### Description

`LP._checkFee()` is invoked when fees are updated.

```
    function _checkFee() internal view {
        if (
            _getFee(FeeType.DAO) +
                _getFee(FeeType.DATA_PROVIDER) +
                _getFee(FeeType.AFFILIATES) >
            FixedMath.ONE
        ) revert IncorrectFee();
    }
```
The sum of three fees is compared with 100%. Probably, 100% is not a desired scenario.
Moreover, we expect that there is some expected number that looks realistic, far less than 100%.

### Recommendations

Consider introducing some value like `MAX_FEE` that you see as a reasonable maximum for the sum of all fees.


## Disclamer

The security review does not give any warranties on the security of the code. The security review is not investment advice.
