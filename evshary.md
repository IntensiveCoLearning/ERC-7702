---
timezone: UTC+8
---

# evshary

1. 自我介绍：
     Web3 新手，最近對區塊鏈相關知識非常有興趣，想透過殘酷共學跟大家一起學習。
2. 你认为你会完成本次残酷学习吗？
     會
3. 你的联系方式（推荐 Telegram）
     t.me/evshary

## Notes

<!-- Content_START -->

### 2025.05.14

#### 為何需要 EIP-7702 （目前遇到什麼問題）

以太坊主要有兩種位址：
1. EOA (Externally Owned Accounts)
2. 智能合約帳戶 (Contract Account)

在 EIP-7702 之前，EOA 的功能相對簡單，主要是發送交易功能。不能像是智能合約錢包能夠執行程式碼。
因此使用者必須要先把資產轉移到智能合約錢包，對使用者來說十分麻煩。

在 EIP-7702 之後，EOA 就能支援批次交易處理、gas 贊助、或是複雜權限管理，少了一個轉帳的動作。EOA 更加接近抽象帳戶的概念。

#### EIP-7702 的運作方式

1. 使用者用 EOA 創建新的交易類型
2. 交易中會指定一個智能合約地址，EOA 在該交易中會臨時地像這個智能合約一樣運作
3. 用 EOA 私鑰簽署交易
4. EVM 在處理這筆交易時，會臨時地將發送方的 EOA 視為位於指定智能合約地址的合約，並執行該合約地址上的程式碼
5. 完成交易後，EOA 會回復正常模式

#### 引入 EIP-7702 的可能風險

1. 委託的智能合約如果沒有適當的存取限制，攻擊者可以存取 EOA 內的所有資產
2. 開發者確保初始化程式碼有正確的執行，不然可能會有邏輯錯誤或是被攻擊者利用
3. 委託程式碼不會清除現有儲存，因此某些變數在切換合約時可能會遺留舊數據
4. 過去 EOA 相對來說被認為比較安全，現在也引入了智能合約相關的風險

### 2025.05.15

學習目標：比較 EIP-4337 和 EIP-6551

#### EIP-4337

完整實現帳戶抽象，允許使用者直接用智能合約帳戶當成自己的以太坊帳戶。
這些帳戶就如同智能合約可以執行程式，如多重簽名、恢復帳戶等等進階功能。

* 優點：
  * 完整帳戶抽象：使用者可以直接擁有智能合約帳戶的各種進階功能
  * 不需要硬分叉：沒有改變以太坊的共識機制
* 缺點：
  * 無法無痛移轉：使用者需要創建並管理新的智能合約帳戶
  * 複雜性高：智能合約帳戶本身就比 EOA 複雜
  * 風險提高：會有智能合約相關的風險

#### EIP-6551

允許 NFT 有自己的智能合約帳戶，也就是 TBA (Token-Bound Account)。
這樣 NFT 就不再只是收藏品，也有管理資產以及跟其他 DApp 互動的能力。
這個帳戶由 NFT 的擁有者所控制。

* 優點：
  * 提高 NFT 價值：讓 NFT 能夠管理資產
  * 簡化體驗：和 NFT 相關的資產可以一起管理
* 缺點：
  * 依賴 ERC-721：只針對 ERC-721 的 NFT
  * 風險提高：會有智能合約相關的風險

#### 小結

簡單來說，EIP-7702 是讓使用者最低成本享受到智能合約功能的方式，不像是 EIP-4337 需要重新創個帳戶。
至於 EIP-6551 則只是針對 NFT，這就是另一個故事了。

### 2025.05.16

#### EIP-7702 實際封包差異

在 EIP-2718 前的格式(Type 0, Legacy Transaction)為如下：
[nonce, gasPrice, gasLimit, to, value, data, v, r, s]

而在 EIP-2718 之後則引入了 Typed Transactions，允許有多種交易種類。
當地一個 byte 小於 0x7f，表示是 EIP-2718 定義的 typed transaction，常見有如下幾種：

* (0x01) Type 1 (EIP-2930 Access List Transaction)：增加 AccessList 欄位
* (0x02) Type 2 (EIP-1559 Transaction): 引入了新的 Gas 費用機制
* (0x03) Type 3 (EIP-4844 Blob Transaction): 用來處理大量的 Blob 數據
* (0x04) Type 4 (EIP-7702): 也就是我們要探討的主角

下面是 EIP-7702 的格式：
[chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit, destination, value, data, access_list, authorization_list, signature_y_parity, signature_r, signature_s]

最主要核心是增加了 authorization_list，這個列表，包含了多個 authorization tuples，每個 tuple 有如下資訊

* chain_id: 鏈的 ID
* address: 被授權執行程式碼的智能合約位置
* nonce: EOA 的 nonce
* y_parity, r, s: 對特定訊息的簽名，用於授權

destination 不能為空，代表不能是創建合約的交易。另外 authorization_list 也不能為空。
讓我們用 Gas 贊助的例子來說明 destination 和 authorization_list 的差異：

1. EOA 發起 EIP-7702 交易，destination 是目標智能合約、authorization_list 包含 Gas 贊助智能合約的地址
2. 該 authorization tuples 會包含 EOA 簽名，代表有授權給贊助地址
3. EVM 會驗證 authorization_list 的 Gas 贊助智能合約
4. Gas 贊助智能合約會以某種形式為交易支付 Gas 費用
5. 交易成功送出，原發送者仍為該 EOA，但 Gas 成本由授權合約負擔

雖然交易由 EOA 簽名發出，但授權合約可以攔截交易並代為執行、支付 Gas，類似於 Account Abstraction 的使用者操作（UserOperation）行為

### 2025.05.17

EIP-7702 要注意的問題

* 私鑰仍然需要仔細保管
* 注意簽署 chainID 為 0 的委託，有可能有多鏈重放的問題
* EIP-7702 只有更新位址的 code 欄位，並沒有做初始化
* 如果使用者從 A 合約委託到 B 合約，存儲的結構有可能有變化。所以要確保重新委託後存儲結構是沒有問題。
* 注意假儲值的風險
* 外部合約不能假設發起交易的人都是 EOA，可能是 smart contract
* 開發者要確保跟主流基礎設施兼容
* 釣魚的風險：如果使用者把 EOA 委託給有問題的智能合約，就會有問題

EIP-7702 可以撤銷授權：把 address 設定為 `0x0000000000000000000000000000000000000000`，就可以把帳戶的 code 欄位變成空

<!-- Content_END -->
