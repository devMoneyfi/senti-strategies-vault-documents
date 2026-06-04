# Withdraw Flow — MoneyFi / Senti Trading Vault

## 1. Mục tiêu

Withdraw Flow dùng để đưa vốn từ Exness/Senti trading account về lại user wallet.

Có 2 loại withdraw:

```text
1. Withdraw Full
   User rút toàn bộ vốn, vault đóng.

2. Withdraw Partial
   User rút một phần vốn, vault vẫn tiếp tục active.
```

Dòng tiền thực tế:

```text
Exness Sub Account
→ Treasury / Multisig Wallet
→ User Wallet
```

Trading flow:

```text
User request withdraw
→ Bot pause / reduce / close positions
→ Backend settle PnL & fees
→ Admin withdraw từ Exness về treasury
→ Multisig payout cho user
```

---

## 2. Actors

### User

- Request withdraw full hoặc partial.
- Xem estimated amount nhận được.
- Confirm withdraw.
- Nhận payout on-chain.

### Vault Backend

- Validate withdraw request.
- Lock shares/balance tương ứng.
- Command bot pause/reduce/close positions.
- Sync MT5 equity/balance.
- Tính PnL, performance fee, net withdrawable.
- Tạo Admin Task withdraw từ Exness.
- Monitor treasury nhận tiền.
- Tạo multisig payout transaction.
- Update vault/withdrawal status.

### Admin Ops

- Vào Exness PA.
- Tạo withdrawal request từ Exness về treasury/multisig.
- Submit withdrawal proof về backend.
- Confirm withdrawal bằng OTP/security nếu cần.

### Senti / MT5 Bot

- Pause mở lệnh mới.
- Close all positions cho full withdraw.
- Reduce positions cho partial withdraw.
- Resume trading nếu partial withdraw hoàn tất.

### Multisig Signers

- Review payout transaction.
- Approve payout.
- Execute payout về user wallet.

---

## 3. Withdraw Full — User-facing Flow

```text
1. User chọn Withdraw Full.
2. App hiển thị:
   - Current Equity
   - Unrealized PnL
   - Estimated Performance Fee
   - Estimated Net Receive
   - Estimated Processing Status
3. User confirm withdraw.
4. App hiển thị timeline:
   - Withdrawal Requested
   - Strategy Paused
   - Positions Closed
   - Settlement Processing
   - Transfer Back Processing
   - Completed
5. User nhận token về wallet.
6. Vault status = CLOSED.
```

---

## 4. Withdraw Full — Internal Flow

### Step 1 — Create Withdraw Request

```http
POST /vaults/{vaultId}/withdraw-full
```

Request:

```json
{
  "userWallet": "0xUserWallet",
  "vaultId": "vault_uuid"
}
```

Backend validate:

```text
- vault tồn tại
- vault owner đúng user
- vault status = ACTIVE
- không có withdraw request pending
- mt5AccountId đã linked
- active strategy đang RUNNING hoặc PAUSED
```

Create withdrawal record:

```json
{
  "withdrawalId": "wd_uuid",
  "vaultId": "vault_uuid",
  "type": "FULL",
  "status": "WITHDRAW_REQUESTED",
  "requestedAt": "2026-06-04T10:00:00.000Z"
}
```

Vault status:

```text
WITHDRAW_REQUESTED
```

---

### Step 2 — Lock Vault Shares / Balance

Backend lock toàn bộ shares/balance của user.

```text
lockedShares = userVaultShares
lockedMode = FULL_WITHDRAW
```

Mục tiêu:

```text
- User không thể deposit thêm trong lúc withdraw full.
- User không thể tạo withdraw request thứ hai.
- Vault không bị chuyển strategy trong lúc closing.
```

---

### Step 3 — Stop New Positions

Backend gửi command cho Senti/MT5 Bot:

```http
POST /operations/control-active-ea
```

Request đề xuất:

```json
{
  "activeEaId": "active_ea_uuid",
  "command": "PAUSE_NEW_POSITION",
  "reason": "USER_WITHDRAW_FULL"
}
```

Vault status:

