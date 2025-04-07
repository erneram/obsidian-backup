# Reset tables:
---
Elimina todas las tablas existentes y las vuelve a crear
Command that delete all tables and creates them again.
- Destroy all the data
- Reset the counter of "id"
```sh
php artisan migrate:reset

```

# Run migrations:
---
Run migrations to create all tables 

```sh
php artisan migrate:fresh
```
this command reset tables, ids. Ideal for starting again all the tables


