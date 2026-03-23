# Trader Project

## Overview
Crypto trading project using Claude Code with the CCXT MCP server connected to BingX exchange for perpetual futures trading.

## MCP Setup
- **MCP Server**: `@lazydino/ccxt-mcp` (CCXT-based, supports 100+ exchanges)
- **Exchange**: BingX
- **Account name**: `bingx_futures`
- **Trading type**: Perpetual futures (`swap`)
- **Config file**: `ccxt-accounts.json` (contains API keys — do not commit to git)

## Key Files
- `mcp.json` — MCP server configuration for Claude Code
- `ccxt-accounts.json` — BingX API credentials (sensitive)

## Trading Notes
- Always confirm market type is **futures/swap** before placing trades
- API key has **trade + read permissions**; withdraw is disabled
- Use CCXT symbol format for perpetual futures:
  `BTC/USDT:USDT`, `ETH/USDT:USDT`, etc.