```text
CLOSING_POSITIONS
```

User-facing status:

```text
Strategy Paused
```

Nếu API này chưa có, Admin/Bot operator cần pause thủ công trong Senti/MT5.

---

### Step 4 — Close All Open Positions

Backend gửi command:

```json
{
  "activeEaId": "active_ea_uuid",
  "command": "CLOSE_ALL_POSITIONS",
  "reason": "USER_WITHDRAW_FULL"
}
```

Bot xử lý:

```text
1. Không mở position mới.
2. Close toàn bộ open positions.
3. Confirm không còn exposure.
4. Emit event hoặc backend poll positions.
```

Vault status:

```text
POSITIONS_CLOSED
```

User-facing status:

```text
Positions Closed
```

---

### Step 5 — Sync Final MT5 Equity / Balance

Backend gọi realtime account API:

```http
GET https://rtapi.sentitrade.xyz/tm/{mt5AccountId}/v1/account
```

Backend lấy:

```text
BALANCE
EQUITY
PROFIT
MARGIN
MARGIN_FREE
TRADE_ALLOWED
```

Validation:

```text
- Không còn open position.
- PROFIT gần bằng 0 nếu toàn bộ position đã đóng.
- EQUITY gần bằng BALANCE.
- MARGIN = 0 hoặc rất thấp.
```

Snapshot:

```json
{
  "withdrawalId": "wd_uuid",
  "finalBalance": "1236.82",
  "finalEquity": "1236.82",
  "currency": "USD",
  "syncedAt": "2026-06-04T10:10:00.000Z"
}
```

---

### Step 6 — Settle PnL and Fees

Backend tính:

```text
initialDeposit
currentEquity
realizedPnl
highWaterMark
performanceFee
netWithdrawable
```

Performance fee rule:

```text
Management Fee = 0%
Performance Fee = 20% of net profits above High Water Mark
```

Formula:

```text
profitAboveHwm = currentEquity - highWaterMark

if profitAboveHwm > 0:
    performanceFee = profitAboveHwm * 20%
else:
    performanceFee = 0

netWithdrawable = currentEquity - performanceFee
```

Example:

```text
Initial Deposit = $10,000
Current Equity = $12,000
Profit = $2,000
Performance Fee = $400
Net Withdrawable = $11,600
```

Vault status:

```text
SETTLEMENT_COMPLETED
```

User-facing status:

```text
Settlement Processing
```

---

### Step 7 — Create Admin Task: WITHDRAW_FROM_EXNESS

Backend tạo Admin Task để rút tiền từ Exness về treasury.

```json
{
  "taskType": "WITHDRAW_FROM_EXNESS",
  "withdrawalId": "wd_uuid",
  "vaultId": "vault_uuid",
  "exnessAccountId": "exness_account_uuid",
  "amount": "11600",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "withdrawalAddress": "0xTreasuryMultisig",
  "status": "PENDING"
}
```

Vault status:

```text
WAITING_EXNESS_WITHDRAWAL
```

---

### Step 8 — Admin Withdraws from Exness to Treasury

Admin xử lý trong Exness PA:

```text
1. Mở Exness PA.
2. Chọn Withdrawal.
3. Chọn token/network.
4. Chọn đúng Exness sub-account của vault.
5. Nhập amount cần rút.
6. Nhập treasury/multisig wallet address.
7. Kiểm tra thông tin.
8. Confirm withdrawal bằng OTP/security nếu cần.
9. Submit proof về backend.
```

Admin submit:

```json
{
  "taskId": "task_uuid",
  "exnessAccountId": "exness_account_uuid",
  "amount": "11600",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "withdrawalAddress": "0xTreasuryMultisig",
  "paymentId": "optional",
  "screenshotUrl": "optional"
}
```

Backend validate:

```text
- withdrawalAddress là treasury allowlisted address
- amount khớp final settlement
- token/network đúng
- Exness account đúng vault
```

---

### Step 9 — Monitor Treasury Receipt

Backend monitor treasury wallet.

Matching rules:

