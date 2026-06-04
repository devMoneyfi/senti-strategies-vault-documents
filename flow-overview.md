# Flow Overview — MoneyFi / Senti Trading Vault

## 1. Architecture

```text
User App
→ Vault Backend
→ Admin Ops Queue
→ Exness PA
→ Senti / MT5 Bot
→ Multisig Treasury Wallet
```

## 2. Core Principle

User-facing experience should feel simple and automated:

```text
Choose Vault
→ Deposit
→ Trading Active
→ Withdraw when needed
```

Internal operation remains controlled and auditable:

```text
Backend status machine
+ Admin Ops for Exness PA
+ Multisig approval for real fund movement
+ Senti / MT5 Bot for strategy execution
```

---

## 3. Deposit Flow Summary

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
→ Backend sync MT5 balance/equity
→ Backend link account với Senti
→ Backend deploy strategy
→ Backend confirm EA RUNNING
→ Vault ACTIVE
```

User-facing statuses:

```text
Waiting for Deposit
→ Deposit Confirmed
→ Preparing Trading Account
→ Fund Transfer Processing
→ Linking Senti Account
→ Deploying Strategy
→ Vault Active
```

---

## 4. Withdraw Full Flow Summary

```text
User chọn Withdraw Full
→ Backend validate request
→ Backend lock all shares
→ Backend command bot PAUSE_NEW_POSITION
→ Backend command bot CLOSE_ALL_POSITIONS
→ Backend sync final MT5 equity/balance
→ Backend settle PnL + performance fee
→ Backend tạo Admin Task: WITHDRAW_FROM_EXNESS
→ Admin rút tiền từ Exness về treasury/multisig
→ Backend monitor treasury receipt
→ Backend tạo multisig payout cho user
→ Signers approve
→ Execute payout
→ withdrawal.status = PAID
→ vault.status = CLOSED
```

User-facing statuses:

```text
Withdrawal Requested
→ Strategy Paused
→ Positions Closed
→ Settlement Processing
→ Transfer Back Processing
→ Completed
```

---

## 5. Withdraw Partial Flow Summary

```text
User chọn Withdraw Partial
→ Backend validate amount
→ Backend lock partial shares
→ Backend check free balance / exposure
→ Bot reduce positions nếu cần
→ Backend sync MT5 equity/balance
→ Backend settle partial PnL + fee
→ Backend tạo Admin Task: PARTIAL_WITHDRAW_FROM_EXNESS
→ Admin rút phần cần thiết từ Exness về treasury
→ Backend monitor treasury receipt
→ Backend tạo multisig payout cho user
→ Signers approve
→ Execute payout
→ withdrawal.status = PAID
→ vault.status = ACTIVE
→ Bot resume trading nếu đã pause
```

---

## 6. Main Vault Statuses

```text
WAITING_USER_DEPOSIT
USER_DEPOSIT_CONFIRMED
EXNESS_ACCOUNT_ASSIGNED
WAITING_EXNESS_DEPOSIT_INFO
MULTISIG_TX_PENDING
WAITING_EXNESS_CREDIT
EXNESS_FUNDED
SENTI_LINKED
STRATEGY_DEPLOYED
ACTIVE
WITHDRAW_REQUESTED
PARTIAL_WITHDRAW_REQUESTED
CLOSING_POSITIONS
REDUCING_POSITIONS
POSITIONS_CLOSED
WAITING_EXNESS_WITHDRAWAL
TREASURY_FUNDED
PAYOUT_PENDING
PAID
CLOSED
MANUAL_REVIEW_DEPOSIT
MANUAL_REVIEW_FUNDING
MANUAL_REVIEW_WITHDRAWAL
FAILED
```

---

## 7. Required Senti API Gaps

Current Senti APIs are enough for read-only dashboard and basic monitoring, but production vault automation still needs these write/control APIs:

```text
POST /operations/link-mt5-account
POST /operations/deploy-ea-to-account
POST /operations/control-active-ea
POST /operations/get-vault-active-strategies
POST /operations/get-account-drawdown-chart
```

Most critical for deposit and withdraw:

```text
Deposit:
- link-mt5-account
- deploy-ea-to-account

Withdraw:
- control-active-ea
- close-all-positions
- reduce-position
```

---

## 8. MVP Recommendation

For MVP, keep Exness-related and fund movement steps semi-manual:

```text
Admin manually handles Exness PA.
Backend validates and records all submitted info.
Multisig signers approve all real fund transfers.
Bot/Senti controls trading state.
```

This gives a user-facing automated experience while keeping custody, audit, and operational safety under control.
