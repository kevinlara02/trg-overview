# TRG Overview

Private multi-restaurant dashboard powered by Clover POS API.

## Restaurants (7)
- Toast Brea · Toast Downey · Little Toast
- Story Whittier · Story Anaheim
- Benny and Mary's · The Benediction

## Weekly Performance (Mar 20-25, 2026)
| Metric | Value |
|--------|-------|
| Net Sales | $402,728.34 |
| Transactions | 4,601 |
| Avg Ticket | $87.51 |
| Tips | $50,062.07 |
| Top Restaurant | Toast Downey ($109,609) |

## Features
- Live Clover API income sync per restaurant
- Food vs Bar revenue breakdown by category
- Hourly sales heatmap across all locations
- Payment method analysis (Credit/Debit/Cash)
- Expense tracking (US Foods, Webstaurant Store, Amazon)
- Cross-restaurant weekly comparison

## Tech Stack
- Vanilla JavaScript · Chart.js 4.4
- Clover REST API v3
- localStorage persistence

## Files
- `index.html` — Full dashboard application
- `data.json` — Saved restaurant & income data