```text
tx.to == treasuryAddress
tx.token == expectedToken
tx.network == expectedNetwork
tx.amount >= expectedAmount hoặc nằm trong accepted tolerance
tx chưa được dùng trước đó
```

Khi nhận tiền:

```json
{
  "withdrawalId": "wd_uuid",
  "treasuryReceiveTxHash": "0xTreasuryReceiveTxHash",
  "receivedAmount": "11600",
  "receivedAt": "2026-06-04T10:30:00.000Z"
}
```

Vault status:

```text
TREASURY_FUNDED
```

User-facing status:

```text
Transfer Back Processing
```

---

### Step 10 — Create Multisig Payout to User

Backend tạo payout tx:

```text
Treasury / Multisig Wallet
→ User Wallet
```

Payout proposal:

```json
{
  "withdrawalId": "wd_uuid",
  "vaultId": "vault_uuid",
  "from": "0xTreasuryMultisig",
  "to": "0xUserWallet",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "amount": "11600",
  "purpose": "USER_WITHDRAW_FULL_PAYOUT"
}
```

Vault status:

```text
PAYOUT_PENDING
```

---

### Step 11 — Signers Approve and Execute Payout

Multisig signers review:

```text
- user wallet
- amount
- token/network
- withdrawal reference
- fee calculation snapshot
```

Sau khi approve:

```text
Execute payout transaction
```

Backend lưu:

```json
{
  "withdrawalId": "wd_uuid",
  "payoutTxHash": "0xPayoutTxHash",
  "status": "PAID"
}
```

Final statuses:

```text
withdrawal.status = PAID
vault.status = CLOSED
exnessAccount.status = AVAILABLE hoặc ARCHIVED
```

User-facing status:

```text
Completed
```

---

## 5. Withdraw Partial — User-facing Flow

```text
1. User chọn Withdraw Partial.
2. User nhập amount muốn rút.
3. App hiển thị:
   - Requested Amount
   - Estimated Fee
   - Estimated Net Receive
   - Remaining Equity
   - Impact on Strategy
4. User confirm.
5. Bot reduce positions nếu cần.
6. Admin rút phần tiền cần thiết từ Exness về treasury.
7. Multisig payout về user.
8. Vault vẫn ACTIVE.
```

---

## 6. Withdraw Partial — Internal Flow

### Step 1 — Create Partial Withdraw Request

```http
POST /vaults/{vaultId}/withdraw-partial
```

Request:

```json
{
  "userWallet": "0xUserWallet",
  "vaultId": "vault_uuid",
  "amount": "300"
}
```

Backend validate:

```text
- vault status = ACTIVE
- amount > 0
- amount <= available withdrawable equity
- remaining equity >= min vault balance nếu có rule
- không có pending withdrawal khác
```

Create withdrawal record:

```json
{
  "withdrawalId": "wd_uuid",
  "vaultId": "vault_uuid",
  "type": "PARTIAL",
  "requestedAmount": "300",
  "status": "PARTIAL_WITHDRAW_REQUESTED"
}
```

Vault status:

```text
PARTIAL_WITHDRAW_REQUESTED
```

---

### Step 2 — Lock Partial Shares / Balance

Backend lock phần shares tương ứng.

```text
lockedShares = requestedAmount / currentSharePrice
lockedMode = PARTIAL_WITHDRAW
```

---

### Step 3 — Reduce Position or Use Free Balance

Backend quyết định mode:

```text
If free balance đủ:
    Không cần reduce positions.
else:
    Bot reduce positions theo tỷ lệ cần rút.
```

Command đề xuất:

```json
{
  "activeEaId": "active_ea_uuid",
  "command": "REDUCE_POSITION",
  "amount": "300",
  "reason": "USER_WITHDRAW_PARTIAL"
}
```

Vault status:

```text
REDUCING_POSITIONS
```

Sau khi reduce xong:

```text
POSITIONS_REDUCED
```

---

### Step 4 — Sync MT5 Equity / Balance

Backend gọi:

```http
GET https://rtapi.sentitrade.xyz/tm/{mt5AccountId}/v1/account
```

Validate:

