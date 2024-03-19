---
aip: 12
title: 多签账户
author: movekevin
discussions-to: https://github.com/aptos-foundation/AIPs/issues/50
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/01/24
updated: 2023/01/26
---

## 摘要

本AIP提出了一种新的多签账户标准，主要由智能合约（multisig_account）中的透明数据结构和函数管理，比当前基于multied25519认证密钥的多签账户具有更便于使用和强大的功能。此外，对其作为Aptos更一般账户抽象的一部分进行演进的方向也很明确，以便用户管理其账户的更多类型和功能。

这不是一个完整的多签钱包产品，而是一个原始的构造，可能会有SDK支持，以便社区构建更强大的多签产品。

## 动机

多签账户在加密货币中很重要，用于：

- 作为 DAO 或开发者组的一部分，用于升级、操作和管理智能合约。
- 管理链上资金。
- 个人保护其自己的资产，以防一个密钥的丢失导致资金的丢失。

目前，Aptos支持multied25519认证密钥，允许多签交易：

- 这与多方签名交易不同，多方签名交易中，多个签名者分别对交易进行签名，导致执行交易时创建多个签名者。
- 可以通过调用 `create_account` 并传入正确的地址来创建多签账户，该地址是所有者公钥列表的哈希值，阈值k（k-of-n multisig）和 multied25519 方案标识符（1）串联而成。多签强制执行将通过多签账户的认证密钥完成。
- 要创建多签交易，需要传递tx有效载荷，并且多签账户的k个私钥需要用正确的[身份验证设置](https://aptos.dev/guides/creating-a-signed-transaction/#multisignature-transactions) (https://aptos.dev/guides/creating-a-signed-transaction/#multisignature-transactions)。
- 要添加或删除所有者或更改阈值，所有者需要发送具有足够签名的交易，以更改认证密钥以反映新的所有者公钥列表和新的阈值。

目前的多签设置存在一些问题，使其难以使用：

- 很难确定多签账户的当前所有者是谁，以及所需的签名阈值是多少。这些信息需要从认证密钥中手动解析。
- 要创建多签账户的认证密钥，用户需要串联所有者的公钥，并在末尾添加签名阈值。大多数人甚至不知道如何获取自己的公钥，也不知道它们与地址不同。
- 用户必须手动传递tx有效载荷以收集足够的签名。即使SDK简化了签名部分，存储和传递此tx仍然需要某个地方的数据库和一些协调工作以在有足够的签名时执行。
- 多签交易中的nonce必须是多签账户的nonce，而不是所有者账户的nonce。如果在执行之前执行了其他交易，这通常会使多签交易无效，增加nonce。管理这里的多个正在进行的tx的nonce可能会有些棘手。
- 添加或删除所有者不容易，因为涉及更改认证密钥。这种交易的有效载荷不容易理解，并且需要一些特殊逻辑来进行解析/差异化。

## 提议

我们可以创建一个更加用户友好的多签账户标准，生态系统可以在其上构建。这包括两个主要组成部分：

1. 一个多签账户模块，负责创建 / 管理多签账户和创建 / 批准 / 拒绝 / 执行多签账户交易。执行函数默认为私有，只能由以下执行：
2. 一种新的交易类型，允许执行者（必须是所有者之一）代表多签账户执行交易有效载荷。这将通过调用多签账户模块的私有执行函数进行认证。这种交易类型也可以泛化为支持其他模拟 / 委托用例，例如支付 Gas 费用以执行另一个账户的交易。

### 数据结构和多签账户模块

- 一个多签账户模块，允许更轻松地创建和操作多签账户
    - 多签账户将作为一个独立的资源账户创建，拥有自己的地址
    - 多签账户将存储多签配置（所有者列表，阈值）和要执行的交易列表。交易必须按顺序执行（或拒绝），这增加了确定性。
    - 此模块还允许所有者使用标准用户交易创建和批准/拒绝多签交易（这些功能将作为标准公共入口函数）。只有执行这些多签账户交易才需要新的交易类型。

```rust
struct MultisigAccount has key {
  // 所有者地址列表。
  owners: vector<address>,
  // 要通过交易所需的签名数量（k中的k-of-n）。
  signatures_required: u64,
  // 从交易id（递增id）到要为此多签账户执行的交易的映射。
  // 已执行的交易将被删除以节省存储空间，但始终可以通过事件访问。
  transactions: Table<u64, MultisigTransaction>,
  // 上次执行或拒绝的交易id。用于强制执行提案的有序性。
  last_transaction_id: u64,
  // 要分配给下一个交易的交易id。
  next_transaction_id: u64,
  // 控制多签的签名者能力（资源）账户。这可以用于交换签名者。
  // 目前未使用，因为 MultisigTransaction 可以在 VM 中直接验证和创建签名者，但这对于未来的链上可组合性可能是有用的。
  signer_cap: Option<SignerCapability>,
}

/// 要在多签账户中执行的交易。
/// 这必须包含完整的交易有效载荷或其哈希值（存储为字节）。
struct MultisigTransaction has copy, drop, store {
  payload: Option<vector<u8>>,
  payload_hash: Option<vector<u8>>,
  // 已批准的所有者。使用简单映射去重。
  approvals: SimpleMap<address, bool>,
  // 已拒绝的所有者。使用简单映射去重。
  rejections: SimpleMap<address, bool>,
  // 创建此交易的所有者。
  creator: address,
  // 关于交易的元数据，如描述等。
  // 这也可以在未来重复使用，以添加多签交易的新属性，例如到期时间。
  metadata: SimpleMap<String, vector<u8>>,
}
```


### 新的交易类型以执行多签账户交易

```rust
// 用于入口函数有效负载的现有结构
#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize, Deserialize)]
pub struct EntryFunction {
    pub module: ModuleId,
    pub function: Identifier,
    pub ty_args: Vec<TypeTag>,
    #[serde(with = "vec_bytes")]
    pub args: Vec<Vec<u8>>,
}
// 用于 EntryFunction 负载的现有结构，例如调用 "coin::transfer"
#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize, Deserialize)]
pub struct EntryFunction {
    pub module: ModuleId,
    pub function: Identifier,
    pub ty_args: Vec<TypeTag>,
    #[serde(with = "vec_bytes")]
    pub args: Vec<Vec<u8>>,
}

// 我们在这里使用枚举来实现可扩展性，以便我们可以在将来添加脚本有效负载支持，例如：
pub enum MultisigTransactionPayload {
    EntryFunction(EntryFunction),
}

/// 允许多签账户的所有者以多签账户身份执行预先批准的交易的多签交易。
#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize, Deserialize)]
pub struct Multitsig {
    pub multisig_address: AccountAddress,

    // 如果已经在链上存储，则交易有效负载是可选的。
    pub transaction_payload: Option<MultisigTransactionPayload>,
}
```

### 端到端流程

1. 拥有者可以通过调用 `multisig_account::create` 来创建一个新的多重签名账户。
    1. 这可以作为普通用户交易（入口函数）完成，或者通过另一个在其之上构建的模块在链上完成。
2. 拥有者可以随时通过调用 `multisig_account::add_owners` 或  `remove_owners` 来添加 / 移除拥有者。进行此类交易仍需遵循多签账户指定的 k-of-n 方案。
3. 要创建一个新的交易，拥有者可以调用 `multisig_account::create_transaction` 并提供交易负载：指定的模块（地址 + 名称）、要调用的函数名称和参数值。
    1. 负载数据结构仍在实验阶段。我们希望使离线系统能够正确构建此负载（或负载哈希），并且在出现问题时能够进行调试。
    2. 这将在链上存储完整的交易负载，这增加了去中心化（无法进行审查），并且使得获取所有等待执行的交易变得更容易。
    3. 如果需要进行燃气优化，拥有者可以选择调用 `multisig_account::create_transaction_with_hash`，其中仅存储负载哈希（模块 + 函数 + 参数）。以后的执行将使用哈希进行验证。
    4. 只有拥有者可以创建交易，并且将分配交易ID（递增ID）。
4. 交易必须按顺序执行。但是，拥有者可以提前创建多个交易并对其进行批准/拒绝。
5. 要批准或拒绝交易，其他拥有者可以调用 `multisig_account::approve()` 或 `reject()` 并提供交易ID。
6. 如果有足够的拒绝（≥ 签名门槛），任何拥有者都可以通过调用 `multisig_account::remove()` 来移除交易。
7. 如果有足够的批准（≥ 签名门槛），任何拥有者都可以使用特殊的 MultisigTransaction 类型创建下一个交易（通过创建），如果仅在链上存储了哈希，则交易负载是可选的。如果在创建时存储了完整的负载，则多重签名交易不需要指定除多签账户地址之外的任何参数。VM中的详细流程如下：
    1. 交易序言：VM首先调用一个私有函数（`multisig_account::validate_multisig_transaction`）来验证提供的多重签名账户中待执行的下一个交易是否存在，并且是否有足够的批准来进行执行。
    2. 交易执行：
        1. VM首先获取多签交易中底层调用的负载。如果交易序言（验证）成功，这个步骤不应该失败。
        2. VM然后尝试执行此函数并记录结果。
        3. 如果成功，VM调用 `multisig_account::successful_transaction_execution_cleanup` 来跟踪和发出成功执行的事件。
        4. 如果失败，VM丢弃执行负载的结果（通过重置VM会话），同时保留到目前为止已花费的燃气。然后它调用 `multisig_account::failed_transaction_execution_cleanup` 来跟踪失败。
        5. 最后，将燃气收取给发送者账户，并解析任何挂起的 Move 模块发布（如果多重签名交易发布了模块）。

### 参考实现

[https://github.com/aptos-labs/aptos-core/pull/5894](https://github.com/aptos-labs/aptos-core/pull/5894)

## 风险和缺陷

主要风险是智能合约风险，即智能合约代码（multisig_account 模块）或 API 和 VM 执行中可能存在错误或漏洞。可以通过彻底的安全审计和测试来减轻此风险。

## 未来潜力

此提案的一个即时扩展是为多重签名交易添加脚本支持。这将允许定义更复杂的原子多重签名交易。

长期来看：

提案本身将不允许在链上执行多重签名交易 - 其他模块只能创建交易并允许拥有者批准/拒绝。执行将需要发送专用事务的多重签名交易类型。然而，未来可以通过 Move 中的动态调度支持来使其变得更容易。这将允许在链上执行，并且还可以通过标准用户事务类型（而不是特殊的多重签名事务类型）进行链下执行。动态调度还可以允许向多重签名账户模型添加更多的模块化组件，以启用自定义交易验证等。

## 建议的实施时间表

目标完成代码（包括安全审计）和测试网发布：2023年2月。