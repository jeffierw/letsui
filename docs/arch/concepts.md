---
sidebar_position: 4
---

# 概念

前面的章节详细解释了 Walrus 的内部运作方式，本章主要介绍使用 Walrus 时所需了解的所有概念。

## Walrus 的构成

从开发者的角度上来说，Walrus 一部分是 Sui 上的对象和智能合约，一部分是 Walrus 特定的二进制文件和服务。通常，Sui 用于管理 blob 和存储节点元数据，而 Walrus 特定的服务用于存储和读取 blob 内容，这些内容可能非常大。

### 合约

Walrus 在 Sui 上定义了许多[对象和智能合约](https://github.com/MystenLabs/walrus-docs/tree/main/contracts/blob_store)：

- 一个 shared system object 用来记录并管理当前的存储节点委员会。
  ```rust
  const BYTES_PER_UNIT_SIZE: u64 = 1_024;

  public struct System<phantom WAL> has key, store {
      id: UID,
      /// The current committee, with the current epoch.
      /// The option is always Some, but need it for swap.
      current_committee: Option<Committee>,
      /// When we first enter the current epoch we SYNC,
      /// and then we are DONE after a cert from a quorum.
      epoch_status: u8,
      // Some accounting
      total_capacity_size: u64,
      used_capacity_size: u64,
      /// The price per unit size of storage.
      price_per_unit_size: u64,
      /// Tables about the future and the past.
      past_committees: Table<u64, Committee>,
      future_accounting: FutureAccountingRingBuffer<WAL>,
  }

  public struct Committee has store {
    epoch: u64,
    bls_committee: BlsCommittee,
  }
  ```

  System 对象包含有关可用和已用存储的元数据，以及以MIST计价的每 KiB 存储的存储价格。系统对象内的委员会结构可用于读取当前 epoch number 以及有关委员会的信息。

  委员会的一些 public 职能允许合约读取 Walrus 元数据：
  ```rust
  /// Get epoch. Uses the committee to get the epoch.
  public fun epoch<WAL>(self: &System<WAL>): u64;

  /// Accessor for total capacity size.
  public fun total_capacity_size<WAL>(self: &System<WAL>): u64;

  /// Accessor for used capacity size.
  public fun used_capacity_size<WAL>(self: &System<WAL>): u64;

  // The number of shards
  public fun n_shards<WAL>(self: &System<WAL>): u16;
  ```
  
- Storage resources 表示可用于存储 blob 的空存储空间。
  ```rust
  /// Reservation for storage for a given period, which is inclusive start, exclusive end.
  public struct Storage has key, store {
      id: UID,
      start_epoch: u64,
      end_epoch: u64,
      storage_size: u64,
  }
  /// Constructor for [Storage] objects.
  /// Necessary to allow `blob_store::system` to create storage objects.
  /// Cannot be called outside of the current module and[blob_store::system].
  public(package) fun create_storage(
      start_epoch: u64,
      end_epoch: u64,
      storage_size: u64,
      ctx: &mut TxContext,
  ): Storage {
      Storage { id: object::new(ctx), start_epoch, end_epoch, storage_size }
  }
  ```

- Blob resources 表示正在注册并认证为已存储的 Blob。
  ```rust
  /// The blob structure represents a blob that has been   registered to with some storage,
  /// and then may eventually be certified as being available in the system.
  public struct Blob has key, store {
      id: UID,
      stored_epoch: u64,
      blob_id: u256,
      size: u64,
      erasure_code_type: u8,
      certified_epoch: option::Option<u64>, // Store the epoch first certified
      storage: Storage,
  }
  ```
- 对这些对象的更改会发出与 Walrus 相关的事件供 Walrus 处理存取操作。
  ```rust
  // Event definitions

  /// Signals a blob with meta-data is registered.
  public struct BlobRegistered has copy, drop {
      epoch: u64,
      blob_id: u256,
      size: u64,
      erasure_code_type: u8,
      end_epoch: u64,
  }

  /// Signals a blob is certified.
  public struct BlobCertified has copy, drop {
      epoch: u64,
      blob_id: u256,
      end_epoch: u64,
  }

  /// Signals that a BlobID is invalid.
  public struct InvalidBlobID has copy, drop {
      epoch: u64, // The epoch in which the blob ID is first registered as invalid
      blob_id: u256,
  }
  ```

首次注册 Blob 时，会发出 BlobRegistered 事件，通知存储节点它们预期获得与该 Blob ID 关联的碎片slivers。最终当 Blob 获得认证时，会发出BlobCertified事件 ，其中包含有关 Blob ID 和 Blob 将来被删除的 epoch信息。在该 epoch 之前，保证 blob 可用。

当存储节点检测到编码错误的 blob 时，会发出 InvalidBlobID 事件。任何尝试读取此类 blob 的人都肯定会检测到它是无效的。

上面提到 Storage 对象始终与 Blob 对象关联，为 Blob 的存储保留足够长的空间。经过认证的 blob 在底层存储资源保证存储期间可用。

可以使用以下函数通过Sui SDK调用读取其字段：
```rust
// Blob functions
public fun stored_epoch(b: &Blob): u64;
public fun blob_id(b: &Blob): u256;
public fun size(b: &Blob): u64;
public fun erasure_code_type(b: &Blob): u8;
public fun certified_epoch(b: &Blob): &Option<u64>;
public fun storage(b: &Blob): &Storage;

// Storage functions
public fun start_epoch(self: &Storage): u64;
public fun end_epoch(self: &Storage): u64;
public fun storage_size(self: &Storage): u64;
```

### 链下服务

Walrus 还由许多服务和二进制文件组成，这部分暂未开源：

- 客户端（二进制）可以在本地执行，并提供命令行界面 (CLI) 、 JSON API和HTTP API来执行 Walrus 操作。

- 聚合器服务允许通过 HTTP 请求读取 blob。

- 发布者服务用于向 Walrus 存储 blob。

- 一组存储节点存储编码的 blob。这些节点构成了 Walrus 的去中心化存储基础设施。

具体的使用将在[安装与使用章节](../install_and_use/install.md)介绍。