```text
- equity đủ cho requested withdraw
- margin level an toàn
- exposure sau reduce hợp lệ
```

---

### Step 5 — Settle Partial PnL and Fee

Với partial withdraw, không nên charge toàn bộ performance fee của vault nếu user chỉ rút một phần.

Recommended approach:

```text
1. Tính profitAboveHwm.
2. Tính tỷ lệ rút: withdrawAmount / currentEquity.
3. Fee chỉ áp dụng trên phần profit tương ứng với amount được rút.
```

Formula:

```text
profitAboveHwm = max(currentEquity - highWaterMark, 0)
withdrawRatio = requestedWithdrawAmount / currentEquity
feeBase = profitAboveHwm * withdrawRatio
performanceFee = feeBase * 20%
netPayout = requestedWithdrawAmount - performanceFee
```

---

### Step 6 — Create Admin Task: PARTIAL_WITHDRAW_FROM_EXNESS

```json
{
  "taskType": "PARTIAL_WITHDRAW_FROM_EXNESS",
  "withdrawalId": "wd_uuid",
  "vaultId": "vault_uuid",
  "exnessAccountId": "exness_account_uuid",
  "amount": "300",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "withdrawalAddress": "0xTreasuryMultisig",
  "status": "PENDING"
}
```

Vault status:

```text
WAITING_PARTIAL_EXNESS_WITHDRAWAL
```

---

### Step 7 — Admin Withdraws Partial Amount to Treasury

Admin thực hiện tương tự withdraw full nhưng chỉ rút amount cần thiết.

```text
Exness Sub Account
→ Treasury / Multisig Wallet
```

Backend monitor treasury receipt.

Khi nhận tiền:

```text
vault.status = TREASURY_FUNDED_FOR_PARTIAL_WITHDRAW
```

---

### Step 8 — Multisig Payout to User

Backend tạo payout:

```json
{
  "withdrawalId": "wd_uuid",
  "vaultId": "vault_uuid",
  "from": "0xTreasuryMultisig",
  "to": "0xUserWallet",
  "token": "USDT",
  "network": "BNB_CHAIN",
  "amount": "netPayout",
  "purpose": "USER_WITHDRAW_PARTIAL_PAYOUT"
}
```

Signers approve và execute.

Final statuses:

```text
withdrawal.status = PAID
vault.status = ACTIVE
account.status = TRADING
```

Nếu bot đã pause/reduce:

```http
POST /operations/control-active-ea
```

```json
{
  "activeEaId": "active_ea_uuid",
  "command": "RESUME_TRADING",
  "reason": "PARTIAL_WITHDRAW_COMPLETED"
}
```

---

## 7. Withdraw Status Machine

### Full Withdraw Statuses

```text
ACTIVE
WITHDRAW_REQUESTED
CLOSING_POSITIONS
POSITIONS_CLOSED
SETTLEMENT_PROCESSING
SETTLEMENT_COMPLETED
WAITING_EXNESS_WITHDRAWAL
WAITING_TREASURY_RECEIPT
TREASURY_FUNDED
PAYOUT_PENDING
PAID
CLOSED
```

### Partial Withdraw Statuses

```text
ACTIVE
PARTIAL_WITHDRAW_REQUESTED
REDUCING_POSITIONS
POSITIONS_REDUCED
PARTIAL_SETTLEMENT_PROCESSING
WAITING_PARTIAL_EXNESS_WITHDRAWAL
WAITING_TREASURY_RECEIPT
TREASURY_FUNDED_FOR_PARTIAL_WITHDRAW
PARTIAL_PAYOUT_PENDING
PAID
ACTIVE
```

### Error / Review Statuses

```text
MANUAL_REVIEW_WITHDRAWAL
BOT_CLOSE_POSITION_FAILED
BOT_REDUCE_POSITION_FAILED
EXNESS_WITHDRAWAL_FAILED
TREASURY_RECEIPT_DELAYED
PAYOUT_TX_FAILED
FAILED
```

---

## 8. User-facing Withdrawal Timeline

