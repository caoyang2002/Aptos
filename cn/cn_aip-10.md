---
aip: 10
title: Move Objects
author: davidiw, wrwg
discussions-to: https://github.com/aptos-foundation/AIPs/issues/27
Status: Accepted
last-call-end-date:
type: Standard (Framework)
created: 2023/01/05
updated: 2023/01/23
requires: "AIP-9: Resource Groups"
---

# AIP-10 - Move Objects

## 概要

这个AIP提议了 *Move object*，用于全局访问存储在单个地址上的异构资源集合。对象提供了一个丰富的能力模型，允许进行细粒度的资源控制和所有权管理。通过利用账户模型的各个方面，对象可以直接发出事件，从而使链上操作更加丰富。

## 动机

对象模型允许Move将一个复杂类型表示为存储在单个地址内的一组资源。在对象模型中，NFT或代币可以将通用的代币数据放在`Token`资源中，将对象数据放在`ObjectCore`资源中，然后根据需要进一步特化，例如，`Player`对象可以在游戏中定义一个玩家。`ObjectCore`本身存储了当前所有者的`address`以及适用于创建事件流的适当数据。

对象模型通过使用由不同访问器或引用实现的能力框架，提高了类型安全性和数据访问。这些包括：

- `ConstructorRef` 允许创建所有其他能力，允许向对象添加资源，并且只能在创建对象时访问。
- `Object<T>` 指向包含`T`资源的对象。这对于存储对资源的反向查找很有用。
- `DeleteRef `允许持有者从存储中删除对象。
- `ExtendRef `允许持有者访问签名者，以添加新资源。
- `TransferRef` 允许持有者在创建后将新资源转移到对象。

在模块中存储的 `DeleteRef` 可以允许创建者或所有者销毁 tokrn（如果存在）。而 `TransferRef `可以在模块内部使用，通过定义逻辑以确定何时可以转移对象以及由谁转移，或者可以赠送（be given away）并将其视为一个能力来转移对象。

此外，对象还实现了对象的可组合性，允许对象拥有其他对象。每个对象都在其状态中存储其所有者的身份。所有者可以通过创建并将 `Object<T>` 存储在自己的存储空间中来跟踪其拥有的对象。从而在对象模型内实现无缝的双向导航。

## 原因

现有的 Aptos 数据模型强调了 Move 中的 `store` 能力的使用。Store 允许一个结构存在于任何存储在全局存储中的结构中，这是由`key` 能力标记的。因此，数据可以存在于任何结构内部的任何地方和任何地址上。虽然这提供了很大的灵活性，但它也有许多局限性：

- 不能保证数据是可访问的，例如，它可能被放置在一个违反该数据期望的用户定义的资源中，例如，一个创建者试图去销毁被放入用户自定义存储中的NFT。这可能会令用户和创建者对数据感到困惑。
- 不同类型的数据可以通过`any`存储到单个数据结构（例如，map、vector）中，但对于复杂数据类型，`any`在 Move 中会带来额外的成本，因为每次访问都需要反序列化。如果API开发人员期望特定的任何字段改变其所代表的类型，则可能会导致混淆。
- 资源账户允许数据具有更大的自治权，但对于对象来说效率低下，并且没有充分利用资源组。
- 数据不能递归组合，Move目前对递归数据结构有限制。此外，经验表明，真正的递归数据结构可能会导致安全漏洞。
- 现有数据无法从入口函数中轻松引用，例如，为了支持字符串验证需要大量的代码行。由于表中每一个键都可能具有高度的独特性，为入口函数提供专门的支持因而显得尤为复杂。
- 事件的触发不是来自数据本身，而是必须由一个账户来完成，这个账户可能并不与数据直接相关联。
- 转移逻辑受限于各个模块提供的API，并且通常需要在发送方和接收方都加载资源，从而增加了不必要的成本开销。

### 替代方案

在这条路线上，考虑了几种替代方案：

