---
name: agent-trading-skills-index
description: Master index of the SKE-Labs agent-trading-skills repository; use to route to the right trading skill by market condition, setup, or risk context.
version: 1.0.0
tags: [trading, skills, technical-analysis, crypto, risk-management, ict, day-trading]
---

# Agent Trading Skills Index

## Mục đích
Dùng skill này như cổng vào cho mọi task liên quan đến:
- trading
- chart patterns
- day trading
- crypto trading
- fundamental analysis cho trading
- ICT / smart money
- risk management
- technical analysis

## Cách dùng
1. Xác định market/regime/setup/risk context.
2. Chọn skill con phù hợp nhất từ catalog bên dưới.
3. Nếu task tổng quát, ưu tiên: `market-regime-detection`, `multi-timeframe-analysis`, rồi mới sang skill setup cụ thể.

## Skill catalog
### chart-patterns
- candlestick-patterns
- channel-trading
- cup-and-handle
- double-top-bottom
- flag-pennant
- head-and-shoulders
- triangle-patterns
- wedge-patterns

### crypto-trading
- altcoin-rotation
- arbitrage-trading
- dca-strategy
- funding-rate-trading
- on-chain-analysis

### day-trading
- breakout-trading
- gap-trading
- momentum-trading
- news-trading
- pullback-trading
- range-trading
- scalping-strategy

### fundamental-analysis
- earnings-trading
- economic-calendar-trading
- insider-activity-trading
- market-correlation-trading
- sector-rotation
- sentiment-analysis

### ict-smart-money
- breaker-blocks
- fair-value-gaps
- kill-zones
- liquidity-zones
- market-structure-shift
- optimal-trade-entry
- order-blocks
- premium-discount

### risk-management
- correlation-risk
- drawdown-management
- leverage-management
- partial-profit-taking
- position-sizing
- risk-reward-ratio
- stop-loss-strategies
- trailing-stop

### technical-strategies
- bollinger-bands
- divergence-trading
- fibonacci-trading
- ichimoku-cloud
- macd-trading
- market-regime-detection
- mean-reversion
- moving-average-crossover
- multi-timeframe-analysis
- rsi-divergence
- stochastic-trading
- supply-demand-zones
- volume-profile-trading
- vwap-trading

## Routing gợi ý nhanh
- Không rõ thị trường đang trending/ranging/volatile → `market-regime-detection`
- Cần kiểm tra nhiều khung thời gian → `multi-timeframe-analysis`
- Tránh rủi ro trước khi vào lệnh → `position-sizing`, `stop-loss-strategies`, `risk-reward-ratio`
- Crypto on-chain / funding / rotation → `on-chain-analysis`, `funding-rate-trading`, `altcoin-rotation`
- ICT / smart money / structure → `market-structure-shift`, `order-blocks`, `fair-value-gaps`, `liquidity-zones`
- Pattern trading → chọn skill pattern tương ứng
- Intraday / news / opening volatility → `news-trading`, `gap-trading`, `breakout-trading`, `scalping-strategy`

## Notes
Repo này có 57 skill file SKILL.md. Skill index này là entry point để chọn đúng skill con khi làm việc về trading.
