# Deposit Flow — MoneyFi / Senti Trading Vault

## 1. Mục tiêu

Deposit Flow dùng để đưa vốn của user từ ví on-chain vào hệ thống trading:

```text
User Wallet
→ Treasury / Multisig Wallet
→ Exness Sub Account
→ Senti Trading Engine
→ Trading Strategy Active
```

User-facing flow cần đơn giản:

```text
Select Vault
→ Risk Acknowledgement
→ Deposit USDT/USDC
→ Vault Active
```

Internal flow sẽ bao gồm Backend, Admin Ops, Exness PA, Multisig Signers và Senti / MT5 Bot.

---

## 2. Actors

### User

- Chọn vault.
- Xác nhận rủi ro.
- Deposit token on-chain.
- Theo dõi trạng thái kích hoạt vault.

### Vault Backend

- Tạo deposit intent.
- Detect on-chain deposit.
- Validate token, network, amount, sender.
- Assign Exness account từ account pool.
- Tạo Admin Task.
- Tạo multisig funding transaction.
- Sync MT5 balance/equity.
- Link account với Senti.
- Deploy strategy.
- Update vault status.

### Admin Ops

- Vào Exness PA.
- Lấy deposit instruction/address.
- Submit deposit info về backend.
- Upload invoice/screenshot/payment ID nếu cần.

### Multisig Signers

- Review funding transaction.
- Approve transaction.
- Execute transaction.

### Senti / MT5 Bot

- Link account.
- Deploy EA/strategy.
- Confirm strategy RUNNING.
- Start trading.

---

## 3. User-facing Deposit Flow

```text
1. User chọn vault:
   - Conservative
   - Balanced
   - Aggressive

2. User xem thông tin vault:
   - Target APR
   - Risk Score
   - Historical MDD
   - Current TVL
   - Performance Fee
   - Minimum Deposit
   - Supported Token / Network

3. User tick Risk Acknowledgement.

4. User nhập amount deposit.

5. App hiển thị deposit address của Treasury / Multisig.

6. User gửi USDT/USDC on-chain.

7. App hiển thị timeline xử lý.

8. Khi Exness funded + Senti strategy running:
   - Vault status = ACTIVE
   - User thấy Vault Active.
```

---

## 4. Internal Deposit Flow — Step by Step

### Step 1 — Create Deposit Intent

User chọn vault, nhập amount và confirm deposit.

```http
POST /vaults/create-deposit-intent
```

Request:

```json
{
  "userWallet": "0xUserWallet",
  "vaultType": "SENTI_BALANCED",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "amount": "1000"
}
```

Backend validate:

```text
- userWallet hợp lệ
- vaultType hợp lệ
- token được support
- network được support
- amount >= minDeposit
- user đã Risk Acknowledgement
```

Response:

```json
{
  "depositIntentId": "dep_intent_uuid",
  "vaultId": "vault_uuid",
  "status": "WAITING_USER_DEPOSIT",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "amount": "1000",
  "depositAddress": "0xTreasuryMultisig",
  "expiresAt": "2026-06-04T11:00:00.000Z"
}
```

Vault status:

```text
WAITING_USER_DEPOSIT
```

---

### Step 2 — User Deposit On-chain

User gửi token vào Treasury / Multisig Wallet.

```text
User Wallet
→ Treasury / Multisig Wallet
```

Expected data:

```text
token = expected token
network = expected network
amount >= expected amount
to = treasury/multisig address
from = user wallet
```

---

### Step 3 — Backend Detect Deposit

Backend monitor blockchain.

Matching rules:

```text
tx.to == treasuryAddress
tx.from == userWallet
tx.token == expectedToken
tx.network == expectedNetwork
tx.amount >= expectedAmount
txHash chưa từng được dùng
tx đủ số confirmation yêu cầu
```

Khi hợp lệ:

