# Data Model

## Overview

The data model supports inventory tracking, vendor relationships, order management, and BoM generation.

## Core Entities

### 1. Inventory

The primary inventory database storing individual components.

```rust
pub struct InventoryItem {
    pub sku: String,                    // Primary Key: RES-0000001
    pub category: String,               // "Resistor"
    pub name: String,                   // "1MΩ Carbon Film 1/4W"
    pub description: Option<String>,    // "General purpose resistor"
    pub specification: Option<String>,  // "1M Ohm, 1/4W, 5% tolerance"
    pub status: ItemStatus,             // Active, Discontinued, Obsolete
    pub quantity_on_hand: i32,          // Current stock level
    pub reorder_point: i32,             // Alert threshold
    pub created_at: DateTime,
    pub updated_at: DateTime,
    pub retired_at: Option<DateTime>,
}

pub enum ItemStatus {
    Active,
    Discontinued,
    Obsolete,
    OnBackorder,
}
```

**Database Table:**
```sql
CREATE TABLE inventory (
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

CREATE INDEX idx_inventory_category ON inventory(category);
CREATE INDEX idx_inventory_status ON inventory(status);
```

### 2. Vendors

Vendor information and part number mappings.

```rust
pub struct Vendor {
    pub vendor_id: String,              // Unique identifier
    pub name: String,                   // "Digi-Key", "Mouser"
    pub url: Option<String>,            // https://www.digikey.com
    pub contact_email: Option<String>,
    pub notes: Option<String>,
    pub is_active: bool,
}

pub struct VendorPartNumber {
    pub vendor_pn_id: String,           // PK
    pub vendor_id: String,              // FK to Vendors
    pub sku: String,                    // FK to Inventory
    pub vendor_part_number: String,     // "1/4W-1M-MFR123"
    pub vendor_price: Option<f64>,      // $0.05
    pub lead_time_days: Option<i32>,    // 2 days
    pub minimum_order_qty: i32,         // 1
    pub moq_price: Option<f64>,         // Price at MOQ
    pub last_updated: DateTime,
}
```

**Database Tables:**
```sql
CREATE TABLE vendors (
    vendor_id TEXT PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    url TEXT,
    contact_email TEXT,
    notes TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE vendor_part_numbers (
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

CREATE INDEX idx_vpn_sku ON vendor_part_numbers(sku);
CREATE INDEX idx_vpn_vendor ON vendor_part_numbers(vendor_id);
```

### 3. Orders

Client orders and order line items.

```rust
pub struct Order {
    pub order_id: String,               // AUTO-generated
    pub customer_name: String,
    pub order_date: DateTime,
    pub required_date: Option<DateTime>,
    pub status: OrderStatus,            // Draft, Submitted, In Progress, Complete, Cancelled
    pub notes: Option<String>,
    pub created_at: DateTime,
}

pub struct OrderLineItem {
    pub line_item_id: String,           // PK
    pub order_id: String,               // FK
    pub sku: String,                    // FK to Inventory
    pub quantity: i32,
    pub notes: Option<String>,
}

pub enum OrderStatus {
    Draft,
    Submitted,
    InProgress,
    Complete,
    Cancelled,
}
```

**Database Tables:**
```sql
CREATE TABLE orders (
    order_id TEXT PRIMARY KEY,
    customer_name TEXT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    required_date TIMESTAMP,
    status TEXT NOT NULL DEFAULT 'draft',
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_line_items (
    line_item_id TEXT PRIMARY KEY,
    order_id TEXT NOT NULL,
    sku TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    notes TEXT,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (sku) REFERENCES inventory(sku)
);

CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_oli_order ON order_line_items(order_id);
```

### 4. Bills of Materials (BoM)

Product BOMs and component requirements.

```rust
pub struct BillOfMaterials {
    pub bom_id: String,                 // PK
    pub product_name: String,
    pub product_code: Option<String>,
    pub description: Option<String>,
    pub created_at: DateTime,
}

pub struct BomLineItem {
    pub bom_line_id: String,            // PK
    pub bom_id: String,                 // FK
    pub sku: String,                    // FK to Inventory
    pub quantity_required: i32,
    pub notes: Option<String>,
}
```

**Database Tables:**
```sql
CREATE TABLE bills_of_materials (
    bom_id TEXT PRIMARY KEY,
    product_name TEXT NOT NULL,
    product_code TEXT,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE bom_line_items (
    bom_line_id TEXT PRIMARY KEY,
    bom_id TEXT NOT NULL,
    sku TEXT NOT NULL,
    quantity_required INTEGER NOT NULL,
    notes TEXT,
    FOREIGN KEY (bom_id) REFERENCES bills_of_materials(bom_id),
    FOREIGN KEY (sku) REFERENCES inventory(sku)
);

CREATE INDEX idx_bom_product ON bills_of_materials(product_name);
CREATE INDEX idx_bli_bom ON bom_line_items(bom_id);
```

## Relationships

```
┌─────────────────┐
│   Categories    │
│  (Resistor, etc)│
└────────┬────────┘
         │
         ├─→ SKU Counters (sequence tracking)
         │
┌────────▼──────────┐
│   Inventory       │◄─────────┐
│   (SKU PK)        │          │
└────────┬──────────┘          │
         │                     │
    ┌────┴───┬─────────────────┼────┐
    │        │                 │    │
    ▼        ▼                 ▼    ▼
┌────────┐ ┌──────┐      ┌────────┐ ┌────────┐
│Vendor  │ │Orders│      │  BoM   │ │Prices  │
│Part#   │ │Items │      │Items   │ │ History│
└────────┘ └──────┘      └────────┘ └────────┘
```

## Key Constraints

1. **SKU Uniqueness:** Every inventory item has exactly one SKU (never reused)
2. **Component Singularity:** Each physical component has only one inventory entry
3. **Vendor Flexibility:** Multiple vendors can supply the same SKU
4. **Referential Integrity:** All orders and BoMs reference valid SKUs
5. **Immutable History:** Deleted/retired items keep their SKUs and history

## Future Extensions

### Stock Movement History
```sql
CREATE TABLE stock_movements (
    movement_id TEXT PRIMARY KEY,
    sku TEXT NOT NULL,
    quantity_change INTEGER NOT NULL,
    reason TEXT,                       -- 'order', 'restock', 'adjustment', etc.
    reference_id TEXT,                 -- Order ID or other reference
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (sku) REFERENCES inventory(sku)
);
```

### Price History
```sql
CREATE TABLE price_history (
    price_history_id TEXT PRIMARY KEY,
    vendor_pn_id TEXT NOT NULL,
    price REAL NOT NULL,
    effective_date TIMESTAMP,
    FOREIGN KEY (vendor_pn_id) REFERENCES vendor_part_numbers(vendor_pn_id)
);
```

### Supplier Lead Time Tracking
```sql
CREATE TABLE supplier_performance (
    performance_id TEXT PRIMARY KEY,
    vendor_id TEXT NOT NULL,
    order_date TIMESTAMP,
    expected_delivery TIMESTAMP,
    actual_delivery TIMESTAMP,
    FOREIGN KEY (vendor_id) REFERENCES vendors(vendor_id)
);
```
