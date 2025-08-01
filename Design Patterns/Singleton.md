Creational design pattern, ensures that you only have a instance and providing global access point to that instance.

---
## Real World Analogy
Government is an example of singleton pattern. One country can have only one official government. Is the global point of access that identifies the group of people in charge.

---
# Pseudocode
```pseudocode
class Database is 

	// Field that stores the singleton instance
	private static field instance: Database

	// Initialization code, connection to server or other stuff
	private constructor Database() is

	public statuc method getInstance() is
		if (Database.instance == null) then 
			// Ensure that hasn't yet been initialized by other thread
			acquireThreadLock() and then 
				if (Database.instance == null) then 
				Database.instance = New Database()
		return Database.instance

	// All database querys should go throught this method
	public method query(sql) is 
	
```

```pseudocode
class App is 
	method main() is 
		Database foo = Database.getInstance()
		foo.query("select...")

		Database bar = Database.getInstance()
		bar.query("select...")

		foo == bar // True
```

---
## Applicability
1. Use it when your program should have just a single instance available to all clients. (A single database object)
2. Disable the creation of new classes. This method either creates a new object or returns an existing one of it has already been created.
## Implementation
1. Add a private static field to the class for storing the singleton instance.
2. Declare a public static creation method, getting the singleton instance.
3. Implement "Lazy initialization" inside static method. Should create a new object on its first call and returns that instance on all subsequent calls.
4. Make constructor of the class private. The static method of the class will still be able to call the constructor but no the other objects.
5. Go into the main or client code and replace all direct calls to the singletons constructor with calls to its static creation method.

---
## Examples
## PHP
Singleton Implementation
```php
<?php
namespace // File path

class Singleton
{
	// Store the static field. 
	// Array to allow singleton to have subclasses
	private static $instances = [];

	// Constructor should always be private, prevent calls with `new`operator.
	protected function __construct() { }

	// Singletons should not be cloneable.
	protected function __clone() { }

	// Singletons should not be restorable from strings.
	public function __wakeup()
	{
		throw new \Exception("Cannot unserialize a singleton")
	}

	public statuc function getInstance(): Singleton
	{
		$clas = statuc::class;
		if(!isset(self::instances[$cls])) {
			self::$instances[$cls] = new static();
		}
		return self::$instances[$cls];
	}

	
	// Can be executed by the instance
	public function someOtherBusinessLogic () 
	{
		// ...
	}
}
```
Client code
```php
function clientCode()
{
	$s1 = Singleton::getInstance();
	$s2 = Singleton::getInstance();
	if($s1 === $s2) {
		echo "Singleton contains the same instance";
	} 
}
clientCode();
```
> Singleton contains the same instance
## Go
Singleton implementation
```go 
package main 

import ( "fmt" "sync")

var lock = &sync.Mutex{}

type single struct {
}

var singleInstance *single

func getInstance() *single {
	if singleInstance == nil {
		lock.Lock()
		defer lock.Unlock()
		if singleInstance == nil {
			fmt.Println("Creating single instance now")
			singleInstance = &single{}
		} else {
			fmt.Println("Single instance already created")
		}
	} else {
		fmt.Println("Single Instance already created")
	}
	return singleInstance
}
```
Client code
```go
package main

import ( "fmt" )

func main() {
	for i := 0; i < 30; i++ {
		go getInstance()
	}
	fmt.Scanln()
}
```