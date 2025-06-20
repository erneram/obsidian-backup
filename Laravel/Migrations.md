# Reset tables:
---
Elimina todas las tablas existentes y las vuelve a crear
Command that delete all tables and creates them again.
- Destroy all the data
- Reset the counter of "id"
```sh
php artisan migrate:reset
```

# Create migration
---
```
php artisan make:migration nombre_de_la_migracion
```
This command create a migration for a table, used to create tables, alter tables or any other thing to a table.

## Create migration for an specific table
```
php artisan make:migration nombre_de_la_migracion --create=nombre_tabla
```

# Running migrations
---
```sh
php artisan migrate
```
Run all migrations in the `database/migrations`folder

```sh
php artisan migrate:fresh
```
Reset tables, ids. Ideal for restarting all tables (Kind of dropping tables and creating them again) 
> **You will lose all information**

## To run a specific migrations do this:
```
php artisan migrate --path=database/migrations/2025_05_05_123456_nombre_archivo.php
```

## Other types of migrations...

```sh
php artisan migrate --path=database/migrations/tenant
```
Used to run a migration from an specific directory `database/migrations/tenant`


```sh
php artisan migrate --database=mysql
```
Used to run a migration to a specific database configured in `config/database.php`

```sh
php artisan migrate --database=tenant --path=/database/migrations/tenant
```
Use both commands, from a specific directory and to a specific database