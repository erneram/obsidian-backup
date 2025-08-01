e# Create a seeder:
---
This will create a .php inside `database/seeders/NameSeeder.php`
```sh
php artisan make:seeder NameSeeder
```

# Modify Seeder with whatever you want
---
```sh
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\NameModel;
use App\Models\Name2Model;
use Faker\Factory as Faker;

class InventoryMovementSeeder extends Seeder
{
    public function run()
    {
        $faker = Faker::create();
        $name2 = Name2Model::all();

        for ($i = 0; $i < 100; $i++) {
            NameModel::create([
                'column1' => $name2->random()->id,
                'column2' => rand(1, 6),
            ]);
        }
    }
}
```
# Run seeder:
---
Run this command for running a specific seeder
```sh
php artisan db:seed --class=NameSeeder
```
# Run more than one seeder:
---
use this command to run more seeders but be sure that the seeder is inside `database/seeders/DatabaseSeeder.php`

```sh
php artisan db:seed
```
or 
```sh
php artisan db:seed --database=tenant
```

`DatabaseSeeder.php`
```sh
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        // Llamar a los seeders específicos
        $this->call([
            NameSeeder::class,
            Name2Seeder::class,
            Nam3Seeder::class,
            Name4Seeder::class,
            Name5Seeder::class,
        ]);
    }
}

```