```json
{
  "depositIntentId": "dep_intent_uuid",
  "txHash": "0xUserDepositTxHash",
  "confirmedAmount": "1000",
  "confirmedAt": "2026-06-04T10:05:00.000Z",
  "status": "CONFIRMED"
}
```

Update:

```text
deposit.status = CONFIRMED
vault.status = USER_DEPOSIT_CONFIRMED
```

User-facing status:

```text
Deposit Confirmed
```

---

### Step 4 — Assign Exness Account

Backend lấy Exness account từ account pool.

```text
1. Query account_pool.
2. Tìm account status = AVAILABLE.
3. Assign account cho vault.
4. Update account status = ASSIGNED.
5. Update vault status = EXNESS_ACCOUNT_ASSIGNED.
```

Account statuses:

```text
AVAILABLE
ASSIGNED
WAITING_FUNDING
FUNDED
TRADING
ARCHIVED
ERROR
```

Nếu không có account available:

```text
vault.status = WAITING_EXNESS_ACCOUNT
adminTask.type = CREATE_EXNESS_ACCOUNT
```

Khuyến nghị MVP: chuẩn bị sẵn account pool để tránh delay.

---

### Step 5 — Create Admin Task: FUND_EXNESS_ACCOUNT

Backend tạo task cho Admin Ops.

```json
{
  "taskType": "FUND_EXNESS_ACCOUNT",
  "vaultId": "vault_uuid",
  "userId": "user_uuid",
  "exnessAccountId": "exness_account_uuid",
  "amount": "1000",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "status": "PENDING"
}
```

Vault status:

```text
WAITING_EXNESS_DEPOSIT_INFO
```

User-facing status:

```text
Preparing Trading Account
```

---

### Step 6 — Admin Gets Exness Deposit Instruction

Admin xử lý trong Exness PA:

```text
1. Mở Exness PA.
2. Chọn đúng Exness sub-account của vault.
3. Chọn Deposit.
4. Chọn token/network.
5. Nhập amount cần nạp.
6. Exness tạo deposit instruction/address.
7. Admin submit deposit info về backend.
```

Admin submit:

```json
{
  "taskId": "task_uuid",
  "exnessAccountId": "exness_account_uuid",
  "amount": "1000",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "exnessDepositAddress": "0xExnessDepositAddress",
  "paymentId": "optional",
  "screenshotUrl": "optional"
}
```

---

### Step 7 — Backend Validate Exness Deposit Info

Backend validate:

```text
- token đúng
- network đúng
- amount đúng
- deposit address hợp lệ
- task chưa expired
- vault đúng trạng thái
- Exness account đúng vault
```

Nếu hợp lệ:

```text
adminTask.status = INFO_SUBMITTED
vault.status = EXNESS_DEPOSIT_INFO_SUBMITTED
```

Nếu lỗi:

```text
adminTask.status = NEED_REVISION
vault.status = MANUAL_REVIEW_FUNDING
```

---

### Step 8 — Create Multisig Funding Transaction

Backend tạo transaction proposal:

```text
Treasury / Multisig Wallet
→ Exness Deposit Address
```

Proposal:

```json
{
  "vaultId": "vault_uuid",
  "taskId": "task_uuid",
  "from": "0xTreasuryMultisig",
  "to": "0xExnessDepositAddress",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "amount": "1000",
  "purpose": "FUND_EXNESS_ACCOUNT"
}
```

Vault status:

```text
MULTISIG_TX_PENDING
```

User-facing status:

```text
Fund Transfer Processing
```

---

### Step 9 — Multisig Approval and Execution

Signers review:

```text
- destination address
- amount
- token/network
- vault reference
- admin task reference
```

Sau khi đủ chữ ký:

```text
Execute transaction
```

Backend lưu:

```json
{
  "multisigTxId": "safe_tx_uuid",
  "txHash": "0xFundingTxHash",
  "status": "EXECUTED"
}
```

Vault status:

```text
MULTISIG_TX_EXECUTED
```

---

### Step 10 — Wait for Exness Credit

