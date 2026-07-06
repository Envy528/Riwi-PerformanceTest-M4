# RiwiSupply S.A.S. – Relational Database Project

## 1. Project Description
RiwiSupply S.A.S. is a company dedicated to the commercialization and national distribution of industrial supplies. Operational data (suppliers, products, warehouses, purchases and inventory movements) was previously kept in a single unstructured Excel file, which caused duplicated suppliers, inconsistent product names, repeated cities/warehouses and unreliable reporting.

This project analyzes that source file, normalizes the data up to Third Normal Form (3NF), designs an Entity-Relationship Model, and implements the resulting relational database in PostgreSQL, including data loading, sample data manipulation scripts (insert/update/delete) and business-oriented SQL queries.

## 2. Technologies Used
- PostgreSQL
- SQL (DDL / DML)
- Excel (data source and cleaning/staging)
- Draw.io (Entity-Relationship diagram)

## 3. Database Engine Used
PostgreSQL

Database name: `bd_jorge_carmona_mulata`

## 4. Normalization Process

### 4.1 Initial structure
The original file contained a single flat table (`raw_data`) with one row per inventory movement, mixing:
`MovementDate, SupplierName, SupplierCity, Warehouse, WarehouseCity, ProductName, Category, Quantity, UnitPrice, MovementType, PurchaseOrder`.

### 4.2 Problems identified
- Suppliers registered multiple times with different formats (`Aceros del Norte S.A.S`, `Aceros del Norte`, `ACEROS NORTE`).
- Cities written inconsistently (`Cartagena`, `Ctg`; `Barranquilla`, `Barranquila`, `B/quilla`).
- Products with inconsistent names (`Guante Nitrilo` vs `Guantes de Nitrilo`; `Disco Corte 4.5` vs `Disco de Corte 4.5`).
- Categories repeated with different labels (`Herramienta`/`Herramientas`, `Consumible`/`Consumibles`, `EPP`/`Elementos Protección`).
- Redundant fields: city was stored twice per row (once for supplier, once for warehouse), and category/product/supplier/city names were repeated on every movement line instead of being stored once.

### 4.3 Transformations applied
**1FN:** Each column now holds a single atomic value (no combined fields, no repeating groups). Redundant, denormalized text columns from `raw_data` were removed.

**2FN:** Every non-key attribute depends on the whole primary key. Master data (city, category) was separated so attributes like `city_name` or `category_name` are stored once, not repeated on every transactional row.

**3FN:** Transitive dependencies were removed by moving city out of `suppliers`/`warehouses` into an independent `riwi_cities` table, and category out of `products` into `riwi_categories`, referenced through foreign keys instead of repeated text.

### 4.4 Final normalized result
Seven entities in 3NF:
`riwi_cities`, `riwi_categories`, `riwi_suppliers`, `riwi_warehouses`, `riwi_products`, `riwi_inventory_movements`, `riwi_purchases`.

## 5. Database Structure

| Table | Description | Key Columns |
|---|---|---|
| `riwi_cities` | Master list of cities | `riwi_id` (PK), `riwi_city_name` (UNIQUE) |
| `riwi_categories` | Master list of product categories | `riwi_id` (PK), `riwi_category_name` (UNIQUE) |
| `riwi_suppliers` | Suppliers, standardized | `riwi_id` (PK), `riwi_supplier_name` (UNIQUE), `riwi_id_city` (FK) |
| `riwi_warehouses` | Warehouses, standardized | `riwi_id` (PK), `riwi_warehouse_name` (UNIQUE), `riwi_id_city` (FK) |
| `riwi_products` | Products, standardized | `riwi_id` (PK), `riwi_product_name` (UNIQUE), `riwi_id_category` (FK) |
| `riwi_inventory_movements` | Quantity/price/type of each stock movement (IN/OUT) | `riwi_id` (PK), `riwi_movement_type`, `riwi_quantity`, `riwi_unit_price` |
| `riwi_purchases` | Links a purchase order to supplier, warehouse, product and its movement | `riwi_id` (PK), `riwi_purchase_order`, `riwi_movement_date`, FKs to all other tables |

**Relationships / cardinalities:**
- 1 city → N suppliers | 1 city → N warehouses
- 1 category → N products
- 1 supplier → N purchases | 1 warehouse → N purchases | 1 product → N purchases
- 1 inventory movement → 1 purchase (1:1, enforced with `UNIQUE` on `riwi_id_movement`)

