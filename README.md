hldata
Rust reader library for Hyperliquid order log binary data files.

These are LZ4-compressed binary files produced by a processing pipeline that converts raw Hyperliquid L1 exchange data (order status updates, L4 orderbook snapshots) into compact binary formats optimized for backtesting and market microstructure analysis. This crate provides read-only access to these files with a single dependency (lz4_flex).

Data Layout
Data is organized by date, with one directory per day:

converted_order_log_lz4/
  20260120/
    orderlog/
      ETH_20260120.bin          # Order log for ETH on 2026-01-20
      BTC_20260120.bin          # Order log for BTC on 2026-01-20
      ...
    snapshots/
      ETH_20260120.snapshots    # Periodic L4 snapshots for ETH
      BTC_20260120.snapshots    # Periodic L4 snapshots for BTC
      ...
Two File Types
.bin (Order Log) — The primary data file. Contains a full day of order activity for one instrument:

Reconstructed midnight snapshot — The file starts with a depth snapshot (bid/ask levels) representing the order book state at 00:00:00 UTC. This is reconstructed by replaying order updates on top of the last available L4 snapshot from the previous day. This gives you a starting state to replay the full day from.
EndOfSnapshot marker — Separates the snapshot from the order stream.
Order updates — Every order status change for the rest of the day (opens, fills, cancels, triggers, etc.) in chronological order. Each update includes the wallet address, order ID, price, size, side, type, time-in-force, and L1 block number.
.snapshots (Multi-Snapshot) — Contains periodic L4 orderbook snapshots taken throughout the day for one instrument:

Midnight snapshot (block_height=0) — The same reconstructed 00:00:00 state as in the .bin file.
Periodic snapshots — Full L4 orderbook state captured at various block heights during the day. Each snapshot contains every open order (price, size, side, user, OID) at that point in time.
The .bin file is what you need for backtesting (replay order-by-order). The .snapshots file is useful for point-in-time orderbook reconstruction or validation.

Installation
[dependencies]
hldata = "0.1"
Or add via CLI:

cargo add hldata
Quick Start
Reading Order Log Files (.bin)
use hldata::{OrderLogReader, fixed_to_decimal};

let mut reader = OrderLogReader::open("converted_order_log_lz4/20260120/orderlog/ETH_20260120.bin")?;
println!("Instrument ID: {}", reader.header().instrument_id);

// Step 1: Read the reconstructed midnight depth snapshot
let snapshot = reader.read_depth_snapshot()?;
println!("Best bid: {}", fixed_to_decimal(snapshot.bids[0].0));
println!("Best ask: {}", fixed_to_decimal(snapshot.asks[0].0));
println!("{} bid levels, {} ask levels", snapshot.bids.len(), snapshot.asks.len());

// Step 2: Iterate over all order updates for the day
for result in reader.order_updates() {
    let order = result?;
    println!("OID {} {} {} @ {} (block {})",
        order.oid,
        order.order_status().to_str(),
        if order.is_buy() { "BUY" } else { "SELL" },
        fixed_to_decimal(order.price),
        order.block_number);
}
# Ok::<(), std::io::Error>(())
Reading Multi-Snapshot Files (.snapshots)
use hldata::{MultiSnapshotReader, fixed_to_decimal};

let mut reader = MultiSnapshotReader::open(
    "converted_order_log_lz4/20260120/snapshots/BTC_20260120.snapshots"
)?;
println!("Snapshots: {}", reader.snapshot_count());
println!("Has midnight: {}", reader.has_midnight());

// List all available snapshots
for entry in reader.list_snapshots() {
    let label = if entry.is_midnight() { "MIDNIGHT" } else { "regular" };
    println!("  {} block={} ts={}", label, entry.block_height, entry.timestamp_ms);
}

// Read the reconstructed midnight snapshot
let midnight_orders = reader.read_midnight_snapshot()?;
println!("Midnight: {} open orders", midnight_orders.len());

// Read a specific snapshot by block height
// let orders = reader.read_snapshot(856260000)?;

// Or read all snapshots
let all = reader.read_all_snapshots()?;
for (entry, orders) in &all {
    println!("Block {}: {} orders", entry.block_height, orders.len());
}
# Ok::<(), std::io::Error>(())
Low-Level Access with BinaryReader
use hldata::{BinaryReader, Message};

let mut reader = BinaryReader::open("converted_order_log_lz4/20260120/orderlog/ETH_20260120.bin")?;
for result in reader.iter() {
    match result? {
        Message::DepthLevel(level) => { /* bid/ask from snapshot */ }
        Message::OrderUpdate(order) => { /* order status change */ }
        Message::EndOfSnapshot(_) => { /* snapshot section complete */ }
    }
}
# Ok::<(), std::io::Error>(())
File Verification & Statistics
use hldata::BinaryReader;

let stats = BinaryReader::verify("converted_order_log_lz4/20260120/orderlog/ETH_20260120.bin")?;
stats.print_summary("ETH");
// Prints: file size, compression ratio, message counts,
// order status distribution, side distribution, unique users, etc.
# Ok::<(), std::io::Error>(())
Running the Examples
# Read a .bin order log file
cargo run --example read_orderlog -- /path/to/ETH_20260120.bin

# Read a .snapshots file
cargo run --example read_snapshots -- /path/to/BTC_20260120.snapshots
Fixed-Point Arithmetic
All prices and sizes are stored as i64 with a scale factor of 10^8 (PRICE_SCALE = 100_000_000).

use hldata::{decimal_to_fixed, fixed_to_decimal, PRICE_SCALE};

// String to fixed-point
let price = decimal_to_fixed("25.381").unwrap();
assert_eq!(price, 2_538_100_000i64);

// Fixed-point to string
assert_eq!(fixed_to_decimal(2_538_100_000), "25.381");

// Manual conversion to f64
let price_f64 = 2_538_100_000i64 as f64 / PRICE_SCALE as f64; // 25.381
API Reference
Readers
Type	File	Description
OrderLogReader	.bin	High-level: reads depth snapshot then order updates
BinaryReader	.bin	Low-level: iterates over all raw messages
SnapshotReader	.snap	Reads single snapshot files
MultiSnapshotReader	.snapshots	Reads consolidated multi-snapshot files
Data Types
Type	Size	Description
FileHeader	16 bytes	.bin file header
SnapHeader	32 bytes	.snap file header
MultiSnapHeader	24 bytes	.snapshots file header
DepthLevel	32 bytes	Single bid/ask price level
OrderUpdate	108 bytes (V3)	Order status change with all fields
EndOfSnapshot	16 bytes	Marks end of depth snapshot section
Enums
OrderStatus — Open, Filled, Canceled, Triggered, ReduceOnlyCanceled, MarginCanceled, BadAloPxRejected, OpenInterestCapCanceled, Rejected
OrderType — Limit, Market, StopMarket, StopLimit, TakeProfitLimit, TakeProfitMarket, StopLossLimit, StopLossMarket
TimeInForce — GTC, IOC, ALO (post-only), FOK
TriggerCondition — None, PriceAbove, PriceBelow
Utility Functions
decimal_to_fixed(s) -> Option<i64> — Parse decimal string to fixed-point
fixed_to_decimal(v) -> String — Format fixed-point as decimal string
parse_wallet_address(s) -> Option<[u8; 20]> — Parse 0x... hex address
format_wallet_address(bytes) -> String — Format bytes as 0x... address
parse_cloid(s) -> [u8; 16] — Parse client order ID hex string
format_cloid(bytes) -> String — Format client order ID
See FORMAT_SPEC.md for the complete binary format specification with byte-level layouts.

License
MIT