* 使用 `OwnerRef` 而不是在对象内定义所有权。`OwnerRef` 提供了一种自然的 "嵌套" 对象的方法。然而，`OwnerRef` 需要额外的存储空间，仍然需要访问对象以提供门控存储，并且在可删除对象的情况下变成弱引用。
* 有一个显式的对象存储。Move的其他实现实现了动态字段、包和其他机制来跟踪所有权。这需要额外的存储空间，限制了可删除性，并且不可迭代。虽然这可能对某些应用有用，但 Move 对象的目标是让开发人员能够指定他们喜欢的路径，尽可能不持有意见。
* 允许对象具有存储功能。在对象上直接具有存储功能与对象的原则相悖，即始终是全局可访问的。
* 更广泛的引用或能力集。有很多机会投资于更丰富的访问和管理对象的方式。目前的目标不是在这一点上指定一套最佳实践，而是将这个任务推迟到对象在社区中有足够的经验积累后。最终，这里定义的对象核心应该足以支持任何方向的更高级功能，最好是能够横跨所有应用程序。
* 添加对现有引用或能力的吊销支持。每个具有存储能力的引用都可以被放置到任何模块中，以允许创建者指定对象随时间如何演变。由于这样，实际上不存在一种通用的方法来吊销特定能力。能力可以授予给一个账户或一个模块，并且可以存储在任何地方。目的是保持它们的灵活性，以免影响存储或增加在使用中的摩擦。当然，要小心使用能力，因为对能力的天真使用可能会导致不良行为。

## 规范

### 概述

对象构建时考虑了以下几点：

- 简化的存储接口，支持将异构集合的资源存储在一起。这使得数据类型可以共享一个共同的核心数据层（例如，代币），同时具有更丰富的扩展（例如，音乐会门票，剑）。
- 全局可访问的数据和所有权模型，使得创建者和开发者可以决定数据的应用程序和生命周期。
- 可扩展的编程模型，支持利用核心框架包括代币在内的用户应用程序的个性化。
- 支持直接发出事件，从而提高了与对象相关的事件的可发现性。
- 考虑到底层系统，利用资源组实现了更高的燃气效率，避免了昂贵的反序列化和序列化成本，并支持可删除性。

### 对象生命周期

- 一个实体调用顶级对象类型上的 create。
- 顶级对象类型调用其直接祖先上的 create。
- 这一过程重复进行，直到最顶层的祖先是 `Object` 结构，该结构定义了 `create_object` 函数。
- `create_object` 生成一个新的地址，在 `ObjectGroup` 资源组内的该地址存储一个 `Object` 结构，并返回一个 `ConstructorRef`。
- 之前调用的 create 函数可以使用 `ConstructorRef` 来获取一个签名者，在 `ObjectGroup` 资源内存储一个适当的结构，进行任何额外的修改，并将 `ConstructorRef` 返回回来。
- 在对象创建栈中，任何一个模块都可以定义属性，如所有权、删除、可转移性和可变性。

### 核心对象

对象存储在 `ObjectGroup` 资源组中。这允许对象内的其他资源共同存放，以实现数据局部性和数据成本节约。请注意，对象内的所有资源不需要都在 `ObjectGroup` 内共同存放。这由对象的开发者定义其数据布局。

```move
#[resource_group(scope = global)]
struct ObjectGroup { }
```

核心的 `ObjectCore` 由以下 Move 结构表示：

```move
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct ObjectCore has key {
    /// 用于 guid 以保证全局唯一的对象和创建事件流
    guid_creation_num: u64,
    /// 拥有此对象的地址（对象或账户）
    owner: address,
    /// 对象转移是一个常见操作，这允许禁用和启用转移。绕过使用 TransferRef。
    allow_ungated_transfer: bool,
    /// 在转移所有权时发出的事件。
    transfer_events: event::EventHandle<TransferEvent>,
}
```

### 对象

每个对象存储在其自己的地址中，由 `Object<T>` 表示。底层地址可以从用户提供的输入或当前账户的全局唯一 ID 生成器 (`guid.move`) 生成。此外，地址生成利用了与现有账户地址生成不同的域分隔符：`sha3_256(address_of(creator) | seed | DOMAIN_SEPaRATOR)`。

- GUID
  - `seed` 是由 `guid.move` 产生的当前 guid：`bcs::to_bytes(account address | u64)`。
  - 可以删除，因为对象无法重新创建。
  - `DOMAIN_SEPaRATOR` 是 0xFD。
