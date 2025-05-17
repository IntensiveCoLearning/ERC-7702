---
timezone: UTC+8
---

> 请在上边的 timezone 添加你的当地时区(UTC)，这会有助于你的打卡状态的自动化更新，如果没有添加，默认为北京时间 UTC+8 时区


# 你的名字

1. 自我介绍   seashell，在工作之餘學習了 smart contract audit 兩個月，目前嘗試了兩次審計比賽跟一次 codehawk 的 first flight.
2. 你认为你会完成本次残酷学习吗？ 會
3. 你的联系方式（推荐 Telegram） https://t.me/emailpractice

## Notes

<!-- Content_START -->

### 2025.05.14

- What is EIP-7702 ?
EIP-7702 turns every account into a smart contract.
Alone, that’s useless — no user wants to write Solidity.
But with more and more "plugin" being deployed, wallets can really start to be customized :

✅ Set transfer limits to prevent hacks
✅ Pay gas with USDT or any token
✅ Batch txs, automation, allowlists...
________________________________________
- one of EIP-7702's problem 
EIP-7702 lets EOAs temporarily act like smart accounts by attaching custom contract code during a single transaction.

Under the hood, these work via DELEGATECALL.

That means: 
→ Your wallet storage is being directly modified by external contracts (plugins). 
→ Multiple plugins = multiple contracts sharing the same storage space.

This is one of the biggest blockers to EIP-7702 adoption.

Solution
 a. ERC-7201: it assigns unique storage slots to modules, avoiding collisions by design.

________________________________________
- EIP-7702 is compatible with older implementation of customize transaction ( ERC-4337 )

compatible means less pain of adopting newer EIP. that can greatly promote the usage of 7702. 

前情提要，什麼是 4337? 
同樣都要讓用戶能自訂交易邏輯。 4337 是去工廠去生出符合規範的智能合約，然後智能合約自然而然就能自訂交易邏輯 ; 有點像是EVM本身不予許自訂交易，所以要靠著一個額外的4337智能合約去當中介。 就是 4337 體系送出的交易不是讓 EVM 去驗證的，而是讓一個 bundler 去處理，總之就是盡量繞開EVM。 7702 才是真的去讓EVM接受客製交易
7702 不是靠著智能合約，而是讓 EOA自己可以在transaction的時候delegate call別人合約的程式碼，進而控制自己的交易邏輯； 
在4337的情境下，用戶為了客製交易需求，不僅需要新部屬一個智能合約，還要把 EOA 的資產轉過去智能合約。 而7702就可以方便的使用原本的帳戶來達成客製交易


而 4337 能與 7702 相容的原因是
EIP-7702 要求 EOA 在transaction的時候填寫的資料規範，與 EIP-4337 的工廠所要求的交易函數格式吻合。 所以EOA所送的交易，也能被 4337 的 bundler 處理。 
________________________________________

### 2025.05.15
auth list可以放很多個，但只有最後一個會生效。  
proxy poxy 會怎樣? 講者也還不知道

智能合約能做到的功能都能做吧，真的要思考的事情是"插件的來源"，因為目前對 proxy 合約的格式沒有規範，用戶完全可能簽到惡意合約。而且地址是可以做到仿造得很像知名協議的，如果前端的錢包應用沒有主動在簽名的時候提供一些檢查。 比如顯示這個地址是不是知名協議， 7702 對一般用戶是很難使用的，因為要檢查合約地址真的太費神了

7702 可能可以做的應用 : 
給這筆交易設一個交易門檻，交易會在 proxy 合約那邊做一些條件檢查，決定要不要revert 這筆交易

可以用 proxy 合約來製作 session key。  就是製作一個 被限縮權限，只能做一點事情的 key。

7702 不能做的應用:
自動化交易機器人。 因為transaction 不能持續監聽價格，就是得要直接地把一筆交易完成。

給自己設定一個轉帳門檻嗎 讓自己無論如何都只能轉出這麼多錢 : 
    自動化每筆交易都delegate call 一個proxy合約來為我檢查。  用其他的技術更實在，7702 就是拿來客製化"單筆"交易的
    ________________________________________
### 2025.05.16
how to implement 

參考別人的打包交易實作 https://github.com/defispartan/approve/blob/main/app/actions/Permit.tsx
但這是一個EIP 5792 交易是送給另一個合約。 到那邊才會用到 7702 的邏輯， delegate call之類的)  看不到 7702 的 delegatecall 的原因：不是自己實作 7702 的錢包邏輯。
這個repo就負責打包交易送出交易。  
他這邊好像就是送出交易給錢包而已，可能是ㄧ些主流的錢包吧 比如alchemy 或 metamask 然後他們那邊會自動判斷，如果送交易的是EOA 它就會用7702的方式去幫現在的合約進入delegate call
疑惑 : all batch supply of the repo does is send an EIP-5792 transaction, and the wallet (e.g., MetaMask) handles the 7702 implementation under the hood?
there is no 7702 implementation is this repo right?  And that is normal because it is better done by third party wallet?  

