---
aip: 13
title: Coin 标准改进
author: movekevin
discussions-to: https://github.com/aptos-foundation/AIPs/issues/24
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/01/05
updated: 2023/01/26
---

## 概要

自Aptos主网启动以来，社区提出了一些改进建议，将加速Coin标准的采用。具体包括：
1. 隐式币种注册：目前，新币种的接收方需要显式注册才能在第一次接收到它。这在用户体验上与交易所（包括CEX和DEX）以及钱包之间造成了摩擦。转换为隐式模型，无需注册将显著改善用户体验。
2. 增加对批量转账的支持：这是一个次要的便利性改进，允许账户在单个交易中发送代币，包括网络代币（APT）和其他代币。

由于（2）是一个次要的改进，本提案的其余部分将讨论（1）的细节。

## 动机和原理

目前，在许多情况下，代币（包括APT在内的ERC-20代币）不能直接发送到一个账户，如果：
- 该账户尚未创建。需要由另一个可以支付gas费用的账户调用`Aptos_account::transfer` 或 `account::create_account` 。这通常是一个轻微的烦恼，但不是太大的问题。
- 该账户尚未注册以接收该币种。对于每个自定义代币都必须完成此操作（创建账户时默认注册了APT）。这是投诉/痛苦的主要原因。

这种设计的主要历史原因是让账户明确选择他们想要的代币，而不是接收他们不想要的随机代币。然而，这导致了用户和开发者的困扰，因为他们需要记住注册币种，尤其是当只有接收方账户才能执行此操作。一个重要的用例遇到了这个问题，那就是涉及自定义代币的CEX转账。

## 提议

我们可以切换到一种模式，在特定的CoinType没有CoinStore存在时，如果转移，则隐式创建CoinStore（通过注册）。这可以作为aptos_coin中的一个单独流程添加，类似于`aptos_coin::transfer`，如果一个APT转移不存在则隐式创建一个账户。此外，账户可以选择退出此行为（例如，为了避免接收垃圾代币）。详细流程如下：
1. `aptos_coin::transfer_coins<CoinType>(from: &signer, to: address, amount: u64)`  默认情况下，如果 CoinType 不存在，则将注册接收方地址（创建CoinStore）。
2. 账户可以选择退出（例如，为了避免接收垃圾代币），方法是调用 `aptos_account::set_allow_direct_coin_transfers(false)` 。他们也可以稍后使用 `set_allow_direct_coin_transfers(true)` 来恢复。默认情况下，在此提案实施之前的所有现有账户和此后的新账户都将被隐式选择接收所有代币。

## 实施
参考实现：
- 隐式币种注册：https://github.com/aptos-labs/aptos-core/pull/5871
- 批量转账支持：https://github.com/aptos-labs/aptos-core/pull/5895

## 风险和缺点
由于这是一种新的流程，而不是修改现有的 `coin::transfer` 函数，因此对 `coin::register` 和 `coin::transfer` 的现有依赖不应受到影响。只有一个已知的潜在风险：由于账户默认选择接收任意代币，因此它们可能会受到恶意用户发送的数百或数千个垃圾代币的骚扰。可以采取以下措施进行缓解：
- 钱包可以维护一个已知的信誉币种列表，并使用它来过滤用户的币种列表。这是其他链/钱包中常见的标准用户体验。
- 资源API目前允许分页，这有助于当账户有太多资源时。这可以减轻通过创建太多资源（每个任意代币一个CoinStore）“DDOS”账户的风险。
- 索引系统可以进一步帮助过滤垃圾代币。这可以在其他链上的流行资源管理器（如Etherscan）中看到。

## 时间表
此更改可以在2月1日或2月8日（PST时间）的一周内推出到测试网进行测试。