Backend monitor funding tx và chờ Exness credit vào MT5 account.

```text
1. Confirm funding tx on-chain.
2. Save funding tx hash.
3. Poll / sync MT5 balance/equity.
4. Verify Exness balance/equity increased.
```

Vault status:

```text
WAITING_EXNESS_CREDIT
```

User-facing status:

```text
Waiting for Trading Account Funding
```

---

### Step 11 — Sync MT5 Balance / Equity

Backend gọi Senti RTAPI:

```http
GET https://rtapi.sentitrade.xyz/tm/{mt5AccountId}/v1/account
```

Expected fields:

```text
BALANCE
PROFIT
EQUITY
MARGIN
MARGIN_FREE
MARGIN_LEVEL
TRADE_ALLOWED
LOGIN
SERVER
CURRENCY
```

Validation:

```text
- login đúng Exness account
- server đúng
- trade allowed = true
- balance/equity >= expected funded amount
- currency đúng hoặc convert đúng
```

Nếu là Exness Cent account:

```text
USC / 100 = USD display
```

Nếu pass:

```text
vault.status = EXNESS_FUNDED
account.status = FUNDED
```

---

### Step 12 — Link Account with Senti

Đây là API Senti cần bổ sung nếu muốn automation hoàn chỉnh.

Proposed API:

```http
POST /operations/link-mt5-account
```

Request:

```json
{
  "vaultId": "vault_uuid",
  "mt5AccountId": "mt5_account_uuid",
  "exnessLogin": "257424789",
  "broker": "Exness",
  "server": "Exness-MT5Real36"
}
```

Response:

```json
{
  "success": true,
  "vaultId": "vault_uuid",
  "mt5AccountId": "mt5_account_uuid",
  "status": "LINKED"
}
```

Vault status:

```text
SENTI_LINKED
```

Nếu chưa có API:

```text
Admin manual link account trong Senti Dashboard.
Backend lưu mapping mt5AccountId vào vault.
```

---

### Step 13 — Deploy Strategy

Backend deploy strategy theo vault type.

Proposed API:

```http
POST /operations/deploy-ea-to-account
```

Request:

```json
{
  "vaultId": "vault_uuid",
  "mt5AccountId": "mt5_account_uuid",
  "eaDefinitionId": "ea_definition_uuid",
  "presetId": "preset_uuid",
  "symbol": "XAUUSD",
  "timeframe": "M1",
  "inputs": {
    "MagicNumber": 35,
    "BaseLot": 0.01
  }
}
```

Vault status:

```text
STRATEGY_DEPLOYING
```

After success:

```text
STRATEGY_DEPLOYED
```

---

### Step 14 — Confirm Active Strategy

Backend confirm active EA:

```http
POST /operations/get-active-eas
```

Validation:

```text
activeEA.mt5AccountId == vault.mt5AccountId
activeEA.status == RUNNING
activeEA.symbol đúng
activeEA.timeframe đúng
```

Nếu pass:

```text
vault.status = ACTIVE
account.status = TRADING
```

User-facing status:

```text
Vault Active
```

---

## 5. Deposit Status Machine

### Internal Vault Status

```text
DRAFT
WAITING_USER_DEPOSIT
USER_DEPOSIT_DETECTED
USER_DEPOSIT_CONFIRMED
WAITING_EXNESS_ACCOUNT
EXNESS_ACCOUNT_ASSIGNED
WAITING_EXNESS_DEPOSIT_INFO
EXNESS_DEPOSIT_INFO_SUBMITTED
MULTISIG_TX_PENDING
MULTISIG_TX_EXECUTED
WAITING_EXNESS_CREDIT
EXNESS_FUNDED
SENTI_LINKING
SENTI_LINKED
STRATEGY_DEPLOYING
STRATEGY_DEPLOYED
ACTIVE
MANUAL_REVIEW_DEPOSIT
MANUAL_REVIEW_FUNDING
STRATEGY_DEPLOY_FAILED
FAILED
```

