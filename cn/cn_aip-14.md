---
aip: 14
title: 更新锁仓合约
author: movekevin
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/02/10
updated: 2023/02/10
---

## 动机

本提案旨在更新当前的锁定合约中的奖励分配逻辑：https://github.com/aptos-labs/aptos-core/blob/496a2ce5481360e555b670842ef63e6fcfbc7926/aptos-move/framework/aptos-framework/sources/vesting.move#L420

当前的奖励计算目前使用的是基础的  `staking_contract` 的记录本金，每次调用 `staking_contract::request_commission` 时都会更新。

## 提议

使用抵押合约的本金是间接的，锁定合约可以使用剩余的授予金额作为基础来获取到目前为止累积的实际奖励金额。以下是一个例子，以便更清晰地说明：

1. 锁定合约有100 APT和1个抵押参与者（为了简单起见）。100 APT也是剩余的金额。基础的抵押合约有100 APT的本金。
2. 抵押池赚取了50 APT。
3. 在计算属于锁定池的未结算奖励之前，, `vesting::unlock_rewards` 首先要求支付给运营商的佣金，即5 APT（50 APT收益的10%）。剩余的总活跃抵押金额为145 APT，全部属于锁定池。
4. 由于还没有代币被锁定，当前的 `, vesting::unlock_rewards ` 将计算出未结算奖励为145（抵押池中的总活跃金额） - 100（剩余授予金额）= 45，将分配给抵押合约参与者。

## 参考实现

https://github.com/aptos-labs/aptos-core/pull/6106

## 时间安排

目标测试网发布日期：2023年2月