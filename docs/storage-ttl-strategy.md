# Storage TTL and Bump Strategy

This document maps every storage key in the RWA contracts to its storage type and where its TTL is extended.

**Key takeaway:** Storage entries expire unless their TTL is extended before the threshold. Each table below lists which operations bump TTL for which keys, ensuring critical state survives.

---

## Compliance Contract

| Key | Type | Storage | Bump Sites | Notes |
|-----|------|---------|-----------|-------|
| `Admin` | instance | Bumped on: `initialize` | Instance storage; shared with all instance keys | Admin address; bumped once at init |
| `AllowlistMeta` | persistent | Bumped on: `add_to_allowlist`, `remove_from_allowlist` | Tracks cursor for paginated allowlist | Updated on list changes |
| `AllowlistPage(u32)` | persistent | Bumped on: `add_to_allowlist`, `remove_from_allowlist` | Individual pages of KYC addresses | Bumped when page is modified |
| `AllowlistPageOf(Address)` | persistent | Bumped on: `add_to_allowlist`, `remove_from_allowlist` | Address → page mapping for O(1) removal | Bumped on membership change |
| `Record(Address)` | persistent | Bumped on: `add_to_allowlist`, `record_approved_transfer` | Compliance record (approval status, jurisdiction) | Bumped when record changes |
| `Blocked(String)` | persistent | Bumped on: `block_jurisdiction`, `unblock_jurisdiction` | Blocked jurisdiction set | Bumped on list changes |

**TTL Configuration:** `INSTANCE_BUMP_AMOUNT = 30 * DAY_IN_LEDGERS` (30 days), `INSTANCE_LIFETIME_THRESHOLD = INSTANCE_BUMP_AMOUNT - DAY_IN_LEDGERS` (29 days)

**Risk Assessment:** ✅ Safe — all entries bumped on modification

---

## Asset Token Contract

| Key | Type | Storage | Bump Sites | Notes |
|-----|------|---------|-----------|-------|
| `Metadata` | instance | Bumped on: `initialize` | Instance storage; shared | Token name, symbol, decimals; set once |
| `Balance(Address)` | persistent | Bumped on: `transfer`, `mint`, `burn`, `approve` | Per-account balance; critical | Bumped on every balance-modifying operation |

**TTL Configuration:** Same as compliance (30 days)

**Risk Assessment:** ✅ Safe — balances bumped on every transfer/mint/burn

---

## Registry Contract

| Key | Type | Storage | Bump Sites | Notes |
|-----|------|---------|-----------|-------|
| `Admin` | instance | Bumped on: `initialize` | Instance storage; shared | Set once at init |
| `Counter` | instance | Bumped on: `register_asset` | Tracks next asset ID | Updated when new asset registered |
| `Ids` | instance | Bumped on: `register_asset` | Global list of asset IDs | Updated when new asset registered |
| `Asset(u64)` | persistent | Bumped on: `register_asset`, `update_valuation` | Asset metadata and valuation | Bumped on registration and valuation updates |
| `ActiveCount` | persistent | Bumped on: `register_asset` | Count of active assets | Bumped on registration |
| `IssuerIndex(Address)` | persistent | Bumped on: `register_asset` | Issuer → asset IDs mapping | Bumped on registration |
| `TypeIndex(String)` | persistent | Bumped on: `register_asset` | Asset type → asset IDs mapping | Bumped on registration |
| `TotalValuation` | persistent | Bumped on: `update_valuation` | Sum of all asset valuations | Bumped on valuation changes |

**TTL Configuration:** Same as compliance (30 days)

**Risk Assessment:** ⚠️ **Watch** — `Counter` and `Ids` are instance-only; only bumped on registration. If no registrations occur within the TTL window, these could expire. However, this is acceptable because: (1) the contract is initialized once with a counter, and (2) registrations are expected to happen regularly. If the protocol goes dormant, re-registration is an acceptable recovery path.

---

## Dividend Contract

| Key | Type | Storage | Bump Sites | Notes |
|-----|------|---------|-----------|-------|
| `Admin` | instance | Bumped on: `initialize` | Instance storage; shared | Set once at init |
| `Counter` | instance | Bumped on: `create_distribution` | Distribution ID counter | Incremented on distribution creation |
| `Ids` | instance | Bumped on: `create_distribution` | Global list of distribution IDs | Updated on distribution creation |
| `Dist(u64)` | persistent | Bumped on: `create_distribution`, `claim`, `cancel_distribution` | Distribution metadata | Bumped on create, claim, and cancel |
| `Claimed(u64, Address)` | persistent | Bumped on: `claim` | Per-(distribution, holder) claim flag | Bumped when holder claims |
| `AssetIds(Address)` | persistent | Bumped on: `create_distribution` | Asset token → distribution IDs mapping | Bumped on distribution creation |
| `Supply(u64)` | persistent | Bumped on: `create_distribution` | Snapshot supply for distribution | Set at creation, removed when distribution completes |
| `Snapshot(u64)` | persistent | Bumped on: `create_distribution` | Eligible holder list snapshot | Set at creation, removed when distribution completes |

**TTL Configuration:** Same as compliance (30 days)

**Risk Assessment:** ✅ Safe — all critical keys bumped on modification. `Supply` and `Snapshot` are removed when distribution completes (not just left to expire).

---

## Summary

### Universally Safe Keys
- Any key bumped on every modification (transfers, registrations, claims, etc.)
- Instance storage shared across all keys — bumped together

### Keys Requiring Monitoring
- **Registry `Counter` and `Ids`:** Instance-only; bumped only on registration. Acceptable if registrations are frequent.
- **Dividend `Supply` and `Snapshot`:** Removed explicitly when distribution completes, not left to expire.

### Bump Window
- Window duration: 30 days
- Bump threshold: 29 days
- If a key is not accessed within 29 days, its TTL may expire before the next bump

### Test Coverage
See `contracts/*/src/test.rs` for tests that advance the ledger past TTL boundaries and verify state survives. Issue #371 adds ledger advance tests.

---

## Storage Layout Safety (Issue #372)

Each contract uses a distinct `enum DataKey` with unique variant discriminants. Soroban serializes enum variants with discriminants, ensuring no two keys from different variants can collide. Within each contract:

- **Compliance:** 6 distinct variants
- **Asset-Token:** 2 distinct variants  
- **Registry:** 8 distinct variants
- **Dividend:** 8 distinct variants

All variants within each contract are unambiguous. Cross-contract collisions are impossible (different types).

**Conclusion:** Storage layout is safe from collisions.