```json
[
  {
    "step": "Withdrawal Requested",
    "status": "completed",
    "description": "Your withdrawal request has been received."
  },
  {
    "step": "Strategy Paused",
    "status": "in_progress",
    "description": "The strategy is being paused to prevent new positions."
  },
  {
    "step": "Positions Closed / Reduced",
    "status": "pending",
    "description": "Open positions are being closed or reduced."
  },
  {
    "step": "Settlement Processing",
    "status": "pending",
    "description": "PnL and performance fees are being calculated."
  },
  {
    "step": "Transfer Back Processing",
    "status": "pending",
    "description": "Funds are being transferred back from Exness to treasury."
  },
  {
    "step": "Completed",
    "status": "pending",
    "description": "Funds have been paid to your wallet."
  }
]
```

---

## 9. Error Cases

### Bot cannot close positions

```text
status = BOT_CLOSE_POSITION_FAILED
action = retry, admin check MT5 terminal, manual close nếu cần
```

### Bot cannot reduce positions

```text
status = BOT_REDUCE_POSITION_FAILED
action = retry hoặc manual intervention
```

### Equity changed after estimate

```text
status = SETTLEMENT_RECALC_REQUIRED
action = recalculate final withdrawable amount before Exness withdrawal
```

### Exness withdrawal pending too long

```text
status = MANUAL_REVIEW_WITHDRAWAL
action = admin verify Exness PA/payment status
```

### Treasury did not receive funds

```text
status = TREASURY_RECEIPT_DELAYED
action = monitor chain, check payment ID, admin verify Exness withdrawal
```

### Payout tx failed

```text
status = PAYOUT_TX_FAILED
action = create new multisig payout proposal after review
```

### User wallet invalid or blocked

```text
status = MANUAL_REVIEW_WITHDRAWAL
action = user support/manual resolution
```

---

## 10. Required APIs

### MoneyFi Backend APIs

```text
POST /vaults/{vaultId}/withdraw-full
POST /vaults/{vaultId}/withdraw-partial
GET  /vaults/{vaultId}/withdrawals/{withdrawalId}
POST /admin/tasks/{taskId}/submit-exness-withdrawal-info
POST /multisig/create-payout-transaction
POST /vaults/{vaultId}/settle-withdrawal
POST /vaults/{vaultId}/mark-withdrawal-paid
```

### Senti APIs Available

```text
POST /operations/get-active-eas
GET  /tm/{mt5AccountId}/v1/account
POST /operations/get-deals-from-warehouse
POST /operations/get-sync-state-for-account
POST /operations/trigger-sync-for-account
```

### Senti APIs Needed

```text
POST /operations/control-active-ea
POST /operations/get-open-positions
POST /operations/close-all-positions
POST /operations/reduce-position
POST /operations/get-vault-active-strategies
```

---

## 11. Final Withdraw Full Summary

```text
User chọn Withdraw Full
→ Backend validate request
→ Backend lock all shares
→ Backend command bot PAUSE_NEW_POSITION
→ Backend command bot CLOSE_ALL_POSITIONS
→ Backend sync final MT5 equity/balance
→ Backend settle PnL + performance fee
→ Backend tạo Admin Task WITHDRAW_FROM_EXNESS
→ Admin rút tiền từ Exness về treasury/multisig
→ Backend monitor treasury receipt
→ Backend tạo multisig payout cho user
→ Signers approve
→ Execute payout
→ withdrawal.status = PAID
→ vault.status = CLOSED
```

---

## 12. Final Withdraw Partial Summary

```text
User chọn Withdraw Partial
→ Backend validate amount
→ Backend lock partial shares
→ Backend check free balance / exposure
→ Bot reduce positions nếu cần
→ Backend sync MT5 equity/balance
→ Backend settle partial PnL + fee
→ Backend tạo Admin Task PARTIAL_WITHDRAW_FROM_EXNESS
→ Admin rút phần cần thiết từ Exness về treasury
→ Backend monitor treasury receipt
→ Backend tạo multisig payout cho user
→ Signers approve
→ Execute payout
→ withdrawal.status = PAID
→ vault.status = ACTIVE
→ Bot resume trading nếu đã pause
```
