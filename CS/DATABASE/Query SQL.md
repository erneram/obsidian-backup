
This method involves writing a SQL query to search the database's metadata for column names.
```sql
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE COLUMN_NAME = 'your_column_name';
```

## ↩️ Navegación
- [[CS/DATABASE/Database|🗄️ Database]] → [[CS/CS|💻 CS]] → [[HOME|🏠 HOME]]