Here’s a schema design for managing **product expiry-driven stock movement** between the **central warehouse** and **zonal warehouses**, incorporating SKU features, categories, and expiry data.  

---

### **Schema Design**

#### 1. **Central Warehouse Data**  
This table tracks the inventory available in the central warehouse.  
| **Column Name**      | **Data Type**   | **Description**                                      |
|-----------------------|-----------------|------------------------------------------------------|
| `item_id`            | VARCHAR         | Unique identifier for each stock item.              |
| `sku_id`             | VARCHAR         | SKU code for the item.                              |
| `sku_category`       | VARCHAR         | Category of the SKU (e.g., Critical, Non-urgent).   |
| `batch_id`           | VARCHAR         | Batch identifier for traceability.                  |
| `expiry_date`        | DATE            | Expiry date of the item.                            |
| `quantity`           | INTEGER         | Total quantity available in stock.                  |
| `warehouse_location` | VARCHAR         | Physical location/section within the central warehouse. |
| `reorder_point`      | INTEGER         | Minimum stock level before triggering alerts.        |

---

#### 2. **Zonal Warehouse Data**  
This table tracks inventory and allocations for zonal warehouses.  
| **Column Name**      | **Data Type**   | **Description**                                      |
|-----------------------|-----------------|------------------------------------------------------|
| `zonal_id`           | VARCHAR         | Identifier for the zonal warehouse.                 |
| `sku_id`             | VARCHAR         | SKU code for the item.                              |
| `sku_category`       | VARCHAR         | Category of the SKU (e.g., Critical, Non-urgent).   |
| `batch_id`           | VARCHAR         | Batch identifier for traceability.                  |
| `expiry_date`        | DATE            | Expiry date of the item.                            |
| `allocated_quantity` | INTEGER         | Quantity allocated to the zonal warehouse.          |
| `current_stock`      | INTEGER         | Current stock available in the zonal warehouse.     |
| `reorder_point`      | INTEGER         | Minimum stock level before triggering restocks.      |

---

#### 3. **SKU Metadata**  
This table defines SKU-specific attributes for inventory management.  
| **Column Name**      | **Data Type**   | **Description**                                      |
|-----------------------|-----------------|------------------------------------------------------|
| `sku_id`             | VARCHAR         | Unique identifier for the SKU.                      |
| `sku_name`           | VARCHAR         | Descriptive name of the SKU.                        |
| `sku_category`       | VARCHAR         | SKU category (e.g., Critical, Routine, Rare).       |
| `default_threshold`  | INTEGER         | Default minimum stock threshold for all warehouses. |
| `fifo_applicable`    | BOOLEAN         | Whether FIFO rules apply (e.g., YES for perishable).|
| `expiry_sensitive`   | BOOLEAN         | Indicates if expiry-driven transfers are required.  |
| `critical_stock`     | BOOLEAN         | Indicates if the SKU is life-saving or critical.    |

---

### **Relationships Between Tables**
Here are the relationships to ensure consistency in the pull-based system:  
1. **`central_warehouse_data.sku_id`** → **`sku_metadata.sku_id`**  
   - Ensures central warehouse data adheres to SKU rules and features.  
2. **`zonal_warehouse_data.sku_id`** → **`sku_metadata.sku_id`**  
   - Links SKU metadata with zonal warehouse data for feature-driven allocations.  
3. **`zonal_warehouse_data.batch_id`** → **`central_warehouse_data.batch_id`**  
   - Tracks stock movement for batch traceability and expiry checks.  

---

### **Pull-Based Allocation Logic**  
1. **Trigger for Allocation:**  
   - Zonal warehouses monitor **`current_stock`**. If it falls below the **`reorder_point`**, they pull from the central warehouse.  

2. **Allocation Process:**  
   - **Step 1:** Identify SKUs below reorder thresholds at zonal warehouses.  
   - **Step 2:** Check for **expiry-driven prioritization** in the central warehouse (earliest expiry first).  
   - **Step 3:** Transfer quantities based on **batch availability** and FIFO logic.  

3. **Exceptions:**  
   - If no valid stock is available in the central warehouse (e.g., expired stock), trigger alerts for immediate procurement.

---

### Example Data for Tables

#### Central Warehouse Data  
| item_id | sku_id | sku_category | batch_id | expiry_date | quantity | warehouse_location | reorder_point |
|---------|--------|--------------|----------|-------------|----------|---------------------|---------------|
| 1       | VAX01  | Critical     | B001     | 2025-03-15  | 500      | A1                  | 100           |
| 2       | MED02  | Routine      | B002     | 2025-06-01  | 1000     | B3                  | 200           |

#### Zonal Warehouse Data  
| zonal_id | sku_id | sku_category | batch_id | expiry_date | allocated_quantity | current_stock | reorder_point |
|----------|--------|--------------|----------|-------------|---------------------|---------------|---------------|
| ZN01     | VAX01  | Critical     | B001     | 2025-03-15  | 100                 | 50            | 100           |
| ZN02     | MED02  | Routine      | B002     | 2025-06-01  | 200                 | 80            | 200           |

#### SKU Metadata  
| sku_id | sku_name        | sku_category | default_threshold | fifo_applicable | expiry_sensitive | critical_stock |
|--------|-----------------|--------------|-------------------|-----------------|------------------|----------------|
| VAX01  | Vaccine A       | Critical     | 100               | TRUE            | TRUE             | TRUE           |
| MED02  | Medicine B      | Routine      | 200               | FALSE           | FALSE            | FALSE          |
