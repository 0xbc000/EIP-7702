---
timezone: UTC+8
---

# none

无名的人阿！

>我是这路上 没名字的人
我没有新闻 没有人评论
要拼尽所有 换得普通的剧本
曲折辗转 不过谋生
我是离开 小镇上的人
是哭笑着 吃过饭的人
是赶路的人 是养家的人
是城市背景的 无声
我不过 想亲手触摸
弯过腰的每一刻
留下的 湿透的脚印 是不是值得
这哽咽 若你也相同
就是同路的朋友


坚持走完这一路

## Notes

<!-- Content_START -->

### 2025.05.14
#### EIP-7702缘起
* **why**提出要EIP-7702?
    * 传统EOAz账户只能发起交易，却无法执行合约逻辑，用户体验与智能合约钱包相去甚远。EIP‑7702 允许在单笔交易上下文中「暂时为 EOA 设置合约代码」，赋予其智能账户能力，从而简化UX。相较于此前的EIP‑3074引入新操作码导致生态碎片化，7702通过新增交易类型（SET_CODE_TX_TYPE）而非扩展 EVM 指令，兼容性更佳，不需额外调用代理合约，进一步对齐ERC‑4337路线图。
    * 兼容并衔接 ERC‑4337。EIP‑4337 已提出基于「用户操作（UserOp）」的抽象账户架构，而 EIP‑7702 可直接在现有 EOA 上执行 Account Abstraction，无需迁移新地址，并与已部署的智能合约钱包共存。
* **who**提出的EIP‑7702
    * EIP‑7702 的撰写者和提案人是 Vitalik Buterin，同时 Tin Erispe 和 Ayush Bherwani 也对该提案做出了贡献。
* EIP‑7702的核心目的
    * 智能账户能力：EOA 可临时执行合约逻辑，实现批量交易（batching）、Gas 赞助（sponsorship）、社交恢复（social recovery）等功能。
    * 简化钱包UX：无需预先持有 ETH、单签名即可完成多步操作，用户可享受类似 Web2 的流畅体验。
    * 安全可控：链 ID 绑定、Nonce 绑定、可撤销授权等多重防护，避免被永久锁定或跨链滥用。
* Pectra
    * EIP‑7702提案随 Ethereum 2025 年 5 月 7 日上线的 Pectra 升级一并激活，Pectra 是迄今为止最全面的一次硬分叉，包含共 11 项 EIP 标准，除 EIP‑7702 外，还包括企业级质押（EIP‑7251）、验证者可编程退出（EIP‑7002/6110）、Layer‑2 提效（EIP‑7691/7523）、以及若干预编译与历史哈希等安全性能优化提案。

| EIP 编号   | 名称                          | 功能简介                           |
| -------- | --------------------------- | ------------------------------ |
| EIP‑7702 | Smart Accounts for Everyone | 让 EOA 临时具备智能账户能力               |
| EIP‑7251 | Enterprise‑Grade Staking    | 单验证者质押上限从 32 ETH 提升至 2,048 ETH |
| EIP‑7002 | Execution‑Layer Exits       | 允许执行层发起验证者主动退出                 |
| EIP‑6110 | On‑Chain Deposit Handling   | 缩短验证者激活时间从 \~12h 到 \~13min     |
| EIP‑7691 | Increased Blobspace         | 区块中 Blobspace 翻倍，促进 L2 数据可用性   |
| EIP‑7523 | State Bloat Mitigation      | 清理空账户、防止无谓合约创建                 |
| EIP‑2537 | BLS Precompile              | 内建 BLS12‑381 操作，加速聚合签名         |
| EIP‑2935 | Historical Hashes           | 存储历史区块哈希，支持无状态客户端              |
| EIP‑7594 | Peer-to-Peer DAS            | P2P 数据可用性抽样，强化 L2 安全           |
| …        | …                           | 其他如安全与开发者体验优化等                 |

### 2025.05.15
#### EIP‑7702 交易数据格式
* 交易类型
```
TransactionType = 0x04（Set‑Code 交易）
```
* TransactionPayload
```
[
  chain_id,
  nonce,
  max_priority_fee_per_gas,
  max_fee_per_gas,
  gas_limit,
  destination,        // 不能为 null
  value,
  data,
  access_list,       // 同 EIP‑4844
  authorization_list,  
  signature_y_parity,
  signature_r,
  signature_s
]

```
* authorization_list
```
[ chain_id, address, nonce, y_parity, r, s ]
```

### 2025.05.16
在本地 Anvil 节点上，通过 Alloy 库发送 EIP-7702 类型的交易
```rust
use alloy::{
    eips::eip7702::Authorization,
    network::{EthereumWallet, TransactionBuilder, TransactionBuilder7702},
    node_bindings::Anvil,
    primitives::U256,
    providers::{Provider, ProviderBuilder},
    rpc::types::TransactionRequest,
    signers::{local::PrivateKeySigner, SignerSync},
    sol,
};
use eyre::Result;

// 使用 Solidity 合约代码生成 Rust 接口
sol!(
    #[allow(missing_docs)]
    #[sol(rpc, bytecode = "608080604052...")]
    contract Log {
        #[derive(Debug)]
        event Hello();
        event World();

        function emitHello() public {
            emit Hello();
        }

        function emitWorld() public {
            emit World();
        }
    }
);

#[tokio::main]
async fn main() -> Result<()> {
    // 启动本地 Anvil 节点，启用 Prague 硬分叉
    let anvil = Anvil::new().arg("--hardfork").arg("prague").try_spawn()?;

    // 创建两个用户，Alice 和 Bob
    let alice: PrivateKeySigner = anvil.keys()[0].clone().into();
    let bob: PrivateKeySigner = anvil.keys()[1].clone().into();

    // 使用 Bob 的钱包创建提供者
    let rpc_url = anvil.endpoint_url();
    let wallet = EthereumWallet::from(bob.clone());
    let provider = ProviderBuilder::new().wallet(wallet).on_http(rpc_url);

    // 部署 Alice 授权的合约
    let contract = Log::deploy(&provider).await?;

    // 创建授权对象，供 Alice 签名
    let authorization = Authorization {
        chain_id: U256::from(anvil.chain_id()),
        address: *contract.address(),
        nonce: provider.get_transaction_count(alice.address()).await?,
    };

    // Alice 对授权对象进行签名
    let signature = alice.sign_hash_sync(&authorization.signature_hash())?;
    let signed_authorization = authorization.into_signed(signature);

    // 准备调用合约的 calldata
    let call = contract.emitHello();
    let emit_hello_calldata = call.calldata().to_owned();

    // 构建交易请求
    let tx = TransactionRequest::default()
        .with_to(alice.address())
        .with_authorization_list(vec![signed_authorization])
        .with_input(emit_hello_calldata);

    // 发送交易并等待广播
    let pending_tx = provider.send_transaction(tx).await?;

    println!("Pending transaction... {}", pending_tx.tx_hash());

    // 等待交易被包含并获取收据
    let receipt = pending_tx.get_receipt().await?;

    println!(
        "Transaction included in block {}",
        receipt.block_number.expect("Failed to get block number")
    );

    assert!(receipt.status());
    assert_eq!(receipt.from, bob.address());
    assert_eq!(receipt.to, Some(alice.address()));
    assert_eq!(receipt.inner.logs().len(), 1);
    assert_eq!(receipt.inner.logs()[0].address(), alice.address());

    Ok(())
}

```

<!-- Content_END -->
