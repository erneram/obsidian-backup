# Creating a model
---
```sh
php artisan make:model Producto
```
This command will create a file inside `app/models`

> when creating a model you should use singular and PascalCase; Producto, not Products

For creating Model, Controller, Seeder, Migration at the same time use 
```sh
php artisan make:model Producto --controller --seed --migration
```
---
## Add Attributes
```php
protected $fillable = [
    'invitation_id',
    'personable_type',
    'personable_id',
];
```
---
## Add Relationships

When creating name of functions you can simply name it like the table you want the relationship. Depending on the relation it will be in `PLURAL`or `SINGULAR

| Tipo de relación | Convención                                | Ejemplo real                    |
| ---------------- | ----------------------------------------- | ------------------------------- |
| `belongsTo`      | **Singular** del modelo relacionado       | `invitation()`, `user()`        |
| `hasOne`         | **Singular** del modelo relacionado       | `profile()`, `document()`       |
| `hasMany`        | **Plural** del modelo relacionado         | `vehicles()`, `accessPeriods()` |
| `morphTo`        | **Singular** que describe el propósito    | `personable()`, `scannedBy()`   |
| `morphMany`      | **Plural** del modelo que depende de este | `invitationPersons()`           |
| `morphOne`       | **Singular**                              | `photo()`                       |

### Common 
1. belongsTo
	```php
	public function invitation()
	{
    return $this->belongsTo(Invitation::class);
	}
	```
2. hasMany
	```php
	public function documents()
	{
    return $this->hasMany(GuestDocument::class);
	}
	```
3. hasOne
	```php
	public function profile()
	{
    return $this->hasOne(Profile::class);
	}
	```
### Polimorphic 
Migration example:
```php
Schema::create('invitation_persons', function (Blueprint $table) {
    $table->id();
    $table->morphs('personable'); // Crea personable_type y personable_id
    $table->foreignId('invitation_id')->constrained()->onDelete('cascade');
    $table->timestamps();
});
```
In `InvitationPerson`'s models, you should have
```php
public function personable()
{
    return $this->morphTo();
}
```