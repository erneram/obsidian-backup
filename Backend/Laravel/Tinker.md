Tinker allows you to interact with your entire Laravel application on the command line, including your Eloquent models, jobs, events, and more.
```sh
php artisan tinker
```

Common usage of tinker right now, accessing to data from models. 
```sh
# Import first the model:
use App\Models\User;

# Or directly usign the model:
User::select('atributes')->get()
```

When getting the data you can use methods like:
### Methods to Retrieve Multiple Records

- `get()` → Execute the query and return a **Collection** of models.
    
- `paginate($perPage)` → Return paginated results with total count (LengthAwarePaginator).
    
- `simplePaginate($perPage)` → Return simple pagination without total count.
    
- `cursor()` → Return a **LazyCollection** (memory efficient for large datasets).
    
- `chunk($size, $callback)` → Process records in chunks to reduce memory usage.
    
- `lazy()` → Similar to chunk but returns a LazyCollection.
    
- `lazyById()` → Lazy loading results ordered by ID.
    

---

### Methods to Retrieve a Single Record

- `first()` → Return the first result or `null`.
    
- `firstOrFail()` → Return first result or throw `ModelNotFoundException`.
    
- `find($id)` → Retrieve a model by primary key.
    
- `findOrFail($id)` → Retrieve by primary key or throw exception.
    
- `sole()` → Retrieve exactly one record (throws if 0 or more than 1).
    
- `value('column')` → Get a single column value from the first result.
    

---

### Column / Value Extraction

- `pluck('column')` → Retrieve a collection of values for a single column.
    
- `pluck('column', 'key')` → Retrieve key-value pairs.
    
- `select('columns')` → Specify columns to retrieve.
    
- `addSelect('column')` → Add more columns to the select statement.
    
- `distinct()` → Retrieve distinct results.
    

---

### Aggregate Methods (SQL Aggregates)

- `count()` → Count matching records.
    
- `sum('column')` → Get the sum of a column.
    
- `avg('column')` → Get the average value.
    
- `min('column')` → Get the minimum value.
    
- `max('column')` → Get the maximum value.
    
- `exists()` → Determine if records exist.
    
- `doesntExist()` → Determine if no records exist.
    

---

### Relationship Loading

- `with('relation')` → Eager load relationships.
    
- `withCount('relation')` → Add relationship count column.
    
- `load('relation')` → Lazy eager load relationships.
    
- `loadCount('relation')` → Lazy eager load relationship count.


# bcrypt on thinker

```

---

## ↩️ Navegación
- [[Backend/Backend|🛠️ Backend]] → [[HOME|🏠 HOME]]
