# 🏭 Laravel Factories and Seeders: Step by Step Guide

This document explains how to create a **Factory** in Laravel, how to use it inside a **Seeder**, and includes a cookie analogy to make it easier to understand.

---

## 📌 What is a Factory?

A **Factory** is a class that defines how to generate fake data (`faker`) for a specific model.  
It serves as a blueprint to create test or seed records.

---

## 🍪 Analogy: Cookies and Molds

| Laravel Concept        | Analogy                |
|------------------------|------------------------|
| Factory                | The **mold** for making cookies |
| Seeder                 | The **baker** who uses the mold to make cookies |
| Database               | The **baking tray** where cookies are placed |

Without the mold (Factory), you must shape each cookie by hand.  
Without the baker (Seeder), no one uses the mold.  
Together, they are fast and efficient 🍪🔥

---

## ✅ Step-by-Step: Create and Use a Factory

### 1. Create the model (optional if you already have it)

```sh
php artisan make:model Vehicle -m
```
This creates:
- `app/Models/Vehicle.php`
- `database/migrations/xxxx_create_vehicles_table.php`

### 2. Create the Factory
```sh
php artisan make:factory VehicleFactory --model=Vehicle
```
Location: `database/factories/VehicleFactory.php`

### 3. Define fake data in the Factory
Edit `VehicleFactory.php`:

```sh
public function definition(): array 
{     
	return [
	'plate' => strtoupper($this->faker->bothify('???###')),
	'model' => $this->faker->numberBetween(2000, now()->year),
	'car_line' => $this->faker->word,
	'make' => $this->faker->randomElement([
		'Toyota', 'Ford', 'Tesla', 'Kia', 'Porsche', 'Nissan', 'Mazda'
		]),
	'color' => $this->faker->safeColorName(),
	]; 
}
```

### 4. Create the Seeder
```sh 
php artisan make:seeder VehicleSeeder
```
Location: `database/seeders/VehicleSeeder.php`

### 5. Use the Factory in the Seeder

Inside `VehicleSeeder.php`:

```sh
public function run(): void 
{     
	\App\Models\Vehicle::factory()->count(20)->create(); 
}
```
