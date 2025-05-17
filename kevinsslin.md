---
timezone: UTC+8
---
# Kevin Lin

1. 自我介绍：我是 Kevin，來自台灣，喜歡學習 DeFi 並分享
2. 你认为你会完成本次残酷学习吗？：會的會的
3. 你的联系方式（推荐 Telegram）：[@kevinsslin](https://t.me/kevinsslin)

## Notes

<!-- Content_START -->

### 2025.05.14

see below

### 2025.05.15

# EIP-7702 Proposal Note

## One-Sentence Summary
EIP-7702 enables Externally Owned Accounts (EOAs) to set their own code via a special transaction, allowing them to act like smart contract wallets by delegating execution to a contract address.

## Motivation: Three Core Features
- **Batching**: Perform multiple operations (e.g., ERC-20 approve + transfer) in a single atomic transaction.
- **Sponsorship**: Allow a third party (like a DApp or relayer) to pay gas fees on behalf of a user.
- **Privilege De-escalation**: Enable sub-keys with limited permissions (e.g., only spend a specific token, set daily limits, or restrict to certain apps).

## EIP-2718 "Set Code Transaction" Structure
- **TransactionType**: `SET_CODE_TX_TYPE` (`0x04`)
- **TransactionPayload**:
  ```solidity
  rlp([
    chain_id,
    nonce,
    max_priority_fee_per_gas,
    max_fee_per_gas,
    gas_limit,
    destination,
    value,
    data,
    access_list,
    authorization_list,
    signature_y_parity,
    signature_r,
    signature_s
  ])
  ```
  - **authorization_list**:
    ```solidity
    [
      [chain_id, address, nonce, y_parity, r, s],
      ...
    ]
    ```
    - Each tuple is a signature authorizing the EOA (authority) to delegate to a contract (address).
    - `authority = ecrecover(msg, y_parity, r, s)`
    - `msg = keccak(MAGIC || rlp([chain_id, address, nonce]))`
  - The outer transaction fields follow the same semantics as EIP-4844.
  - The transaction signature (`signature_y_parity`, `signature_r`, `signature_s`) is over `keccak256(SET_CODE_TX_TYPE || TransactionPayload)`.
- **ReceiptPayload**:
  ```solidity
  rlp([status, cumulative_transaction_gas_used, logs_bloom, logs])
  ```

## Delegation Indicator
- Uses the banned opcode `0xef` (from EIP-3541), so the EVM treats the code as a delegation pointer, not regular code.
- Format: `0xef0100 || address`
- When called, the EVM loads and executes the code at `address` in the context of the EOA (authority).

## Misc
- If transaction execution fails (e.g., revert), the delegation indicator is not rolled back — the delegation remains.
- To change or remove delegation, send a new EIP-2718 transaction (set to a new contract or to the zero address to revert to a normal EOA).
- During delegated execution, `CODESIZE` and `CODECOPY` behave differently from `EXTCODESIZE` and `EXTCODECOPY` on the authority.
  - For example, when executing a delegated account:
    - `EXTCODESIZE` returns 23 (the size of `0xef0100 || address`)
    - `CODESIZE` returns the size of the code residing at the delegated address

## Breaking Invariants
- `tx.origin == msg.sender` is only true in the topmost execution frame.
  - This breaks some legacy patterns:
    - Ensuring `msg.sender` is an EOA (no longer reliable)
    - Protecting against atomic sandwich attacks (bad practice anyway)
    - Preventing reentrancy (rarely used for this purpose)

## Security Considerations
- **Storage Management**: Delegated contracts should avoid storage collisions (e.g., use ERC-7201 Storage Namespaces).
- **Replay Protection**: Delegated contracts should implement their own nonces and signature checks.

## References
- [EIP-7702: Set Code for EOAs](https://eips.ethereum.org/EIPS/eip-7702)
- [EIP-2718: Typed Transaction Envelope](https://eips.ethereum.org/EIPS/eip-2718)
- [EIP-4844: Shard Blob Transactions](https://eips.ethereum.org/EIPS/eip-4844)
- [EIP-3541: Reject New Contracts Starting with the 0xEF Byte](https://eips.ethereum.org/EIPS/eip-3541)
- [ERC-7201: Namespaced Storage Layout](https://eips.ethereum.org/EIPS/eip-7201)

### 2025.05.16

### 2025.05.17

# 🔍 A Deep Dive into EIP-7702 with Best Practices

> **Speaker:** Kong (Leader of Security Audit Team)
> [Original Video](https://www.youtube.com/watch?v=uZTeYfYM6fM)

---

## 📌 Recap & Additional Notes of EIP-7702

* **Chain ID Replay**:

  * 若將 `chainId` 設定為 `0`，可以在所有支持 EIP-7702 的鏈上進行 replay。
  * 對錢包服務商友善，僅需用戶簽署一次鏈下簽名即可在所有鏈上創建智能合約錢包。

* **交易與授權分離**:
  
  * 交易的發起者（付 gas fee）和交易的授權者（鏈下簽名者）可以是不同人，實現 gas fee 的代付。
  * 同一交易中同一地址的多個授權規則中，只有最新的一條會被應用。

* **Address Creation Limitations (EIP-3541)**:

  * 開發者無法使用 `new`, `create`, `create2` 等方式創建以 `0xef` 開頭的地址。

---

## 🚩 Best Practices

### 1. 🔑 Private Key Management

* 即使透過 EIP-7702 可實現如 social recovery 等私鑰遺失的方案，但私鑰仍具最高權限，能重新 delegate 給其他地址。
* 私鑰管理仍極為重要，應設計嚴謹的保護機制。

### 2. 🔄 Multi-chain Replay Risks

* 將 `chainId` 設為 `0` 啟用多鏈 replay，需注意不同鏈上同一地址的智能合約代碼可能不同，存在潛在安全風險。

### 3. 🚧 Atomic Initialization and Delegation Issues

* EIP-7702 不允許在單一交易中同時進行 delegate 和合約初始化（no initcode）。
* 存在搶跑（front-run）攻擊風險，應分開處理 delegation 與 initialization。

### 4. 📦 Storage Management

* Delegation 切換可能引發 storage conflict，建議：

  * for user: 在 re-delegate 前將資金提取以避免損失。
  * for dev: 採用 [ERC-7201 Storage Namespaces](https://eips.ethereum.org/EIPS/eip-7201) 分隔儲存空間。
  * for wallet service provider: 利用 [ERC-7779](https://eips.ethereum.org/EIPS/eip-7779) 檢查儲存相容性並清理舊的儲存資料。

### 5. 🚨 False Top-up 防範

* 隨著智能合約錢包增多，交易所可能遭遇更多智能合約 deposit 的情況。
* 建議透過交易追蹤（tracing）防止 false top-up 攻擊（[詳細說明](https://slowmist.medium.com/how-does-the-false-top-up-attack-break-through-the-defense-of-the-exchange-d6e8ebb434f5)）。

### 6. 🔄 Account Conversion (EOA ↔ CA)

* EOA 和智能合約帳戶（CA）之間的轉換可能，使得 `msg.sender == tx.origin` 的假設失效。
* 開發者不應再假設交易發起者一定是 EOA。

### 7. ✅ Contract Compatibility

* Delegated contract 須確認與各種 token 或協議的相容性，例如 ERC-721 NFT 需要實作 `onERC721Received()` callback。

### 8. 🎣 Phishing Risks

* EIP-7702 的鏈下簽名授權將使釣魚攻擊更容易執行。
* 錢包服務提供商應詳盡提醒用戶授權合約的詳細資訊以防範攻擊。

---

### 📖 Sources & Further Reading

* [EIP-7702 Official Documentation](https://eips.ethereum.org/EIPS/eip-7702)
* [EIP-3541 Details](https://eips.ethereum.org/EIPS/eip-3541)
* [ERC-7201: Storage Namespaces](https://eips.ethereum.org/EIPS/eip-7201)
* [ERC-7779 Storage Compatibility Checks](https://eips.ethereum.org/EIPS/eip-7779)
* [SlowMist's False Top-up Explanation](https://slowmist.medium.com/how-does-the-false-top-up-attack-break-through-the-defense-of-the-exchange-d6e8ebb434f5)


<!-- Content_END -->