actions/BatchSupply.tsx :  

  // 一、 先打包資料

 const handleSupply = async () => {
   const approveData = encodeFunctionData({
    abi: IERC20_ABI,
    functionName: 'approve',
    args: [AaveV3Sepolia.POOL,tokenBalance]
  });
  const supplyData = encodeFunctionData({
    abi: IPool_ABI,
    functionName: 'supply',
    args: [AaveV3Sepolia.ASSETS.AAVE.UNDERLYING, tokenBalance, address || '0x0', 0]
  });
  
  // 二 send call送出交易 sendcalls 會自動調用MetaMask、或其他兼容 EIP-4337 / EIP-7702 的錢包，跳出來讓用戶簽名。  
  
  下面似乎就等同於 
  //IERC20(token).approve(poolAddress, tokenBalance);
  //pool.supply(token, amount, userAddress, 0);
  
    sendCalls({
      calls: [ {
          to: AaveV3Sepolia.ASSETS.AAVE.UNDERLYING,
          data: approveData,
        },
        {
          to: AaveV3Sepolia.POOL,
          data: supplyData,
        },],

  // 三、paymaster不知道是怎麼實作的  目前就是看到.env裡面有一個paymaster的連結 
        
        capabilities: {
          paymasterService: {
            [toHex(chainId)]: {
              optional: true,
              url: paymasterUrl,
            }
          }
      },  
  })
  }



  actions/Permit.tsx : await signTypedData({ domain, types, message })    有提供先讓使用者預先簽名的功能。 這樣使用者如果採用傳統的交易(approve再轉帳)而不是打包交易，那原本會需要用到兩次的 gas fee 。就是先approve，消耗掉一次gas。再交易 。  permit 就是在試著做到鍊下簽名: 我先取得用戶的簽名，然後之後就直接送出交易，不需要預先approve。 付出一次gas fee就好。

  note簽名的內容跟簽名要在一起，一起去生成一組 v r s 
  note 如果沒有透明顯示簽名內容，使用者以為自己簽了5 token  但實際上簽了100 token

state/TokenProvider.tsx :  從permit拿到簽名，然後儲存資料在前端瀏覽器(本地)的樣子。 簽名存在這裡，  刪除簽名資料、查看餘額 等等的功能好像也在這個檔案

___
### 2025.05.17
- 新交易格式：SET_CODE_TX_TYPE（Type = 0x04）
採用 RLP 編碼，包含以下字段（部分與 EIP-1559 相似）：

chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas, gas_limit

destination, value, data, access_list

authorization_list（新增）

簽名字段：y_parity, r, s

authorization_list 是核心
必填，非空
每個授權條目包含 6 個字段：
chain_id: 0 = 可跨鏈重放（需 Nonce 一致）
address: 被委託執行的目標合約地址
nonce: 授權者的當前帳戶 nonce
y_parity, r, s: 授權者對 (chain_id, address, nonce) + magic number 0x05 的簽名（用 keccak256）

-  一筆交易可以包含多個授權，但：
同一授權者的多個條目 → 只有最後一個會被使用
簽名驗證失敗 → 不會使交易失敗，只跳過該條目（防 DoS）

-  交易與授權分離（支持 Meta-Tx / Gas 贊助）
交易的 sender（支付 Gas 的人）與授權 signer（真正要執行的人）可以不同。

這讓第三方幫你發送交易成為可能，例如：



- 交易執行流程簡述
驗證 transaction nonce 與 sender 的帳戶匹配
遍歷 authorization_list，對每條目進行：
驗簽
檢查 nonce
nonce 驗證通過 → nonce +1

設定授權者的帳戶 code 為 0xef0100 || address
0xef0100 是委託指示符，address 是目標智能合約
若 address 為零地址，則清除 code
最後執行 destination 的 call，但會載入委託 code 執行

-  委託行為：智能 EOA
一旦授權成功，該 EOA 會被視為一個「智能 EOA」：

CALL / DELEGATECALL 等操作，會執行委託合約的代碼
CODESIZE / CODECOPY 取得的是委託合約的 code
EXTCODESIZE / EXTCODECOPY 仍作用於原 EOA

禁止多層委託（遞歸）

-  Gas 成本
內在成本：基於 EIP-2930，外加：
PER_EMPTY_ACCOUNT_COST × 授權條目數量（約 12500 Gas / 條）
即使授權失敗或重複，也會收取 Gas
若授權帳戶先前已存在 → 可部分退還 Gas（避免 DoS）

### 2025.07.12

<!-- Content_END -->