- 用户指定
  - `seed` 由用户定义，比如一个字符串。
  - 不能删除，因为对象可以重新创建，最小限度地，这样的冲突可能会影响正确的事件编号序列。
  - `DOMAIN_SEPaRATOR` 是 0xFE。

### 能力

- `Object<T>`
    - 对象指针，地址的包装器
    - 能力：`copy, drop, store`
    - 数据布局 `{ inner: address }`
- `ConstructorRef`
    - 在创建时提供给对象创建者
    - 可以创建任何其他 ref 类型
    - 能力：`drop`
    - 数据布局 `{ self: ObjectId, can_delete: bool }`
- `DeleteRef`
    - 用于从 `ObjectGroup` 中移除对象
    - 对于单个对象可以有多个
    - 如果对象地址是由用户输入生成的，则无法创建
    - 能力：`drop, store`
    - 数据布局：`{ self: ObjectId }`
- `ExtendRef`
    - 用于创建事件或将其他资源移入对象存储中。
    - 能力：`drop, store`
    - 数据布局 `{ self: ObjectId }`
- `TransferRef`
    - 用于创建 LinearTransferRef，从而进行所有权转移。
    - 能力：`drop, store`
    - 数据布局 `{ self: ObjectId }`
- `LinearTransferRef`
    - 用于执行转移。
    - 强制一个实体只能转移一次，假设他们没有直接访问 `TransferRef`
    - 能力：`drop`
    - 数据布局 `{ self: ObjectId, owner: address }`

### API 函数

用于地址转换到和从 `ObjectId` 的函数：
```move
/// 根据给定地址生成一个 Object。这将验证 T 是否存在。
public fun address_to_object<T: key>(object: address): Object<T>;

/// 返回 ObjectId 内的地址。
public fun object_address<T>(object: &Object<T>): address;

/// 从源材料中派生对象地址：sha3_256([creator address | seed | 0xFE])。
/// Object 必须与 create_resource_address 不同。
public fun create_object_address(source: &address, seed: vector<u8>): ObjectId;
```

用于创建对象的函数：
```move
/// 创建一个新的命名对象并返回 ConstructorRef。通过知道用于创建它们的用户生成的种子，可以全局查询命名对象。命名对象无法删除。
public fun create_named_object(creator: &signer, seed: vector<u8>): ConstructorRef;

/// 从由账户生成的 GUID 创建一个新对象。
public fun create_object_from_account(creator: &signer): ConstructorRef;

/// 从由对象生成的 GUID 创建一个新对象。
public fun create_object_from_object(creator: &signer): ConstructorRef;

/// 生成 DeleteRef，可用于从全局存储中移除对象。
public fun generate_delete_ref(ref: &ConstructorRef): DeleteRef;

/// 生成 ExtendRef，可用于向对象添加新事件和资源。
public fun generate_extend_ref(ref: &ConstructorRef): ExtendRef;

/// 生成 TransferRef，可用于管理对象的转移。
public fun generate_transfer_ref(ref: &ConstructorRef): TransferRef;

/// 为 ConstructorRef 创建一个签名者
public fun generate_signer(ref: &ConstructorRef): signer;

/// 返回 ConstructorRef 中的地址
public fun object_from_constructor_ref<T: key>(ref: &ConstructorRef): Object<T>;
```

用于向对象添加新事件和资源的函数：
```move
/// 为对象创建一个 GUID，通常用于事件
public fun create_guid(object: &signer): guid::GUID;

/// 生成一个新的事件句柄。
public fun new_event_handle<T: drop + store>(object: &signer): event::EventHandle<T>;
```

用于删除对象的函数：
```move
/// 返回 DeleteRef 中的地址。
public fun object_from_delete<T: key>(ref: &DeleteRef): Object<T>;

/// 从全局存储中删除指定的对象。
public fun delete(ref: DeleteRef);
```

用于扩展对象的函数：
```move
/// 为 ExtendRef 创建一个签名者
public fun generate_signer_for_extending(ref: &ExtendRef): signer;
```

