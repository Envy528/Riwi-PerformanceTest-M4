# RiwiSupply S.A.S. – Relational Database Project

## Project Description

RiwiSupply S.A.S. is a company dedicated to the commercialization and national distribution of industrial supplies. Operational data (suppliers, products, warehouses, purchases and inventory movements) was previously kept in a single unstructured Excel file, which caused duplicated suppliers, inconsistent product names, repeated cities/warehouses and unreliable reporting.

This project analyzes that source file, normalizes the data up to Third Normal Form (3NF), designs an Entity-Relationship Model, and implements the resulting relational database in PostgreSQL.

## Technologies Used

- PostgreSQL
- SQL
- Excel (data source and cleaning/staging)
- Draw.io (Entity-Relationship diagram)

## Database Engine Used

PostgreSQL

Database name: bd_jorge_carmona_mulata

## Normalization Process

### Initial structure

The original file contained a single flat table raw_data with one row per inventory movement, mixing: MovementDate, SupplierName, SupplierCity, Warehouse, WarehouseCity, ProductName, Category, Quantity, UnitPrice, MovementType, PurchaseOrder.

### Problems identified

Suppliers were registered multiple times with different formats, such as Aceros del Norte S.A.S, Aceros del Norte and ACEROS NORTE. Cities were written inconsistently, for example Cartagena, Ctg, Barranquilla, Barranquila and B/quilla. Products had inconsistent names, like Guante Nitrilo vs Guantes de Nitrilo, or Disco Corte 4.5 vs Disco de Corte 4.5. Categories were repeated under different labels, such as Herramienta/Herramientas, Consumible/Consumibles and EPP/Elementos Protección. On top of that, fields were redundant: city was stored twice per row (once for the supplier, once for the warehouse), and category/product/supplier/city names were repeated on every movement line instead of being stored once.

### Transformations applied

First Normal Form (1NF) requires every column to hold a single atomic value, with no combined fields and no repeating groups. Checking raw_data, this was never actually violated: every cell already held a single value, so no transformation was needed here, it was just verified.

Second Normal Form (2NF) requires every non-key attribute to depend on the whole primary key. Since raw_data was a single table holding all the information, attributes like SupplierName, SupplierCity, Warehouse, WarehouseCity, ProductName and Category did not depend on the table's primary key. This was fixed by separating those attributes into their own tables, so each one only stores the attributes that actually depend on its primary key.

Third Normal Form (3NF) requires removing transitive dependencies, where a non-key attribute depends on another non-key attribute instead of depending directly on the key. City was not really an attribute of suppliers or warehouses, it is its own concept shared by both, and leaving it as free text duplicated in two places caused the inconsistent spellings mentioned above. The same happened with Category, which is its own concept and not just a text label glued to each product. Both were extracted into their own tables (riwi_cities and riwi_categories), referenced through foreign keys instead of repeated text.

### Final normalized result

Seven entities in 3NF: riwi_cities, riwi_categories, riwi_suppliers, riwi_warehouses, riwi_products, riwi_inventory_movements, riwi_purchases.

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

Windows

1. Download the installer from the official site: https://www.postgresql.org/download/windows/
2. Run the installer and follow the setup wizard (keep the default port 5432, host localhost).
3. Set a password for the postgres superuser when prompted, remember it, you'll need it to connect.
4. Optionally install pgAdmin (bundled with the installer) as a graphical client.
5. Once finished, open SQL Shell (psql) from the Start Menu. When prompted, press Enter to accept the defaults (Server: localhost, Database: postgres, Port: 5432, Username: postgres) and type the password you set.

Ubuntu Linux

Install PostgreSQL:
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

Start and enable the service:
```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Set a password for the postgres role (needed to connect via localhost):
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

Connect using localhost instead of the local peer socket:
```bash
psql -h localhost -U postgres -p 5432
```
The -h localhost flag forces a TCP connection so it prompts for the password, same as on Windows.

## Instructions to Load Data

1. Dimension tables (riwi_cities, riwi_categories, riwi_suppliers, riwi_warehouses, riwi_products)

Only INSERT INTO ... VALUES was used for each of these, due to their small amount of data. Run them in this order, since later tables reference earlier ones through foreign keys: riwi_cities and riwi_categories first, then riwi_suppliers, riwi_warehouses and riwi_products.

2. Fact tables (riwi_inventory_movements, riwi_purcharses)

Due to a larger amount of data, a CSV import was used instead. The steps followed in pgAdmin were:

1. Navigate to bd_jorge_carmona_mulata > Schemas > public > Tables.
2. Right-click the target table (e.g. riwi_purcharses) and select Import/Export Data...
3. Select Import.
4. In Filename, browse to the corresponding .csv file.
5. Set Format to csv and Encoding to UTF8.
6. In the Options tab, enable Header, since the first row of the CSV contains column names, not data.
7. Click OK.

riwi_inventory_movements needs to be loaded before riwi_purcharses, since riwi_purcharses references it through riwi_id_movement.

## Explanation of Each SQL Query

Query 1, available stock per product. Calculates net stock by summing riwi_quantity as positive for IN movements and negative for OUT movements, grouped by product. This answers the inventory manager's need to know current stock in order to plan future purchases.

Query 2, inventory movements with product and warehouse detail. Joins riwi_inventory_movements, riwi_purcharses, riwi_products and riwi_warehouses to show, for each movement type/product/warehouse combination, how many times that movement occurred. This answers the logistics supervisor's need to see movements per warehouse together with the products involved.

Query 3, total purchased per supplier. Joins riwi_suppliers with riwi_purcharses and riwi_inventory_movements, filters to IN movements only (actual purchases received), and sums unit_price times quantity per supplier. This answers the purchasing lead's need to know how much has been bought from each supplier.

Query 4, number of movements registered per warehouse. Joins riwi_warehouses with riwi_purcharses and riwi_inventory_movements, counting rows per warehouse. This answers operations management's need to identify the most active warehouses.

Query 5, product with the highest purchase volume. Counts how many separate purchase transactions each product appears in (COUNT(*)), rather than the total quantity purchased. This reflects an interpretation of "rotación" (turnover) as purchase frequency: a product bought often in small amounts is considered higher-turnover than one bought rarely in large amounts. This answers the analyst's need to identify the product with the highest turnover.

Query 6, total inventory value per warehouse. Joins riwi_warehouses with riwi_purcharses and riwi_inventory_movements, filters to OUT movements, and sums unit_price times quantity per warehouse. This answers the operations manager's need to know the economic value of inventory distributed across warehouses.

## Developer Information

Full name: Jorge Carmona
Clan: Mulata

## Repository

GitHub URL: `https://github.com/Envy528/Riwi-PerformanceTest-M4.git`
