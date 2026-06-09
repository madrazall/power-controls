# SKU Generation Specification

## Format Definition

```
[3-LETTER-PREFIX]-[7-DIGIT-NUMBER]
└─────┬─────┘  │  └─────┬─────┘
   Category   Hyphen  Sequence
```

### Components

#### 1. Three-Letter Prefix (Category)
- **Purpose:** Identifies component category
- **Format:** Uppercase letters only, no spaces or special characters
- **Examples:**
  - `RES` - Resistors
  - `CAP` - Capacitors
  - `IND` - Inductors
  - `DIO` - Diodes
  - `TRN` - Transistors
  - `ICH` - Integrated Circuits (High-speed)
  - `PWR` - Power components
  - `CON` - Connectors
  - `PCB` - PCB/Board assemblies
  - `CAB` - Cables & Harnesses
  - `MEC` - Mechanical components

#### 2. Hyphen Separator
- **Always present**
- **Single character**
- **No flexibility** - enforcement at database level

#### 3. Seven-Digit Number
- **Format:** Zero-padded decimal
- **Range:** 0000001 through 9999999
- **Assignment:** Sequential within category
- **Reuse:** Never - retired items keep SKUs permanently

## SKU Generation Algorithm

```rust
fn generate_sku(category: &str) -> Result<String, Error> {
    // 1. Validate & normalize category (3 uppercase letters)
    let prefix = validate_prefix(category)?;
    
    // 2. Query database for highest sequence in category
    let last_sequence = db.get_max_sequence(&prefix)?;
    
    // 3. Increment sequence
    let new_sequence = last_sequence + 1;
    
    // 4. Check overflow (7-digit limit)
    if new_sequence > 9_999_999 {
        return Err("Category exhausted - 9,999,999 items max");
    }
    
    // 5. Format: RES-0000001
    let sku = format!("{}-{:07}", prefix, new_sequence);
    
    // 6. Return SKU
    Ok(sku)
}
```

## Category Management

### Pre-defined Categories

Categories are managed in the database with configurable prefixes:

| Category Name | Prefix | Purpose | Max Items |
|---|---|---|---|
| Resistors | RES | Passive resistive elements | 9,999,999 |
| Capacitors | CAP | Energy storage - AC applications | 9,999,999 |
| Inductors | IND | Energy storage - magnetic | 9,999,999 |
| Diodes | DIO | Rectification, switching | 9,999,999 |
| Transistors | TRN | Switching, amplification | 9,999,999 |
| ICs High-Speed | ICH | Digital logic, processors | 9,999,999 |
| ICs Analog | ICA | Amplifiers, sensors | 9,999,999 |
| Power Components | PWR | Power supplies, transformers | 9,999,999 |
| Connectors | CON | Physical interconnects | 9,999,999 |
| Cables/Harnesses | CAB | Wired assemblies | 9,999,999 |
| Mechanical | MEC | Hardware, structural | 9,999,999 |
| Assemblies | ASM | Sub-assemblies, modules | 9,999,999 |
| PCB/Boards | PCB | Circuit boards | 9,999,999 |
| Custom | CUS | Vendor-specific, unique items | 9,999,999 |

### Adding New Categories

New categories are added via admin interface:
1. Provide 3-letter prefix
2. Validate prefix doesn't conflict with existing
3. Initialize sequence counter at 0
4. New items in category start at 0000001

## Uniqueness Constraints

### Database Schema

```sql
CREATE TABLE inventory (
    sku TEXT PRIMARY KEY,              -- RES-0000001 (unique, immutable)
    category TEXT NOT NULL,            -- "Resistor"
    prefix TEXT NOT NULL,              -- "RES"
    sequence INTEGER NOT NULL,         -- 1
    name TEXT NOT NULL,                -- "1MΩ Carbon Film"
    description TEXT,
    spec_value TEXT,                   -- "1M"
    spec_unit TEXT,                    -- "Ω"
    status TEXT DEFAULT 'active',      -- 'active', 'obsolete', 'discontinued'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    retired_at TIMESTAMP,
    UNIQUE(prefix, sequence)           -- Prevent duplicate sequences in category
);

CREATE TABLE sku_counters (
    prefix TEXT PRIMARY KEY,           -- "RES"
    next_sequence INTEGER NOT NULL     -- 2 (next to assign)
);
```

## Validation Rules

✓ **Valid SKUs:**
- `RES-0000001`
- `CAP-0000042`
- `PWR-9999999`

✗ **Invalid SKUs:**
- `RES0000001` (missing hyphen)
- `RES--0000001` (double hyphen)
- `RES-000001` (only 6 digits)
- `RES-00000001` (8 digits)
- `res-0000001` (lowercase prefix)
- `R3S-0000001` (non-letter in prefix)
- `RES_0000001` (wrong separator)
- `RES-0000000` (sequence starts at 1, not 0)

## Migration Path for Existing Inventory

If importing existing inventory:
1. Group by category
2. Sort by current part number
3. Assign sequential SKUs starting from 0000001
4. Create audit log with original part numbers
5. Maintain mapping table: `original_part_number` ↔ `new_sku`

## SKU Retirement Process

When an item is no longer stocked:
1. Mark as `status = 'discontinued'` or `'obsolete'`
2. **DO NOT** reuse the SKU
3. Keep complete history (audit trail)
4. If item is restocked, restore status to `'active'`
5. SKU remains permanent

## Performance Considerations

- **Sequence Counter Index:** `sku_counters.prefix` (PK) - O(1) lookup
- **Category Lookup:** `inventory.prefix` (indexed) - fast filtering
- **Uniqueness:** `(prefix, sequence)` unique constraint - prevents duplicates at insert
- **Concurrent Access:** SQLite WAL mode ensures safe multi-user SKU generation
