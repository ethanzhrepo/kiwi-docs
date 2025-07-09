# CLI Usage Guide

## Overview

`kiwi-cli` is the command-line tool for the Kiwi payment system, used for system administration, merchant management, fund collection, and other operations.

## Global Configuration

### Configuration File Locations
- Default config file: `~/.kiwi-cli/config.json`
- EVM chains config: `~/.kiwi-cli/evm_chains.json`
- Admin key storage: `~/.kiwi-cli/` directory

### Global Flags
- `--config`: Specify configuration file path

## Configuration Management (config)

### Basic Usage
```bash
kiwi-cli config set [key] [value]   # Set configuration item
kiwi-cli config get [key]           # Get configuration item
kiwi-cli config list                # List all configuration items
kiwi-cli config delete [key]        # Delete configuration item
```

Configuration items are saved in `~/.kiwi-cli/config.json`.

## Admin Commands (admin)

### Generate RSA Key Pairs

#### Generate Admin RSA Keys
```bash
kiwi-cli admin generate-admin-rsa \
    --save google,dropbox,box,keychain,fs \
    --path [key name] \
    --save-public
```

**Parameters:**
- `--save`: Storage methods (required), supports: google, dropbox, box, keychain, fs
- `--path`: Key save path
- `--save-public`: Whether to save public key to `~/.kiwi-cli/admin_pool_pub.pem` (default: true)

#### Generate CLI Communication Keys
```bash
kiwi-cli admin generate-cli-rsa \
    --force    # Force overwrite existing private key file
```

Generate 4096-bit RSA key pair for CLI communication, private key is AES encrypted and saved in `~/.kiwi-cli/admin_cli_private.pem`.

#### RSA Signing
```bash
kiwi-cli admin rsa-sign \
    --provider fs \
    --encrypted-private-key-file [private key file path] \
    --data [data to sign]
```

Sign data using RSA-PSS with SHA384.

## Merchant Management (merchant)

### Merchant Operations

#### List Merchants
```bash
kiwi-cli merchant list \
    --all          # List all merchants (no pagination)
    --page 1       # Page number (default: 1)
    --size 10      # Page size (default: 10)
```

#### Add Merchant
```bash
kiwi-cli merchant add \
    --name [merchant name] \
    --notify-url [notification URL]
```

**Parameters:**
- `--name`: Merchant name (required)
- `--notify-url`: Notification callback URL (optional)

#### Delete Merchant
```bash
kiwi-cli merchant delete --id [merchant ID]
```

#### Reset Merchant API Key
```bash
kiwi-cli merchant reset-key --id [merchant ID]
```

## Wallet Management (wallet)

### Wallet Address Management

#### Add Wallet Addresses
```bash
kiwi-cli wallet add \
    --count [generation count] \
    --type [chain type]
```

Batch generate receiving wallet addresses, wallet private keys are encrypted using AES-256-GCM.

#### Get Wallet Information
```bash
kiwi-cli wallet get [parameters]
```

#### Enable/Disable Wallets
```bash
kiwi-cli wallet enable [wallet parameters]      # Enable wallet
kiwi-cli wallet disable [wallet parameters]     # Disable wallet
kiwi-cli wallet enable-batch [batch]            # Batch enable
kiwi-cli wallet disable-batch [batch]           # Batch disable
```

#### Wallet Statistics
```bash
kiwi-cli wallet stat
```

Display wallet pool statistics.

## Fund Collection (collect)

### Collection Task Management

#### List Collection Tasks
```bash
kiwi-cli collect collect-task list \
    --status [status]    # Filter by status: pending, collected
```

#### Create Collection Task
```bash
kiwi-cli collect collect-task create \
    --threshold 0      # Collection threshold (amounts below this won't be collected)
```

#### Get Task Details
```bash
kiwi-cli collect collect-task get [taskId]
```

#### Delete Task
```bash
kiwi-cli collect collect-task delete [taskId]
```

### Execute Collection

```bash
kiwi-cli collect run \
    --task-id [task ID] \
    --to [collection target address] \
    --relayer-provider [relayer wallet provider] \
    --relayer-private-key-file [relayer wallet file] \
    --encrypted-rsa-admin-key-file [encrypted RSA private key file]
```

**Required Parameters:**
- `--task-id`: Task ID to execute collection for
- `--to`: Target address for fund collection
- `--relayer-provider`: Relayer wallet provider (google, fs, keychain, dropbox, s3)
- `--relayer-private-key-file`: Relayer wallet file path
- `--encrypted-rsa-admin-key-file`: Encrypted RSA private key file for decrypting wallet keys

**Optional Parameters:**
- `--auto-7702`: Use EIP-7702 protocol (default: true)
- `--batch-size`: Maximum addresses per transaction (default: 200)
- `--report-dir`: Collection report save directory (default: ./reports)
- `--template-address`: EIP-7702 template contract address
- `--token-filter`: Token filter for collection (comma-separated token symbols)
- `--gas-price`: Custom gas price
- `--max-gas-cost`: Maximum gas cost
- `--gas-multiplier`: Gas price multiplier (default: 1.0)
- `--log-level`: Log level (default: info)
- `--log-file`: Log file
- `--log-format`: Log format (default: text)
- `--retry-failed`: CSV file path for retrying failed transactions