用于转移对象的函数：
```move
/// 禁用直接转移，转移只能通过 TransferRef 触发
public fun disable_ungated_transfer(ref: &TransferRef);

/// 启用直接转移。
public fun enable_ungated_transfer(ref: &TransferRef);

/// 为一次性转移生成一个 LinearTransferRef。这要求生成时的所有者也是转移时的所有者。
public fun generate_linear_transfer_ref(ref: TransferRef): LinearTransferRef;

/// 使用 LinearTransferRef 将对象转移到目标地址。
public fun transfer_with_ref(ref: LinearTransferRef, to: address);

/// 可以用于转移的入口函数，如果 allow_ungated_transfer 设置为 true。
public entry fun transfer_call(owner: &signer, object: address, to: address);

/// 如果 allow_ungated_transfer 设置为 true，则转移给定对象。请注意，这允许嵌套对象的所有者转移该对象，只要在层次结构的每个阶段都启用了 allow_ungated_transfer。
public fun transfer<T>(owner: &signer, object: Object<T>, to: address);

/// 将给定对象转移到另一个对象。有关更多信息，请参见 `transfer`。
public fun transfer_to_object<O, T>(owner: &signer, object<O>: Object, to: Object<T>);
```

用于验证所有权的函数
```move
/// 返回当前所有者。
public fun owner<T: key>(object: Object<T>): address;

/// 如果提供的地址是当前所有者，则返回 true。
public fun is_owner<T: key>(object: Object<T>, owner: address): bool;
```

### API 事件

```move
/// 当对象的所有者字段发生更改时发出。
struct TransferEvent has drop, store {
    object: address,
    from: address,
    to: address,
}
```

## 参考实现

首次提交见 https://github.com/aptos-labs/aptos-core/pull/5976

## 风险和缺点

讨论的开放领域包括：

- 添加和删除资源是否应计算对象中的资源数量以确保删除完整？
- 我们是否应将基于用户输入的对象标记为已删除，而不是根本不支持它？这样，DeleteRef 在更高层次仍然有用。
- 对象是否应跟踪已生成的 refs，并且它们是否应该是热土豆？或者，是否应该发出事件？
- `Object<T>` 指向对象是否应防止删除该对象？

当前的基础对象可以扩展以支持所有这些用例。决定是保守一些，以便在对象标准发展之后可以执行决定。

## 未来潜力

- 允许入口函数指定与对象关联的（预期的）功能，可以通过属性或通过允许传递某些 `Ref` 类型来实现。
- 允许用户指定可以转移到他们的哪些资源。这可以通过 1) 支持表示用户已接受的对象的用户对象存储或 2) 在帐户结构中具有指示用户是否愿意接受对象的标志来完成。
- `Alias<T>`
    - 假设模块 `hero` 定义了多个版本的结构体（`HeroV1`、`HeroV2`，...）
    - 应用程序只关心模块 `hero` 中 `Hero` 的概念而不是特定版本。
    - 由于借用全局语义，API 不能公开版本化的 `Hero`，模块本身可以期望 `Object<Hero>` 并分派到适用于 `Hero` 的相关版本。
- 禁用转移事件
    - 对象最终可能用于创建丰富的数据结构，事件的成本可能是不可忽略的。
    - 在气体优化后，这到底产生了多少成本相对于执行成本，尚待观察。
- 唯一资源命名的容器对象
    - 用于可替换资产的对象容器导致多个无法在没有作为输入提供给交易的显式引用的情况下找到的副本。
    - 让我们提供一个新的对象地址方案：`account_address | type_info<T>() | 0xFC`
    - 这些地址的对象是通过 `create_typed_object<T>(account: &signer, &_proof: T)` 创建的，该函数确保这些对象实际上将包含 T，一个用于可替换资产的存储。
    - 现在，任何其他对象都可以在链上读取它而无需任何来自交易的输入，例如，`balance<0x1::aptos_coin::AptosCoin>(addr)` 将从存储在 `addr | 0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin> | 0xFC` 的 `0x1::coin::CoinStore` 对象中读取。
    - 地址计算可能是不可忽略的，因此支持显式寻址的 API 也将是有益的。

## 建议的实施时间表

- 初步实现已完成，准备进行审查。
- 假设反馈总体上是积极的，这可能会进一步推进到 2 月测试网版本。
- 如果进展顺利，可能会在 3 月份发布到主网。