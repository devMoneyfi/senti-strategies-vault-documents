# MoneyFi MultiVault & Senti Trading Vault PRD v2

# MONEYFI MULTI-VAULT MODEL

## 1. Vault Categories

- Stable LP Vault
- Senti Conservative Vault (~50% Target APR)
- Senti Balanced Vault (~100% Target APR)
- Senti Aggressive Vault (~200% Target APR)

## 2. Hero Metrics (Highest Priority)

- TVL
- Realtime APR
- IRR
- Max Drawdown (MDD)
- Risk Score
- Current Equity

## 3. Performance Metrics

- Realtime APR
- IRR
- All-Time Return
- 30D Return
- 90D Return
- YTD Return
- Equity Curve
- Drawdown Curve

## 4. Risk Metrics

- Max Drawdown
- Current Drawdown
- Recovery Time
- Risk Score

### Risk Mapping

| Vault Type | Risk Score |
|------------|------------|
| Conservative | 3/10 |
| Balanced | 5/10 |
| Aggressive | 8/10 |

## 5. Trading Statistics

- Win Rate
- Profit Factor
- Expectancy
- Average Win
- Average Loss
- Total Trades
- Active Strategies

## 6. Professional Metrics

- Sharpe Ratio
- Sortino Ratio
- Calmar Ratio

## 7. Strategy Information

For each strategy:

- Strategy Name
- Version
- Historical Return
- Historical MDD
- Current Allocation

### Example Strategies

- EMA Mirror
- Grid DCA
- ORB

## 8. Fee Structure

### Management Fee

- 0%

### Performance Fee

- 20% of net profits

### Display

- Gross Profit
- Performance Fee Paid
- Net Profit
- Withdrawable Balance

### Example

| Item | Value |
|--------|--------|
| Deposit | $10,000 |
| Profit | $2,000 |
| Performance Fee | $400 |
| Net Profit | $1,600 |

## 9. High Water Mark

Performance fee is only charged on profits above the previous highest account value.

## 10. Deposit Flow

1. Select Vault
2. Risk Acknowledgement
3. Deposit USDT (BNB Chain)
4. Vault Activation

### Statuses

- Deposit Confirmed
- Exness Account Created
- Senti Linked
- Strategy Deployed
- Vault Active

## 11. Withdrawal Flow

- Withdrawal Requested
- Pause Strategy
- Close Positions
- Settle PnL & Fees
- Transfer Back
- Completed

## 12. Fund Deployment Timeline

### Deposit Timeline

1. Deposit Confirmed
2. Bridge / Fund Transfer
3. Create Account
4. Link Account
5. Strategy Deployment
6. Trading Active

### Withdrawal Timeline

1. Withdrawal Requested
2. Pause Strategy
3. Close Positions
4. Settle PnL & Fees
5. Transfer Back
6. Completed

### Display

- Current Phase
- Estimated Time Remaining
- Progress Percentage
- Last Updated Timestamp

## 13. Account Structure

```text
User Wallet
└── MoneyFi Vault
    └── Exness Sub Account
        └── Senti Trading Engine
            └── Trading Strategies
```

### Display

- Sub-account ID
- Current Equity
- Current Exposure
- Status

## 14. Vault Comparison

| Vault | Target APR | Risk Score |
|---------|-----------|------------|
| Conservative | 50% | 3/10 |
| Balanced | 100% | 5/10 |
| Aggressive | 200% | 8/10 |

### Common

- Min Deposit: $1,000
- Performance Fee: 20%
- Network: BNB Chain

## 15. Product Positioning

1. Choose your risk profile.
2. Deposit capital.
3. MoneyFi handles the trading.

MoneyFi automatically creates and manages trading infrastructure through Exness and Senti while optimizing strategy allocation based on the selected risk profile.