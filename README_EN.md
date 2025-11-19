[Back to root README](./README.md)

## MNad.Fun Smart Contract Suite

MNad.Fun is a smart contract system for creating, trading, and managing **bonding curve–based tokens** on the **Monad** blockchain.  
It enables:

- **Creators** to launch new tokens backed by bonding curves, with configurable parameters and fee mechanics.
- **Traders** to buy and sell those tokens through a single, on-chain entrypoint with slippage protection and gas-efficient approvals.

The protocol combines **bonding curves** with **automated market making** to provide **continuous liquidity** and **transparent price discovery** from the moment a token is created.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Core Contracts](#core-contracts)
- [Key Components](#key-components)
- [Core Flows](#core-flows)
- [Public Functions](#public-functions)
- [Events](#events)
- [Usage Notes](#usage-notes)
- [Testing](#testing)
- [Development & Security](#development--security)

---

## Architecture Overview

At a high level, MNad.Fun consists of:

- A **router-style contract** (`GNad`) that orchestrates all user-facing operations.
- A **factory** (`BondingCurveFactory`) that deploys and configures individual bonding curves.
- **Per-token bonding curve contracts** (`BondingCurve`) which manage reserves, pricing, and life‑cycle (including locking and DEX listing).
- A **wrapped native token** (`WMon`) providing an ERC20-compatible interface for the Monad native asset.
- A **fee vault** (`FeeVault`) securing accumulated protocol fees behind multisig controls.
- A **single-mint ERC20 token implementation** (`Token`) for newly created assets.

This separation keeps the protocol:

- **Modular** – contracts have clear, isolated responsibilities.
- **Upgradable by composition** – new curves or routing logic can be introduced without breaking existing deployments.
- **Auditable** – each contract has a well-defined surface area.

---

## Core Contracts

### `GNad.sol`

- Central router that coordinates all protocol operations.
- Handles:
  - Token creation via bonding curves.
  - Buy/sell flows against bonding curves.
  - Interactions with `WMon` and `FeeVault`.
- Implements:
  - Slippage protection via min/max output parameters.
  - Deadline checks to prevent stale transactions.
  - EIP‑2612 `permit` support for gasless approvals.

### `BondingCurve.sol`

- Implements bonding curve mechanics using a **constant product formula**.
- Responsible for:
  - Maintaining **virtual** and **real** reserves.
  - Computing spot prices and trade outcomes.
  - Handling buy and sell operations (including lock behavior).
  - Transitioning to a **locked** state once configured thresholds are reached.
  - Supporting **DEX listing** when lock conditions are satisfied.

### `BondingCurveFactory.sol`

- Deploys new instances of `BondingCurve`.
- Maintains a **registry** of all deployed curves.
- Enforces **standardized parameters** (fees, virtual reserves, targets).
- Acts as a central configuration point for:
  - Fee schedule.
  - Virtual reserves and targets.
  - Allowed listing settings.

### `WMon.sol`

- Wrapped Monad (WMon) ERC20 implementation.
- Provides:
  - Deposit / withdraw interface for the native Monad asset.
  - ERC20-compatible balance and transfer semantics.
  - EIP‑2612 `permit` for gasless approvals.

### Supporting Contracts

#### `FeeVault.sol`

- Collects protocol fees from bonding curve operations.
- Secured by **multisig withdrawal**:
  - Multiple signatures required to release funds.
  - Clear separation between trading logic and treasury management.

#### `Token.sol`

- Standard ERC20 implementation tailored for MNad.Fun–created tokens.
- Features:
  - **Single-mint restriction** – token supply can be minted only once.
  - EIP‑2612 `ERC20Permit` support.
  - Optional **burn** functionality for token holders.

### Libraries

- `lib/BCLib.sol`
  - Bonding curve math utilities.
  - Amount-in / amount-out calculations.
  - Fee computation helpers.

- `lib/Transfer.sol`
  - Safe native token transfer helpers.
  - Defensive handling of low-level transfer failures.

### Interfaces & Errors

- Interfaces (`interfaces/*.sol`):
  - Define surface areas for all major contracts.
  - Encourage type-safe integrations and easier tooling.

- Errors (`errors/*.sol`):
  - Centralized custom error definitions.
  - Clear, developer-friendly revert messages.

---

## Key Components

| Component            | Role                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------- |
| Creator              | Launches new tokens and associated bonding curves                                           |
| Trader               | Buys and sells tokens via `GNad`                                                            |
| `GNad`               | Router contract handling creation and trading flows                                         |
| `WMon`               | Wrapped Monad token used as the base asset                                                  |
| `BondingCurveFactory`| Deploys and registers new bonding curve contracts                                           |
| `BondingCurve`       | Manages reserves and bonding curve pricing                                                  |
| `Token`              | ERC20 implementation for MNad.Fun–created tokens                                           |
| DEX                  | External Uniswap V2–compatible AMM used after listing                                       |
| `FeeVault`           | Multisig-controlled vault that holds protocol fees                                          |

---

## Core Flows

### 1. Token Creation

1. A creator calls `createBc` on `GNad`.
2. `BondingCurveFactory` deploys:
   - A new `BondingCurve` instance.
   - A corresponding `Token` instance.
3. Optional: an initial buy can be executed as part of creation to seed liquidity.
4. The function returns:
   - Bonding curve address.
   - Token address.
   - Initial reserve data.

### 2. Trading on the Bonding Curve

- **Buy path**:
  - Trader provides WMon and calls `buy`, `protectBuy`, or `exactOutBuy`.
  - Bonding curve updates reserves and mints/transfers `Token` to the trader.

- **Sell path**:
  - Trader approves or permits `Token` and calls a sell function.
  - Bonding curve updates reserves and transfers WMon back to the trader.

### 3. Locking & DEX Listing

1. Once the configured **lock target** is reached, the bonding curve enters a **locked** state.
2. Trading on the curve stops.
3. The owner or designated role calls `listing`:
   - `Token` and WMon are moved into a Uniswap V2–compatible pair.
   - Liquidity is provided.
   - LP tokens can be burned per configuration to enforce permanent liquidity.

---

## Public Functions

### Creation

- `createBc`
  - Deploys a new token and bonding curve pair.
  - Optionally executes an initial buy.

### Buy Functions

| Function       | Description                                                      |
| -------------- | ---------------------------------------------------------------- |
| `buy`          | Market buy at the current bonding curve price                    |
| `protectBuy`   | Buy with slippage protection (`amountOutMin`)                    |
| `exactOutBuy`  | Purchase an exact amount of tokens from a bonding curve          |

### Sell Functions

| Function             | Description                                                                 |
| -------------------- | --------------------------------------------------------------------------- |
| `sell`               | Market sell at the current bonding curve price                             |
| `sellPermit`         | Same as `sell`, but uses EIP‑2612 `permit` for gasless approval            |
| `protectSell`        | Sell with slippage protection                                              |
| `protectSellPermit`  | Protected sell combined with gasless approval                              |
| `exactOutSell`       | Sell tokens to receive an exact amount of WMon/native asset                |
| `exactOutSellPermit` | `exactOutSell` with EIP‑2612 permit                                        |

### View / Utility

- `getBcData` – Returns bonding curve parameters and state (address, virtual reserves, k).
- `getAmountOut` – Simulates the output amount for a given input.
- `getAmountIn` – Simulates the required input for a desired output.
- `getFeeVault` – Returns the address of the protocol fee vault.

---

## Events

### `GNad` Events

```solidity
event GNadCreate();
event GNadBuy();
event GNadSell();
```

### `BondingCurve` Events

```solidity
event Buy(
    address indexed sender,
    address indexed token,
    uint256 amountIn,
    uint256 amountOut
);

event Sell(
    address indexed sender,
    address indexed token,
    uint256 amountIn,
    uint256 amountOut
);

event Lock(address indexed token);
event Sync(
    address indexed token,
    uint256 reserveWNative,
    uint256 reserveToken,
    uint256 virtualWNative,
    uint256 virtualToken
);

event Listing(
    address indexed curve,
    address indexed token,
    address indexed pair,
    uint256 listingWNativeAmount,
    uint256 listingTokenAmount,
    uint256 burnLiquidity
);
```

### `BondingCurveFactory` Events

```solidity
event Create(
    address indexed creator,
    address indexed bc,
    address indexed token,
    string tokenURI,
    string name,
    string symbol,
    uint256 virtualNative,
    uint256 virtualToken
);
```

---

## Usage Notes

- ⏰ **Deadline parameter**
  - All trading functions accept a deadline to ensure transaction freshness.
  - Transactions with expired deadlines will revert.

- 🔐 **Token approvals & permits**
  - For sell functions, you must either:
    - Call `approve()` on the token contract, or  
    - Use a permit-enabled function (e.g. `sellPermit`).

- 💱 **WMon as base asset**
  - All trades are executed against **WMon**, the wrapped Monad token.

- 🛡️ **Slippage protection**
  - Use `protectBuy` and `protectSell` for price-sensitive flows.

- 🔒 **Locked tokens**
  - Once the lock target is reached, further trades on the bonding curve are disabled.

- 📊 **Virtual vs real reserves**
  - Virtual reserves influence pricing but do not correspond to on-chain balances.
  - Real reserves reflect actual token and WMon balances held by the curve contract.

---

## Testing

The repository includes comprehensive tests using **Foundry**.

```bash
# Run all tests
forge test

# Run with verbose output (shows console.log)
forge test -vv

# Run a specific test file
forge test --match-path test/WMon.t.sol

# Run with gas reporting
forge test --gas-report
```

See `test/README.md` for detailed testing documentation.

---

## Development & Security

### Tech Stack

- **Solidity**: ^0.8.13  
- **Foundry**: Development and testing framework  
- **OpenZeppelin**: ERC20 and ERC20Permit base implementations  
- **Uniswap V2**: Reference DEX for listing and liquidity provisioning  

### Project Structure

```text
src/
├── GNad.sol                 # Main router contract
├── BondingCurve.sol         # Bonding curve implementation
├── BondingCurveFactory.sol  # Factory for creating curves
├── WMon.sol                 # Wrapped Monad token
├── Token.sol                # ERC20 token implementation
├── FeeVault.sol             # Multisig fee vault
├── lib/                     # Utility libraries
│   ├── BCLib.sol            # Bonding curve calculations
│   └── Transfer.sol         # Safe transfer utilities
├── interfaces/              # Contract interfaces
└── errors/                  # Error definitions

test/
└── *.t.sol                  # Test files
```

### Security Considerations

- **Multisig-controlled fees** – protocol revenue is separated from user funds and guarded by multisignature requirements.
- **Slippage checks & deadlines** – protect users against MEV, front‑running, and stale orders.
- **Access control** – administrative actions are protected by explicit modifiers and role checks.
- **Defensive math** – Solidity 0.8+ built‑in overflow checks combined with targeted, well‑tested math libraries.

For support, integration questions, or security disclosures, please refer to the [Support Guide](./SUPPORT_EN.md).

[English](./README.md)

# MNad.Fun Smart Contract

## Table of Contents

- [System Overview](#system-overview)
- [Contract Architecture](#contract-architecture)
- [Key Components](#key-components)
- [Main Functions](#main-functions)
- [Events](#events)
- [Usage Notes](#usage-notes)
- [Testing](#testing)
- [Development Information](#development-information)

## System Overview

MNad.Fun is a smart contract system for creating and managing bonding curve-based tokens on the Monad blockchain. It enables creators to mint new tokens with associated bonding curves and allows traders to buy and sell these tokens through a centralized endpoint. The system uses a combination of bonding curves and automated market makers to provide liquidity and price discovery for newly created tokens.

## Contract Architecture

### Core Contracts

1. **GNad.sol**

   - Central contract that coordinates all system operations
   - Handles token creation, buy, and sell operations
   - Manages interactions with WMon (Wrapped Monad) and fee collection
   - Implements various safety checks and slippage protection
   - Supports EIP-2612 permit functionality for gasless approvals

2. **BondingCurve.sol**

   - Implements bonding curve logic using a constant product formula
   - Calculates token prices based on virtual and real reserves
   - Manages token reserves and liquidity
   - Handles buy/sell operations with a locked token mechanism
   - Supports DEX listing when the target is reached

3. **BondingCurveFactory.sol**

   - Deploys new bonding curve contracts
   - Maintains a registry of created curves
   - Ensures standardization of curve parameters
   - Manages configuration (fees, virtual reserves, target tokens)

4. **WMon.sol**

   - Wrapped Monad token implementation
   - Provides an ERC20 interface for the native Monad token
   - Enables deposit/withdraw functionality
   - Supports EIP-2612 permit

### Supporting Contracts

5. **FeeVault.sol**

   - Collects and manages trading fees
   - Implements a multisig withdrawal mechanism
   - Requires multiple signatures for withdrawals
   - Provides secure fee management

6. **Token.sol**

   - Standard ERC20 implementation for created tokens
   - Includes ERC20Permit for gasless approvals
   - Single-mint restriction (can only be minted once)
   - Burn functionality for token holders

### Libraries

- **lib/BCLib.sol**
  - Bonding curve calculation functions
  - Amount in/out calculations
  - Fee calculation utilities

- **lib/Transfer.sol**
  - Safe native token transfer utilities
  - Gracefully handles transfer failures

### Interfaces

- Defines interfaces for all major contracts
- Ensures correct contract interaction
- Facilitates type safety and integration

### Errors

- Centralizes error definitions as string constants
- Provides clear error messages
- Improves the debugging experience

## Key Components

| Component           | Description                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------ |
| Creator             | Initiates the creation of new tokens and bonding curves                                   |
| Trader              | Interacts with the system to buy and sell tokens                                          |
| GNad                | Main contract handling bonding curve creation, buying, and selling                        |
| WMon                | Wrapped Monad token used for transactions                                                 |
| BondingCurveFactory | Deploys new bonding curve contracts                                                       |
| BondingCurve        | Manages token supply and price calculations using a constant product formula              |
| Token               | Standard ERC20 token contract deployed for each new token                                 |
| DEX                 | External decentralized exchange (Uniswap V2 compatible) used for trading after listing    |
| FeeVault            | Repository for accumulated trading fees; withdrawals controlled via multisig              |

## Main Functions

### Create Functions

- `createBc`: Creates a new token and its associated bonding curve
  - Can optionally perform an initial buy during creation
  - Returns the bonding curve address, token address, and initial reserves

### Buy Functions

| Function      | Description                                           |
| ------------- | ----------------------------------------------------- |
| `buy`         | Market buys tokens at the current bonding curve price |
| `protectBuy`  | Buys tokens with slippage protection                  |
| `exactOutBuy` | Buys an exact amount of tokens from a bonding curve   |

### Sell Functions

| Function             | Description                                                                     |
| -------------------- | ------------------------------------------------------------------------------- |
| `sell`               | Market sells tokens at the current bonding curve price                          |
| `sellPermit`         | Market sells tokens at the current bonding curve price using permit             |
| `protectSell`        | Sells tokens with slippage protection                                           |
| `protectSellPermit`  | Sells tokens with slippage protection using permit                              |
| `exactOutSell`       | Sells tokens on the bonding curve for an exact amount of native tokens         |
| `exactOutSellPermit` | Sells tokens on the bonding curve for an exact amount of native tokens with permit |

### Utility Functions

- `getBcData`: Gets data for a specific bonding curve (address, virtual reserves, k)
- `getAmountOut`: Calculates the output amount for a given input
- `getAmountIn`: Calculates the input amount required for a desired output
- `getFeeVault`: Returns the address of the fee vault

## Events

### GNad Events

```solidity
event GNadCreate();
event GNadBuy();
event GNadSell();
```

### BondingCurve Events

```solidity
event Buy(
    address indexed sender,
    address indexed token,
    uint256 amountIn,
    uint256 amountOut
);

event Sell(
    address indexed sender,
    address indexed token,
    uint256 amountIn,
    uint256 amountOut
);

event Lock(address indexed token);
event Sync(
    address indexed token,
    uint256 reserveWNative,
    uint256 reserveToken,
    uint256 virtualWNative,
    uint256 virtualToken
);

event Listing(
    address indexed curve,
    address indexed token,
    address indexed pair,
    uint256 listingWNativeAmount,
    uint256 listingTokenAmount,
    uint256 burnLiquidity
);
```

### Factory Events

```solidity
event Create(
    address indexed creator,
    address indexed bc,
    address indexed token,
    string tokenURI,
    string name,
    string symbol,
    uint256 virtualNative,
    uint256 virtualToken
);
```

## Usage Notes

- ⏰ **Deadline parameter**: Ensures transaction freshness in all trading functions
- 🔐 **Token approvals**: Some functions require pre-approval of token spending
- 💱 **WMon**: All transactions use WMon (Wrapped Monad) tokens
- 📝 **EIP-2612 permit**: Gasless approvals are available for buy/sell operations
- 🛡️ **Slippage protection**: Implemented in `protectBuy` and `protectSell` functions
- 🔒 **Locked tokens**: Bonding curves lock when the target token amount is reached
- 📊 **Virtual reserves**: Used for price calculation, separate from real reserves
- 🏭 **DEX listing**: Automatic listing on a DEX when the locked token target is reached

## Testing

This project includes comprehensive test coverage using Foundry. To run tests:

```bash
# Run all tests
forge test

# Run with verbose output (shows console.log)
forge test -vv

# Run specific test file
forge test --match-path test/WMon.t.sol

# Run with gas reporting
forge test --gas-report
```

See [test/README.md](test/README.md) for more testing information.

## Development Information

This smart contract system is designed to create and manage bonding curve-based tokens on the Monad blockchain. The system uses:

- **Solidity**: ^0.8.13
- **Foundry**: For development and testing
- **OpenZeppelin**: For ERC20 and ERC20Permit implementations
- **Uniswap V2**: For DEX integration after listing

### Key Features

- Constant product bonding curve formula
- Virtual and real reserve management
- Multisig fee vault
- Gasless approvals via EIP-2612
- Automatic DEX listing mechanism
- Comprehensive test coverage

### Project Structure

```
src/
├── GNad.sol                 # Main contract
├── BondingCurve.sol         # Bonding curve implementation
├── BondingCurveFactory.sol  # Factory for creating curves
├── WMon.sol                 # Wrapped Monad token
├── Token.sol                # ERC20 token implementation
├── FeeVault.sol             # Multisig fee vault
├── lib/                     # Utility libraries
│   ├── BCLib.sol            # Bonding curve calculations
│   └── Transfer.sol         # Safe transfer utilities
├── interfaces/              # Contract interfaces
└── errors/                  # Error definitions

test/
└── *.t.sol                  # Test files
```

### Workflow

1. **Token creation**: Users create a new token and bonding curve via `createBc`
2. **Trading**: Traders interact with the bonding curve via `buy`/`sell` functions
3. **Target reached**: When the token reserve reaches the lock target, the bonding curve automatically locks
4. **DEX listing**: After locking, the curve can be listed on a DEX via the `listing` function

### Security Features

- Multisig fee management
- Slippage protection mechanisms
- Deadline validation
- Access control modifiers
- Safe math operations

### Deployment Steps

1. Deploy the WMon contract
2. Deploy the FeeVault contract (configure multisig owners)
3. Deploy the BondingCurveFactory contract
4. Deploy the GNad contract and initialize it
5. Configure factory parameters (fees, virtual reserves, etc.)

📌 For questions or support, please open an issue in the GitHub repository.

📖 Need help? Check out my [Support Guide](./SUPPORT_EN.md)