## 6. Entity Relationship Model
See `ER_Model.pdf` (Draw.io export) included in this repository, showing entities, attributes, primary keys, foreign keys, relationships and cardinalities.

## 7. Instructions to Create the Database

### 7.1 Install PostgreSQL

**Windows**
1. Download the installer from the official site: https://www.postgresql.org/download/windows/
2. Run the installer and follow the setup wizard (keep the default port `5432`, host `localhost`).
3. Set a password for the `postgres` superuser when prompted — remember it, you'll need it to connect.
4. Optionally install **pgAdmin** (bundled with the installer) as a graphical client.
5. Once finished, open **SQL Shell (psql)** from the Start Menu. When prompted, press Enter to accept the defaults (`Server: localhost`, `Database: postgres`, `Port: 5432`, `Username: postgres`) and type the password you set.

**Ubuntu Linux**
1. Install PostgreSQL:
   ```bash
   sudo apt update
   sudo apt install postgresql postgresql-contrib
   ```
2. Start and enable the service:
   ```bash
   sudo systemctl start postgresql
   sudo systemctl enable postgresql
   ```
3. Set a password for the `postgres` role (needed to connect via `localhost`):
   ```bash
   sudo -i -u postgres
   psql
   ```
   ```sql
   ALTER USER postgres WITH PASSWORD 'your_password';
   \q
   ```
   ```bash
   exit
   ```
4. Connect using `localhost` instead of the local peer socket:
   ```bash
   psql -h localhost -U postgres -p 5432
   ```
   (The `-h localhost` forces a TCP connection so it prompts for the password, same as on Windows.)

### 7.2 Create and Populate the Database
1. From `psql`, run the DDL script:
   ```sql
   \i 01_ddl.sql
   ```
   (Use the full path to the file, e.g. `\i C:/riwi_project/01_ddl.sql` on Windows or `\i /home/user/riwi_project/01_ddl.sql` on Ubuntu.)
2. Connect to the new database:
   ```sql
   \c bd_jorge_carmona_mulata
   ```
3. Verify the tables were created:
   ```sql
   \dt riwi_*
   ```

## 8. Instructions to Load Data
Data is loaded from CSV files (exported from the cleaned Excel dimensions/facts) using PostgreSQL's `COPY` command, in dependency order (masters first, then transactional tables):

```sql
COPY riwi_cities(riwi_id, riwi_city_name) FROM 'cities.csv' DELIMITER ',' CSV HEADER;
COPY riwi_categories(riwi_id, riwi_category_name) FROM 'categories.csv' DELIMITER ',' CSV HEADER;
COPY riwi_suppliers(riwi_id, riwi_supplier_name, riwi_id_city) FROM 'suppliers.csv' DELIMITER ',' CSV HEADER;
COPY riwi_warehouses(riwi_id, riwi_warehouse_name, riwi_id_city) FROM 'warehouses.csv' DELIMITER ',' CSV HEADER;
COPY riwi_products(riwi_id, riwi_product_name, riwi_id_category) FROM 'products.csv' DELIMITER ',' CSV HEADER;
COPY riwi_inventory_movements(riwi_id, riwi_movement_type, riwi_quantity, riwi_unit_price) FROM 'inventory_movements.csv' DELIMITER ',' CSV HEADER;
COPY riwi_purchases(riwi_id, riwi_purchase_order, riwi_movement_date, riwi_id_supplier, riwi_id_warehouse, riwi_id_product, riwi_id_movement) FROM 'purchases.csv' DELIMITER ',' CSV HEADER;
```

This order guarantees referential integrity, since master tables (cities, categories) are loaded before the tables that reference them.

## 9. SQL Queries – Explanation

| # | Query | Business need |
|---|---|---|
| 1 | Available stock per product | Inventory staff needs current stock levels per product to plan future purchases. |
| 2 | Inventory movements with product and warehouse detail | Logistics supervisor needs to know movements per warehouse and the products involved. |
| 3 | Total purchased per supplier | Purchasing lead needs to know how much has been bought from each supplier. |
| 4 | Number of movements per warehouse | Operations manager needs to know which warehouses are most active. |
| 5 | Product with the highest purchase volume | Analyst needs to identify the product with the highest rotation. |
| 6 | Total inventory value per warehouse | Operations manager needs the economic value of inventory per warehouse. |

(Full SQL code for each query is in `06_queries.sql`.)

## 10. Developer Information
- **Full name:** Jorge Carmona
- **Clan:** Mulata

## 11. Repository
GitHub URL: `<add your repository URL here>`