## Transaction Management (transaction)

### Query Orders
```bash
kiwi-cli transaction get-order [order_id]
```

Automatically detect order type by order ID:
- P prefix: Payment order
- S prefix: Subscribe order (includes associated bills)
- B prefix: Subscribe bill

### Query Transactions
```bash
kiwi-cli transaction get-tx [tx_hash]
```

Get transaction details by transaction hash.

## System Management (system)

### System Status

#### Check Server Connectivity
```bash
kiwi-cli system ping
```

#### View Server Status
```bash
kiwi-cli system server-status
```

Display server status and metrics including CPU, memory usage, and EVM chain listener status.

### Nonce Management

#### Sync All Nonces
```bash
kiwi-cli system nonce sync
```

#### Get Nonce Status
```bash
kiwi-cli system nonce status <chain_id> <address>
```

#### Sync Single Address Nonce
```bash
kiwi-cli system nonce sync-single <chain_id> <address>
```

#### Validate Nonce Sequences
```bash
kiwi-cli system nonce validate
```

## Tools (tools)

### Generate OTP
```bash
kiwi-cli tools generate-otp
```

Generate OTP key for Google Authenticator, displays on screen after generation with configuration guidance.

### Airdrop Management
```bash
kiwi-cli tools airdrop [parameters]
```

Airdrop fee management for pre-collection fee airdrops.

### System Statistics
```bash
kiwi-cli tools stat
```

Display system statistics including deposits, payments, and merchant data.

## Security Features

1. **Key Management**: Supports multiple storage providers (filesystem, cloud storage, HSM)
2. **Encryption Protection**: RSA encryption, HSM support, and secure key management
3. **Batch Operations**: Many commands support batch processing for efficiency
4. **Multi-chain Support**: Supports multiple blockchain networks
5. **Advanced Features**: EIP-7702 support for efficient token collection

## Batch Airdrop Contract Addresses

Universal for all EVM chains:
- Contract Address: `0xE9511e55d2AaC1F62D7e3110f7800845dB2a31F1`
- TRON Chain: `TNnHipM7aZMYYanXhESgRV9NmjndcgvaXu`

Reference project: [https://github.com/WhiteRiverBay/wrb-airdrop-contract](https://github.com/WhiteRiverBay/wrb-airdrop-contract)

## Batch Query Addresses

EVM multicall contract:
- Contract Address: `0x058C6121efBF3e7C1f856928f7e9ecBC71c5772a`

Reference project: [https://github.com/WhiteRiverBay/evm-multicall](https://github.com/WhiteRiverBay/evm-multicall)

## Usage Examples

### Complete Merchant Setup Process
```bash
# 1. Set API endpoint
kiwi-cli config set api_url https://api.example.com

# 2. Generate admin keys
kiwi-cli admin generate-admin-rsa --save fs --path admin_keys

# 3. Add merchant
kiwi-cli merchant add --name "Example Merchant" --notify-url https://merchant.com/notify

# 4. Generate wallet addresses
kiwi-cli wallet add --count 1000 --type ETH

# 5. Check system status
kiwi-cli system server-status
```

### Fund Collection Process
```bash
# 1. Create collection task
kiwi-cli collect collect-task create --threshold 0.001

# 2. View task list
kiwi-cli collect collect-task list

# 3. Execute collection
kiwi-cli collect run \
    --task-id TASK123 \
    --to 0x1234567890abcdef \
    --relayer-provider fs \
    --relayer-private-key-file ./relayer.key \
    --encrypted-rsa-admin-key-file ./admin.pem
```

## Command Categories

### Administrative Commands
- **config**: Configuration management
- **admin**: Admin key generation and signing
- **system**: System monitoring and maintenance

### Business Operations
- **merchant**: Merchant account management
- **wallet**: Wallet address pool management
- **transaction**: Transaction and order queries

### Operational Tools
- **collect**: Fund collection automation
- **tools**: Utility tools (OTP, statistics, airdrop)

## Advanced Features

### EIP-7702 Support
The CLI supports EIP-7702 protocol for efficient token collection, enabling:
- Batch collection from multiple addresses
- Reduced gas costs
- Improved transaction throughput

### Multi-Provider Support
Storage providers supported:
- **fs**: Local filesystem
- **google**: Google Drive
- **dropbox**: Dropbox
- **box**: Box.com
- **keychain**: macOS/iOS Keychain
- **s3**: Amazon S3

### Security Best Practices
- All private keys are encrypted at rest
- RSA-PSS signing with SHA384
- Support for hardware security modules (HSM)
- Multi-layer encryption for sensitive data

For detailed information about the Admin REST API that this CLI interacts with, please refer to [CLI Administrator Interface Overview (ADMIN_REST_API_en.md)](docs/ADMIN_REST_API_en.md).