### User-facing Status

```text
Waiting for Deposit
Deposit Confirmed
Preparing Trading Account
Fund Transfer Processing
Linking Senti Account
Deploying Strategy
Vault Active
```

---

## 6. Deposit UI Timeline

```json
[
  {
    "step": "Deposit Confirmed",
    "status": "completed",
    "description": "Your on-chain deposit has been confirmed."
  },
  {
    "step": "Trading Account Preparing",
    "status": "in_progress",
    "description": "We are preparing your Exness trading account."
  },
  {
    "step": "Fund Transfer Processing",
    "status": "pending",
    "description": "Funds will be transferred from treasury to the trading account."
  },
  {
    "step": "Senti Linked",
    "status": "pending",
    "description": "Your trading account will be linked to Senti engine."
  },
  {
    "step": "Strategy Deployment",
    "status": "pending",
    "description": "Trading strategy will be deployed based on your selected vault."
  },
  {
    "step": "Vault Active",
    "status": "pending",
    "description": "Your vault is active and trading has started."
  }
]
```

---

## 7. Error Cases

### User deposit sai token

```text
status = MANUAL_REVIEW_DEPOSIT
action = Admin review, refund hoặc credit thủ công
```

### User deposit sai network

```text
status = MANUAL_REVIEW_DEPOSIT
action = recover nếu được, nếu không thì manual support
```

### User deposit thiếu amount

```text
status = PARTIAL_DEPOSIT_DETECTED
action = cho user nạp thêm, refund, hoặc tạo vault nếu amount >= minDeposit
```

### Duplicate tx hash

```text
status = REJECTED_DUPLICATE_TX
action = không credit lần hai
```

### Exness deposit address expired

```text
status = MANUAL_REVIEW_FUNDING
action = admin tạo lại deposit instruction
```

### Multisig tx failed

```text
status = MULTISIG_TX_FAILED
action = backend tạo proposal mới sau khi review
```

### Exness chưa credit

```text
status = WAITING_EXNESS_CREDIT hoặc MANUAL_REVIEW_FUNDING
action = continue monitoring, admin verify Exness PA
```

### Strategy deploy failed

```text
status = STRATEGY_DEPLOY_FAILED
action = retry deploy, admin check Senti/MT5 terminal
```

---

## 8. Required APIs

### MoneyFi Backend APIs

```text
POST /vaults/create-deposit-intent
GET  /vaults/{vaultId}/deposit-status
POST /vaults/{vaultId}/confirm-deposit
POST /admin/tasks/{taskId}/submit-exness-deposit-info
POST /admin/tasks/{taskId}/approve-review
POST /multisig/create-funding-transaction
POST /vaults/{vaultId}/activate
```

### Senti APIs Available

```text
POST /operations/get-ea-catalog
POST /operations/get-mt5-accounts
POST /operations/get-active-eas
GET  /tm/{mt5AccountId}/v1/account
```

### Senti APIs Needed

```text
POST /operations/link-mt5-account
POST /operations/deploy-ea-to-account
POST /operations/get-vault-active-strategies
POST /operations/control-active-ea
```

---

## 9. Final Deposit Flow Summary

```text
User chọn vault
→ User risk acknowledgement
→ Backend create deposit intent
→ User deposit USDT/USDC vào treasury/multisig
→ Backend detect on-chain tx
→ Backend confirm deposit
→ Backend assign Exness account
→ Backend tạo Admin Task: FUND_EXNESS_ACCOUNT
→ Admin lấy Exness deposit instruction
→ Admin submit deposit info
→ Backend validate info
→ Backend tạo multisig transfer sang Exness deposit address
→ Signers approve
→ Execute funding tx
→ Backend monitor tx confirmation
→ Backend sync MT5 balance/equity
→ Exness funded
→ Backend link account với Senti
→ Backend deploy strategy
→ Backend confirm EA RUNNING
→ Vault ACTIVE
```
