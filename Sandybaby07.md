---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. 自我介绍 Hihi 我是Sandy
2. 你认为你会完成本次残酷学习吗？ 應該會：）
3. Telegram :sandyyy0707

## Notes

<!-- Content_START -->

### 2025.05.14

#### DESIGNATING A SMART CONTRACT TO AN EOA
role: EOA/Sequencer/Node

<img width="692" alt="image" src="https://github.com/user-attachments/assets/b846c750-ac2e-4b62-961f-dc60b7d7d9e8" />

```odyssey_sendTransaction```

odyssey_sendTransaction request that executes an EIP-7702 Transaction to designate a contract to the EOA (via authorizationList), as well as executing a function on it (via data):

```
curl 'https://odyssey.ithaca.xyz/' --data '{
  "jsonrpc":"2.0",
  "id":0,
  "method":"odyssey_sendTransaction",
  "params":[{
    "authorizationList":[{
      "address":"0x35202a6E6317F3CC3a177EeEE562D3BcDA4a6FcC",
      "chainId":"0xde9fb",
      "nonce":"0x0",
      "r":"0xc9e930f2051c331b994ca1ec5d398379923dc1aae3b1aaaccf4242e872b8ce79",
      "s":"0x182672f89807a836962b0f29ce566f30b1588bd8b6bd4f9a8e24b3675ed11c3e",
      "yParity":"0x0"
    }],
    "data":"0xdeadbeef00000000000000000000000000000000000000000000000000000000cafebabe",
    "to":"0xFC7b76a8cA893f00976a14559D08b77aa4e4Bf2e"
  }]
}'
```
#### EXECUTING A CALL TO A DELEGATED EOA
<img width="649" alt="image" src="https://github.com/user-attachments/assets/b0da9d47-b12f-49bd-ad71-61dbefa5e091" />

odyssey_sendTransaction request that executes an EIP-1559 Transaction to execute a function on a delegated EOA (via data):
```
curl 'https://odyssey.ithaca.xyz/' --data '{
  "jsonrpc":"2.0",
  "id":0,
  "method":"odyssey_sendTransaction",
  "params":[{
    "data":"0xdeadbeef00000000000000000000000000000000000000000000000000000000cafebabe",
    "to":"0xFC7b76a8cA893f00976a14559D08b77aa4e4Bf2e"
  }]
}'
```
#### SEQUENCER
Sequencer 應僅處理符合以下條件的交易：

- 為 EIP-7702 類型，且將智慧合約指定給一個 EOA（外部擁有帳戶），或為 EIP-1559 類型且指定至 EOA；

- 不超過 Sequencer 定義的每筆交易 gas 上限

- 不包含 value 欄位（或其值設為 0）

- 不包含 nonce 欄位（由 Sequencer 管理）

- 不包含 from 欄位（由 Sequencer 管理）

### 2025.05.15
#### Nonce 增加機制
EIP-7702 提出一套全新的機制，每次成功授權後就會增加 EOA（外部持有帳戶）的 nonce 值。處理 nonce 的流程是：

首先，檢查交易的 nonce 是否跟帳戶目前的 nonce 相符，然後把交易 nonce 加一
接著檢查每個授權清單中的 nonce 確認用的是正確的數值，成功授權後相對應的授權 nonce 也會加一

這代表如果一筆交易包含授權清單，而且授權簽署者跟交易簽署者是同一人，你就必須把：

tx.nonce 設定為 account.nonce
authorization.nonce 設定為 account.nonce + 1

若一筆交易包含多個由同一個 EOA 簽署的授權（雖然實際上很少見啦），每個授權的 nonce 必須按順序一個一個加上去。總之，這個機制需要 SDK 正確處理 nonce 值。

#### 授權清單規範與委託機制
EIP-7702 要求 authorization_list 不能是空的，確保每筆 EIP-7702 交易都明確表達要授權某個實作地址的意圖。協議還會檢查每個 authorization_list 元組的參數，包括：

address 長度
chain_id、nonce、y_parity、r 和 s 的合理範圍

在 authorization_list 的每個元組中：

address 欄位是被授權的地址
簽署者的地址則是從簽名和資料中算出來的

ps1: 贊助交易：這種設計讓 EIP-7702 能把支付 Gas 費的責任從被授權的 EOA 轉移出去，實現了大家常說的「贊助交易」功能

ps2:Layer 2 整合：
Layer 2 的Sequencer可以直接整合 EIP-7702 錢包建立介面
也可以加入 API 來收集使用者授權並用 authorization_list 批次提交

ps3:錢包服務提供商的應用：錢包服務商可以幫沒幣的使用者送出交易！

### 2025.05.17

#### 錢包服務
User 在使用錢包服務時最大的進入障礙往往是註記詞、私鑰的問題，在passkey普及的狀況下，所有需要密碼登入的服務都紛紛採用無密碼登入的方式，作為開發者也很期待把這格功能在web3錢包上實現，但是在7702還沒出來之前，使用4337搭配Passkey的實作仍不盡理想，沒有經驗的使用者使用的合約還是會受第三方制約，不夠去中心化，而EOA上搭載合約德發生，大大提升了可用性，使用者可以完全自託管錢包，同樣可以在無密碼的場景下使用合約錢包，另外gas station 的服務不在指定原生幣作為燃料，降低初學者的使用門檻，也讓服務保有很大的彈性。

<!-- Content_END -->
