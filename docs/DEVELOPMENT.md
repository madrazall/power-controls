# Development Setup & Workflow

## Prerequisites

- Rust 1.70+ ([Install](https://rustup.rs/))
- Node.js 18+ ([Install](https://nodejs.org/))
- Tauri CLI: `cargo install tauri-cli`

## Project Structure Setup

### Initialize Tauri Project

```bash
# Create Tauri app structure
cargo tauri init

# Select options:
# - Package name: power-controls
# - Window title: Power Controls
# - UI framework: None (we'll use Svelte/Vue)
# - TypeScript: Yes
```

### Directory Layout

```
power-controls/
├── src-tauri/                 # Rust backend
│   ├── src/
│   │   ├── main.rs           # Tauri app entry
│   │   ├── commands/          # Tauri command handlers
│   │   │   ├── mod.rs
│   │   │   ├── inventory.rs   # Inventory CRUD
│   │   │   ├── sku.rs        # SKU generation
│   │   │   ├── vendors.rs    # Vendor management
│   │   │   ├── orders.rs     # Order management
│   │   │   └── bom.rs        # BoM operations
│   │   ├── models/            # Data structures
│   │   │   ├── mod.rs
│   │   │   ├── inventory.rs
│   │   │   ├── vendor.rs
│   │   │   ├── order.rs
│   │   │   └── sku.rs
│   │   ├── db/               # Database layer
│   │   │   ├── mod.rs
│   │   │   ├── migrations.rs  # Schema migrations
│   │   │   └── queries.rs     # SQL queries
│   │   └── utils/             # Utilities
│   │       ├── mod.rs
│   │       ├── sku_generator.rs
│   │       └── validation.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
├── src/                       # Frontend (Svelte/Vue)
│   ├── components/
│   ├── views/
│   ├── stores/
│   ├── App.svelte
│   └── main.js
├── docs/
├── package.json
├── tsconfig.json
├── vite.config.ts
└── README.md
```

## Rust Dependencies (Cargo.toml)

```toml
[package]
name = "power-controls"
version = "0.1.0"
edition = "2021"

[dependencies]
tauri = { version = "1.0", features = ["shell-open"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rusqlite = { version = "0.29", features = ["bundled"] }
uuid = { version = "1.0", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
regex = "1.0"

[dev-dependencies]
# Testing dependencies
```

## Database Setup

### Initialize SQLite Database

Create `src-tauri/src/db/migrations.rs`:

```rust
pub const INIT_SQL: &str = r#"
    -- SKU Counters
    CREATE TABLE IF NOT EXISTS sku_counters (
        prefix TEXT PRIMARY KEY,
        next_sequence INTEGER NOT NULL DEFAULT 1
    );
    
    -- Inventory
    CREATE TABLE IF NOT EXISTS inventory (
        sku TEXT PRIMARY KEY,
        category TEXT NOT NULL,
        name TEXT NOT NULL,
        description TEXT,
        specification TEXT,
        status TEXT NOT NULL DEFAULT 'active',
        quantity_on_hand INTEGER DEFAULT 0,
        reorder_point INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        retired_at TIMESTAMP
    );
    
    CREATE INDEX IF NOT EXISTS idx_inventory_category ON inventory(category);
    CREATE INDEX IF NOT EXISTS idx_inventory_status ON inventory(status);
    
    -- Vendors
    CREATE TABLE IF NOT EXISTS vendors (
        vendor_id TEXT PRIMARY KEY,
        name TEXT NOT NULL UNIQUE,
        url TEXT,
        contact_email TEXT,
        notes TEXT,
        is_active BOOLEAN DEFAULT TRUE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    -- Vendor Part Numbers
    CREATE TABLE IF NOT EXISTS vendor_part_numbers (
        vendor_pn_id TEXT PRIMARY KEY,
        vendor_id TEXT NOT NULL,
        sku TEXT NOT NULL,
        vendor_part_number TEXT NOT NULL,
        vendor_price REAL,
        lead_time_days INTEGER,
        minimum_order_qty INTEGER DEFAULT 1,
        moq_price REAL,
        last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (vendor_id) REFERENCES vendors(vendor_id),
        FOREIGN KEY (sku) REFERENCES inventory(sku),
        UNIQUE(vendor_id, vendor_part_number)
    );
    
    CREATE INDEX IF NOT EXISTS idx_vpn_sku ON vendor_part_numbers(sku);
    CREATE INDEX IF NOT EXISTS idx_vpn_vendor ON vendor_part_numbers(vendor_id);
    
    -- Pre-populate common categories
    INSERT OR IGNORE INTO sku_counters (prefix, next_sequence) VALUES
        ('RES', 1), ('CAP', 1), ('IND', 1), ('DIO', 1), ('TRN', 1),
        ('ICH', 1), ('ICA', 1), ('PWR', 1), ('CON', 1), ('CAB', 1),
        ('MEC', 1), ('ASM', 1), ('PCB', 1), ('CUS', 1);
"#;
```

## Development Commands

```bash
# Install dependencies
npm install
cd src-tauri && cargo build

# Development server (Vite + Tauri)
npm run tauri dev

# Build release
npm run tauri build

# Run tests
cd src-tauri && cargo test
```

## Git Workflow

1. **Branch naming:**
   - Feature: `feat/inventory-crud`
   - Fix: `fix/sku-generation`
   - Docs: `docs/architecture`

2. **Commit messages:**
   ```
   feat: add inventory item creation with auto SKU
   
   - Implements sequential SKU generation
   - Validates category prefix
   - Stores in SQLite with audit trail
   ```

3. **Pull requests:** Link to relevant design docs and test results

## Testing Strategy

### Unit Tests (Rust)
- SKU generation logic
- Input validation
- Database queries

### Integration Tests
- Full inventory workflow
- Vendor data integration
- Order processing

### Manual Testing
- UI workflow validation
- Database consistency checks
- Performance under load

## Performance Benchmarks (Target)

- Add inventory item: < 100ms
- Query 10,000 items: < 500ms
- Generate SKU: < 10ms
- Vendor lookup: < 50ms

## Deployment

### Building for Internal Server

```bash
# Build release executable
npm run tauri build

# Output location
# ./src-tauri/target/release/power-controls (Linux)
# ./src-tauri/target/release/power-controls.app (macOS)
# ./src-tauri/target/release/power-controls.exe (Windows)
```

### LAN Distribution
- Copy executable to internal server
- Distribute via network share or installer
- Database file: `~/.config/power-controls/database.db`

## Troubleshooting

### SQLite Errors
- Check database file permissions
- Verify WAL (write-ahead logging) enabled for concurrent access
- Run `PRAGMA integrity_check` in SQLite CLI

### Tauri Build Issues
- Update Rust: `rustup update`
- Clear build cache: `cargo clean`
- Check platform-specific dependencies

## Next Steps

1. Set up project structure with Tauri CLI
2. Implement database initialization
3. Build SKU generation module
4. Create inventory CRUD endpoints
5. Implement vendor data integration
6. Build UI components
