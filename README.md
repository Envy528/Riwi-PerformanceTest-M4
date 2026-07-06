# RiwiSupply S.A.S. – Relational Database Project

## Project Description

RiwiSupply S.A.S. is a company dedicated to the commercialization and national distribution of industrial supplies. Operational data (suppliers, products, warehouses, purchases and inventory movements) was previously kept in a single unstructured Excel file, which caused duplicated suppliers, inconsistent product names, repeated cities/warehouses and unreliable reporting.

This project analyzes that source file, normalizes the data up to Third Normal Form (3NF), designs an Entity-Relationship Model, and implements the resulting relational database in PostgreSQL.
##  Technologies Used
- PostgreSQL
- SQL
- Excel (data source and cleaning/staging)
- Draw.io (Entity-Relationship diagram)

##  Database Engine Used
PostgreSQL

Database name: bd_jorge_carmona_mulata

## ormalization Process

### Initial structure
The original file contained a single flat table raw_data with one row per inventory movement, mixing:
MovementDate, SupplierName, SupplierCity, Warehouse, WarehouseCity, ProductName, Category, Quantity, UnitPrice, MovementType, PurchaseOrder.

### Problems identified
- Suppliers registered multiple times with different formats Aceros del Norte S.A.S, Aceros del Norte, ACEROS NORTE
- Cities written inconsistently Cartagena, Ctg; Barranquilla, Barranquila, B/quilla.
- Products with inconsistent names Guante Nitrilo vs Guantes de Nitrilo; Disco Corte 4.5 vs Disco de Corte 4.5.
- Categories repeated with different labels Herramienta/Herramientas, Consumible/Consumibles, EPP/Elementos Protección.
- Redundant fields: city was stored twice per row (once for supplier, once for warehouse), and category/product/supplier/city names were repeated on every movement line instead of being stored once.

### Transformations applied
**1FN:** Each column now holds a single atomic value (no combined fields, no repeating groups). Redundant, denormalized text columns from raw_data were removed.

**2FN:** Every non-key attribute depends on the whole primary key. Master data (city, category) was separated so attributes like city_name or category_name are stored once, not repeated on every transactional row.

**3FN:** Transitive dependencies were removed by moving city out of suppliers/warehouses into an independent riwi_cities table, and category out of products into riwi_categories, referenced through foreign keys instead of repeated text.

### Final normalized result
Seven entities in 3NF:
riwi_cities, riwi_categories, riwi_suppliers, riwi_warehouses, riwi_products, riwi_inventory_movements, riwi_purchases.

## Database Structure

- riwi_cities = id (PK), riwi_city_name 
- riwi_categories = id (PK), riwi_category_name 
- riwi_suppliers = id (PK), riwi_supplier_name, riwi_id_city (FK)
- riwi_warehouses = id (PK), riwi_warehouse_name, riwi_id_city (FK)
- riwi_products = id (PK), riwi_product_name, riwi_id_category (FK)
- riwi_inventory_movements = id (PK), riwi_movement_type, riwi_quantity, riwi_unit_price
- riwi_purcharses = id (PK), riwi_purcharse_order, riwi_id_supplier (FK), riwi_id_warehouse (FK), riwi_id_product (FK), riwi_id_movement (FK)  

## Instructions to Create the Database

### Install PostgreSQL

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



## Developer Information
- **Full name:** Jorge Carmona
- **Clan:** Mulata

## Repository
GitHub URL: `https://github.com/Envy528/Riwi-PerformanceTest-M4.git`