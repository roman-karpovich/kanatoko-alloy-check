# Alloy real-oracle expiry replay

Two offline tests run current Alloy NAV WASM against captured mainnet
Reflector and RedStone contracts. They confirm expected behavior; no pricing
bug or production failure was found.

Requires Rust 1.95.0 via rustup. Run:

```sh
./check.sh
# After Cargo dependencies have been fetched:
./check.sh --offline
```

Expected: two passing tests. The first build may download Cargo dependencies;
test execution uses the committed captures and has no RPC fallback.

## Scenarios

1. **Existing mainnet-test configuration:** Reflector XLM/USDC is primary
   with a 900-second age limit; RedStone XLM/USD is secondary with a 26-hour
   limit. The USDC/USD conversion uses the existing USD-quoted RedStone
   route, also with a 26-hour limit. The primary is selected one second
   before and exactly at its expiry; one second later NAV selects RedStone.
   RedStone is accepted exactly at its own limit, then `NoFreshPrice` follows.
2. **Isolated conversion expiry:** only the local XLM Reflector route's age
   allowance changes to 27 hours, so its raw asset feed stays eligible while
   the captured RedStone USDC/USD price expires after 26 hours. At conversion
   equality the primary still works; one second later NAV selects XLM/USD
   RedStone. When that secondary expires, NAV returns `NoFreshPrice`.

Oracle code, stored feed prices and feed timestamps are not replaced or
modified. The tests assert that selected feed observations are unchanged
after advancing time. The second case is an explicit local route-setting
variation, not a claim about a deployed fund's configuration. Its capture
contains an XLM/USD update later than the USDC/USD update, leaving a real
fallback window; another capture may not have that ordering.

## Provenance

- Alloy source: `0xDanomite/alloy-stellar` at
  `4ed8639e8e624d2b9674ac7edb665bbd5dcfa5da`, unchanged.
- NAV built with Rust 1.95.0 and Stellar CLI 27.0.0:
  `wasm/alloy_nav.wasm`, SHA-256
  `5a627cb151cd8620da00885d48702ae06520d720631b1400e7d11487515518b0`.
- The same oracle IDs and feed types as upstream's
  `contracts/nav/src/test_mainnet.rs` are used. This complements that test;
  it does not claim that Alloy lacks fork coverage.
- `primary-expiry.json`: Protocol 29 mainnet ledger 65510790,
  timestamp 1794839992; 12 present and 2 confirmed-absent entries.
- `conversion-expiry.json`: Protocol 29 mainnet ledger 65510792,
  timestamp 1794840002; 12 present and 2 confirmed-absent entries.
- Both captures record zero final replay RPC reads and contain coherent
  ledger provenance and content digests.
- Kanatoko is vendored at exact source revision
  `46dc28854e707e8b6e423e04a3aeb347c8f6a3c6` (local-address fix, PR #1).
  Its version label 29.1.1 does not make it identical to the published crate.

The local NAV admin uses mocked authorization. These tests do not sign or
submit transactions, change deployed NAV routes, trade assets, or establish
fee/transaction/Core parity. No real oracle updates are made.
Only ledger time is advanced; ledger sequence, rent and archival are not
modeled. This is frozen-feed expiry coverage, not a forecast of live updates.

## Build and capture

The pinned Alloy contract source is under `upstream/alloy`. To rebuild with
Stellar CLI 27.0.0, enter that directory and run:

```sh
RUSTUP_TOOLCHAIN=1.95.0 RUSTC_WRAPPER= stellar contract build \
  --locked --package alloy-nav --out-dir /path/to/this/repro/wasm
```

The initial probe capture is not part of this package. An explicit
read-only refresh is available through `KANATOKO_CAPTURE_RPC` when calling
`cargo test -- --test-threads=1` directly; `check.sh` clears that variable.
Refresh can change feed ordering and scenario preconditions. A protocol
upgrade needs a matching Host, not an implicit fallback.
