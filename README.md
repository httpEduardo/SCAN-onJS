# UniswapV2 Skim Opportunity Scanner

[![Node.js Version](https://img.shields.io/badge/node-%3E%3D12.0.0-brightgreen.svg)](https://nodejs.org/)
[![License: Unlicense](https://img.shields.io/badge/license-Unlicense-blue.svg)](http://unlicense.org/)
[![Web3.js](https://img.shields.io/badge/web3.js-1.3.0-orange.svg)](https://web3js.readthedocs.io/)

A tool to scan all UniswapV2 contracts on the Ethereum network and identify balance/reserve imbalances that can be claimed through the `skim()` function.

## Table of Contents

- [What is Skim?](#what-is-skim)
- [How It Works](#how-it-works)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
  - [Update Market Data](#update-market-data)
  - [Find Skim Opportunities](#find-skim-opportunities)
- [Scripts Overview](#scripts-overview)
- [Example Output](#example-output)
- [Configuration Options](#configuration-options)
- [Troubleshooting](#troubleshooting)
- [Important Notes](#important-notes)
- [Disclaimer](#disclaimer)
- [Contributing](#contributing)
- [License](#license)

## What is Skim?

UniswapV2 implements an interesting function called `skim(address)` that allows anyone to claim a positive discrepancy between the actual token balance in a contract and the reserve number stored in the Pair contract. This occurs when:

- Tokens with variable supply (like aTokens) generate yield
- Rebase tokens (like AMPL) adjust their supply
- Direct token transfers are made to the pair contract
- Other edge cases create balance/reserve mismatches

These scripts scan all UniswapV2 contracts looking for these opportunities.

## How It Works

The scanner works in two phases:

1. **Market Discovery** (`uni-markets.js`): Scans the blockchain for newly deployed UniswapV2 pairs and maintains a registry
2. **Opportunity Detection** (`skim.js`): Analyzes each pair for balance/reserve imbalances and calculates their USD value

## Features

- ✅ Scans all UniswapV2 pairs on Ethereum mainnet
- ✅ Automatically discovers new pairs as they're deployed
- ✅ Calculates USD value of skim opportunities using CoinGecko API
- ✅ Filters opportunities by minimum dollar value
- ✅ Handles non-standard ERC20 tokens (whitelist/blacklist)
- ✅ Real-time WebSocket connection for fast scanning
- ✅ Example Forwarder contract to avoid front-running

## Prerequisites

Before you begin, ensure you have:

- **Node.js** v12.0.0 or higher
- **npm** package manager
- **Ethereum node** with WebSocket support on port 8546 (or configure to use Infura/Alchemy)
- **Git** for cloning the repository

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/httpEduardo/SCAN-onJS.git
   ```

2. Navigate to the repository directory:
   ```bash
   cd SCAN-onJS
   ```

3. Install dependencies:
   ```bash
   npm install
   ```

The installation will add the following dependencies:
- `web3` - Ethereum JavaScript API
- `axios` - HTTP client for API requests

## Configuration

### Node Connection

By default, the scripts connect to a local Ethereum node via WebSocket:
```
ws://localhost:8546
```

To use a different provider (Infura, Alchemy, etc.), modify the connection string in both scripts:

**`scripts/uni-markets.js` (line 5):**
```javascript
const web3 = new Web3('ws://localhost:8546');
```

**`scripts/skim.js` (line 11):**
```javascript
const web3 = new Web3('ws://localhost:8546');
```

### Token Filters

- **Blacklist** (`scripts/blacklist.js`): Contains tokens that are self-destructed, not actual contracts, or completely non-standard ERC20 implementations
- **Whitelist** (`scripts/whitelist.js`): Correctly names tokens that don't follow the ERC-20 standard and return bytes32 for their name (e.g., MKR)

### Minimum Value Threshold

Adjust the minimum dollar value for reported opportunities in `scripts/skim.js`:

```javascript
const minDollarValue = 0.01; // Default: $0.01
```

## Usage

### Update Market Data

To scan for newly deployed UniswapV2 pairs and update the market registry:

```bash
npm run update
```

This command:
- Runs `uni-markets.js`
- Scans from the last known block to the latest block
- Updates `logs/events.js` with new pair data

### Find Skim Opportunities

To scan all known pairs for skim opportunities:

```bash
npm run skim
```

This command:
- Runs `skim.js`
- Checks each pair in `logs/events.js`
- Reports imbalances with USD values above the threshold
- Displays opportunities without CoinGecko pricing data

## Scripts Overview

| Script | Purpose | Output |
|--------|---------|--------|
| **uni-markets.js** | Discovers new UniswapV2 pairs deployed since the last scan | Updates `logs/events.js` with new pair addresses and deployment blocks |
| **skim.js** | Analyzes all pairs for balance/reserve imbalances | Prints JSON objects of profitable skim opportunities with USD values |

### uni-markets.js

- Queries the UniswapV2 Factory contract for `PairCreated` events
- Processes events from the last known block to the latest
- Appends new pairs to `logs/events.js`
- Console output: `🦄 pair #[count] deployed in block #[blockNumber]`

### skim.js

- Iterates through all pairs in `logs/events.js`
- For each pair:
  - Retrieves token balances and reserves
  - Calculates the difference
  - Fetches USD prices from CoinGecko
  - Reports opportunities above the minimum threshold
- Console output: JSON formatted pair data with imbalance details

## Example Output

When running `npm run skim`, you might see:

```json
{
  "pairAddress": "0x1234...5678",
  "pairIndex": 1234,
  "token0": {
    "address": "0xabcd...ef01",
    "name": "Wrapped Ether",
    "decimals": 18,
    "balance": "100.123456789",
    "reserve": "100.000000000",
    "imbalance": {
      "diff": "0.123456789",
      "usdPrice": 2000,
      "value": "$246.91 🦄"
    }
  },
  "token1": {
    "address": "0x9876...5432",
    "name": "USD Coin",
    "decimals": 6,
    "balance": "200000.000000",
    "reserve": "200000.000000",
    "imbalance": false
  }
}
```

## Configuration Options

### Adjusting Scan Range

If you encounter issues updating `events.js` over long time periods, try:

**Option A**: Hardcode the directory path in `uni-markets.js`:
```javascript
fs.writeFile('/absolute/path/to/logs/events.js', ...)
```

**Option B**: Narrow the block range for `eth.getLogs()`:
```javascript
getPastLogs(factoryAddress, startBlock, endBlock); // Instead of 'latest'
```

### Price Source Behavior

- The scanner uses CoinGecko API to fetch token prices
- Tokens without CoinGecko listings are still reported
- Most major tokens have price data available
- Rate limiting may apply for high-frequency requests

## Troubleshooting

### Connection Issues

**Problem**: Cannot connect to Ethereum node  
**Solution**: Verify your WebSocket connection and ensure your node is running on port 8546

### Events Update Errors

**Problem**: `events.js` fails to update  
**Solution**: 
- Check file write permissions for the `logs/` directory
- Try hardcoding the absolute path in `fs.writeFile()`
- Reduce the block range if scanning a large number of blocks

### ERC20 Token Errors

**Problem**: Errors when reading token data  
**Solution**: Add problematic tokens to `scripts/blacklist.js`

### Non-Standard Token Names

**Problem**: Token name returns unexpected format  
**Solution**: Add the token to `scripts/whitelist.js` with the correct name

## Important Notes

### Current Data Status

The `events.js` file in this repository is updated to:
- **Block**: 11,057,149
- **Total Pairs**: 15,230 UniswapV2 pairs

### Default Filters

- Minimum skim value: **$0.01** (configurable)
- Tokens without CoinGecko prices are still displayed
- Blacklisted tokens are automatically skipped

## Disclaimer

> ⚠️ **IMPORTANT: Front-Running Risk**
>
> If you attempt to call `skim()` from an Externally Owned Account (EOA), expect to be front-run. There is a growing number of bots searching for these opportunities. 
>
> **Recommendations:**
> - Use the included `Forwarder.sol` contract to obscure your transactions
> - Consider more sophisticated MEV protection strategies
> - Use Flashbots or private transaction pools
> - Be aware that most skim opportunities are too small to be profitable after gas costs

> ⚠️ **MEV and Gas Costs**
>
> Most balance imbalances are so small that they're not worth the gas cost to call. Only pursue opportunities with sufficient profit margins after accounting for:
> - Gas costs for the transaction
> - Potential front-running competition
> - Network congestion and gas price spikes

> ⚠️ **Use at Your Own Risk**
>
> This software is provided "as is" without warranty of any kind. The authors are not responsible for any losses incurred from using this tool. DeFi arbitrage is competitive and risky.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request. Areas for improvement:

- More sophisticated token whitelist/blacklist handling
- Additional DEX support (SushiSwap, etc.)
- Improved price oracles beyond CoinGecko
- Gas optimization strategies
- Front-running protection mechanisms

## License

This project is released into the public domain under The Unlicense. See [UNLICENSE](UNLICENSE) file for details.

---

**Author**: [@nicholashc](https://github.com/nicholashc)  
**Repository**: [github.com/nicholashc/uniswap-skim/](https://github.com/nicholashc/uniswap-skim/)
