# Architecture Documentation

## System Overview

Power Controls is a standalone Tauri + Rust application designed for internal LAN deployment.

```
┌─────────────────────────────────────────┐
│     Tauri Frontend (Web UI)             │
│  (HTML/CSS/JavaScript/TypeScript)       │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│     Rust Backend (Tauri Commands)       │
│  - Inventory Logic                      │
│  - SKU Generation                       │
│  - Vendor Management                    │
│  - Order Processing                     │
│  - BoM Generation                       │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│     SQLite Database                     │
│  - Inventory                            │
│  - Vendors & Part Numbers               │
│  - Orders                               │
│  - BOMs                                 │
│  - SKU Counters (per category)          │
└─────────────────────────────────────────┘
```

## Technology Stack

- **Desktop Framework:** Tauri (Rust backend, Web frontend)
- **Backend Language:** Rust
- **Frontend:** HTML, CSS, JavaScript/TypeScript (Vue.js or Svelte recommended)
- **Database:** SQLite (embedded, no external dependencies)
- **Deployment:** Internal server/LAN distribution

## Key Design Principles

1. **Single Component Per SKU:** No duplicate inventory entries
2. **Sequential SKU Assignment:** Within category prefixes (CAT-0000001, etc.)
3. **Immutable SKU History:** Retired items retain SKUs permanently
4. **Vendor Agnostic:** Support multiple vendors per component
5. **Offline Capable:** Runs on internal LAN without external dependencies

## Module Structure

```
power-controls/
├── src-tauri/
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands/
│   │   │   ├── inventory.rs
│   │   │   ├── sku.rs
│   │   │   ├── vendors.rs
│   │   │   ├── orders.rs
│   │   │   └── bom.rs
│   │   ├── models/
│   │   │   ├── inventory.rs
│   │   │   ├── vendor.rs
│   │   │   ├── order.rs
│   │   │   └── sku.rs
│   │   ├── db/
│   │   │   ├── mod.rs
│   │   │   ├── migrations.rs
│   │   │   └── queries.rs
│   │   └── utils/
│   │       └── sku_generator.rs
│   └── Cargo.toml
├── src/
│   ├── components/
│   ├── views/
│   ├── App.svelte (or Vue equivalent)
│   └── main.js
├── docs/
│   ├── ARCHITECTURE.md (this file)
│   ├── SKU_SPECIFICATION.md
│   ├── DATA_MODEL.md
│   ├── DEVELOPMENT.md
│   └── API_REFERENCE.md
└── package.json
```

## Data Flow

### Adding New Inventory Item
1. User submits component details (name, category, specs)
2. Frontend sends to Tauri command: `add_inventory_item`
3. Rust backend validates category
4. Queries database for highest SKU number in category
5. Generates next sequential SKU
6. Inserts item with auto-generated SKU
7. Returns SKU to frontend for confirmation

### SKU Generation Pipeline
```
Category (e.g., "Resistor") → 3-letter prefix ("RES")
                            ↓
                   Query last sequence number
                            ↓
                    Increment: 0000001 → 0000002
                            ↓
                    Format: RES-0000001
                            ↓
                    Store in inventory + SKU history
```

### Vendor Data Integration
```
Inventory Item (SKU: RES-0000001)
         ↓
  ┌──────┴──────┐
  ↓             ↓
Vendor A    Vendor B
(Part: 1M-5%) (Part: 1MΩ-1/4W)
(Price: $0.05) (Price: $0.08)
```

## Database Dependencies

- SQLite (bundled with Tauri)
- rusqlite (Rust SQLite driver)
- Optional: sqlx for async queries

## Future Scalability

- **Multi-user Support:** SQLite WAL mode for concurrent access
- **Export/Import:** CSV, JSON export capabilities
- **Audit Trail:** Log all inventory changes
- **Advanced Queries:** Full-text search on component names/specs
- **Mobile Companion:** Possible React Native mobile app reading same database
