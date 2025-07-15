# op-celestia-indexer

The `op-celestia-indexer` is a service that indexes L2 block locations on both
Celestia DA and Ethereum DA (standard calldata). It tracks where L2 blocks are 
stored by parsing batch transactions and maintaining a mapping between L2 block 
numbers and their corresponding DA locations.

## Overview

The indexer supports both DA types and determines which one to use based on the
frame version byte in batch transactions:

### Celestia DA
When using Celestia as the DA layer, L2 batch data (frames) are posted to 
Celestia instead of being included as calldata in L1 transactions. L1 
transactions contain:
- Frame version byte `0x01` (OP Stack Alt-DA format)
- Commitment type byte
- DA layer byte `0x0c` (Celestia identifier)
- 8 bytes Celestia block height (little-endian)
- 32 bytes commitment hash

### Ethereum DA
When using standard Ethereum DA, L2 batch data is included as calldata in L1
transactions with:
- Frame version byte `0x00` (standard OP Stack format)
- Frame data directly in calldata

The indexer service:
- Monitors L1 batch inbox transactions
- Checks frame version byte to determine DA type
- Fetches frame data from Celestia or parses L1 calldata
- Parses frames to determine which L2 blocks they contain
- Maintains an index mapping L2 block numbers to DA locations
- Provides RPC APIs to query L2 block locations for both DA types

## CLI Flags

### Required Flags
- `--start-l1-block`: Starting L1 block number for indexing
- `--batch-inbox-address`: Address of the batch inbox contract
- `--l1-eth-rpc`: HTTP provider URL for L1 Ethereum
- `--l2-eth-rpc`: HTTP provider URL for L2 Ethereum
- `--op-node-rpc`: HTTP provider URL for op-node (for verification)

### Optional Flags
- `--enable-admin`: Enable admin API (default: false)
- `--poll-interval`: Polling interval for new blocks (default: 12s)
- `--network-timeout`: Timeout for network requests (default: 10s)
- `--verify-parent-check`: Enable parent check verification in span batches (default: true)
- `--db-path`: Path to the SQLite database (default: in memory)

### Celestia DA Flags
- `--da.rpc`: Celestia DA client RPC endpoint
- `--da.auth_token`: Authentication token for Celestia client
- `--da.namespace`: Namespace for Celestia DA operations
- `--da.fallback_mode`: Fallback mode (disabled/blobdata/calldata)
- `--da.gas_price`: Gas price for Celestia operations

### Standard op-service Flags
- RPC server configuration (`--rpc.addr`, `--rpc.port`, `--rpc.enable-admin`)
- Logging configuration (`--log.level`, `--log.format`)
- Metrics configuration (`--metrics.enabled`, `--metrics.addr`, `--metrics.port`)
- Profiling configuration (`--pprof.enabled`, `--pprof.addr`, `--pprof.port`)

## API Usage

### Get DA Location

Query the DA location for a specific L2 block (works with both Celestia and Ethereum DA):

```bash
curl -X POST -H "Content-Type: application/json" -s \
  --data '{"jsonrpc":"2.0","method":"admin_getDALocation","params":[355],"id":1}' \
  http://localhost:57220 | jq .
```

Response for Celestia DA:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "type": "celestia",
    "data": {
      "height": 353,
      "commitment": "YQEAAAAAAADg6goIrTykl5jyHlGz6Bl2tYTDYzffUY39g3inPvMGDQ==",
      "l2_range": {
        "start": 354,
        "end": 359
      },
      "l1_block": 12345
    }
  }
}
```

Response for Ethereum DA:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "type": "ethereum",
    "data": {
      "tx_hash": "0x123...",
      "l2_range": {
        "start": 354,
        "end": 359
      },
      "l1_block": 12345
    }
  }
}
```

### Get Celestia Location (Legacy)

Query the Celestia location for a specific L2 block (backward compatibility):

```bash
curl -X POST -H "Content-Type: application/json" -s \
  --data '{"jsonrpc":"2.0","method":"admin_getCelestiaLocation","params":[355],"id":1}' \
  http://localhost:57220 | jq .
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "height": 353,
    "commitment": "YQEAAAAAAADg6goIrTykl5jyHlGz6Bl2tYTDYzffUY39g3inPvMGDQ==",
    "l2_range": {
      "start": 354,
      "end": 359
    },
    "l1_block": 12345
  }
}
```

**Note:** This endpoint only returns Celestia DA locations. For Ethereum DA blocks, it will return an error. Use `admin_getDALocation` for both DA types.

### Get Indexer Status

Query the current indexer status:

```bash
curl -X POST -H "Content-Type: application/json" -s \
  --data '{"jsonrpc":"2.0","method":"admin_getIndexerStatus","params":[],"id":1}' \
  http://localhost:57220 | jq .
```

## Example Usage

Start the indexer service:

```bash
op-celestia-indexer \
  --start-l1-block 12000 \
  --batch-inbox-address 0x00a4FE4C6AaA0729d7699c387E7f281DD64aFA2a \
  --l1-eth-rpc  http://127.0.0.1:54049 \
  --l2-eth-rpc http://127.0.0.1:54314 \
  --op-node-rpc http://127.0.0.1:54328 \
  --rpc.enable-admin \
  --db-path indexer.db \
  --log.level debug \
  --da.rpc http://127.0.0.1:54300 \
  --da.namespace 00000000000000000000000000000000000000000008e5f679bf7116cb  \
  --da.auth_token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJwdWJsaWMiLCJyZWFkIiwid3JpdGUiLCJhZG1pbiJdfQ.w8Jg1rSf4TqukE4Os35sXQQ1G9hO2BBYM_0lKHqEyo4

## Testing

Run the test suite:
```bash
just test
```

Run with verbose output:
```bash
go test -v ./...
```

## Building

Build the binary:
```bash
just op-celestia-indexer
```
Clean build artifacts:
```bash
just clean
```

## Migration Notes

### Upgrading from Celestia-only version

The indexer now supports both Celestia DA and Ethereum DA. Existing deployments will continue to work without changes. The database schema is updated to support both DA types:

1. New `eth_locations` table is created for Ethereum DA locations
2. `l2_block_mappings` table gets a new `da_type` column (defaults to 'celestia' for existing entries)
3. Existing Celestia-only data remains unchanged and accessible

The `admin_getCelestiaLocation` RPC endpoint continues to work for backward compatibility but only returns Celestia DA locations. Use the new `admin_getDALocation` endpoint to query both DA types.
