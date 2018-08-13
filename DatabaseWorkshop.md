# Adding persistance to ToDoBackend with Swift-Kuery-ORM

<p align="center">
<img src="https://www.ibm.com/cloud-computing/bluemix/sites/default/files/assets/page/catalog-swift.svg" width="120" alt="Kitura Bird">
</p>

<p align="center">
<a href= "http://swift-at-ibm-slack.mybluemix.net/"> 
    <img src="http://swift-at-ibm-slack.mybluemix.net/badge.svg"  alt="Slack"> 
</a>
</p>

## Prerequisite
This is a follow on tutorial to our [ToDoBackend tutorial](https://github.com/IBM/ToDoBackend). Please complete that before proceeding with this tutorial.

## Installing PostgreSQL

Before we start, ensure you have PostgreSQL installed and running. In Terminal, run:
```
brew services start postgresql
```

If it's already running then skip ahead to step 1. If you do not have it installed then run the following from Terminal:
```
brew install postgresql
brew services start postgresql
```

## Persistance over local storage
Adding a database connection means your state remains after you close your server down, and depending on where your database is stored, different applications on different machines can access the data (for instance, in a Cloud application).

### Step 1 - Create a Database
Create a new database for the project called `tododb` using your Terminal:
```
createdb tododb
```

### Step 2 - Updating Package.swift
Your `Package.swift` needs two new Packages for the PostgreSQL and ORM to work. 
1. Open the `ToDoServer` > `Package.swift` file
2. Add the following two lines to the end of the Package.swift's dependencies section:
```swift
.package(url: "https://github.com/IBM-Swift/Swift-Kuery-ORM", from: "0.3.1"),
.package(url: "https://github.com/IBM-Swift/Swift-Kuery-PostgreSQL", from: "1.2.0"),
```
3. Declare these new packages in the "Application" target (note the lack of hyphens):
```swift
.target(name: "Application", dependencies: ["SwiftKueryPostgreSQL", "SwiftKueryORM", "KituraCORS", "KituraOpenAPI", "Kitura", "CloudEnvironment","SwiftMetrics","Health",]),
```
4. We need to fetch the new dependencies and generate a new Xcode Project file. To do so, make sure Xcode is closed and:
```bash
cd ~/ToDoBackend/ToDoServer
swift package generate-xcodeproj
open ToDoServer.xcodeproj
```

### Step 3 - Importing into the Application
1. In Xcode, open `Sources` > `Application` > `Application.swift`
2. Add imports for the two new libraries below the other import statements at the top of the file:
```swift
import SwiftKueryORM
import SwiftKueryPostgreSQL
```
### Step 4 - Extending ToDo to conform to Model
In order for your `ToDo` struct located in `Application` > `Model.swift`, we need to conform `ToDo` to Swift-Kuery-ORMs Model protocol. At the bottom of `Application.swift` add:
```swift
extension ToDo: Model {
}
```
This exposes new methods to ToDo for saving, finding, deleting etc. from a database.

### Step 5 - Creating a connection with the database
Add the following Class to the bottom of the file, outside the `App` classes final curly brace:
```swift
class Persistence {
	static func setUp() {
		let pool = PostgreSQLConnection.createPool(host: "localhost", port: 5432, options: [.databaseName("tododb")], poolOptions: ConnectionPoolOptions(initialCapacity: 10, maxCapacity: 50, timeout: 10000))
		Database.default = Database(pool)
	}
}
```
This code is going to create a connection pool and set it as the default database. We now need to call the `setUp()` method inside  `App`s `postInit()` method:
```swift
		Persistence.setUp()
		do {
			try ToDo.createTableSync()
		} catch let error {
			print("Database already exists.")
		}
```
This will create a table for us. We can now begin modifying the methods to switch saving to the `todoStore` and instead save to the database.

To test everything is working as it should, we can run the Kitura server and check that a table called `ToDos` is created:
```sql
psql tododb
SELECT * FROM "ToDos"
```
This should print the columns of a table, with no data in any of the colums. We will change the methods for handling requests now to store and retrieve data from this database.

### Step 6 - Adding  @escaping
All of the handlers now need to have `@escaping` added to their function signature, as the ORM uses an escaping method. Add `@escaping` to each method, after `completion:`. For example:
```swift
func storeHandler(todo: ToDo, completion: @escaping (ToDo?, RequestError?) -> Void ) { ...
func deleteAllHandler(completion: @escaping (RequestError?) -> Void ) { ...
```
### Step 7 - storeHandler() updates
We will first update storeHandler() to use the database. Remove all of the code between the methods curly braces and paste in the following:
```swift
		var todo = todo
		if todo.completed == nil {
			todo.completed = false
		}
		todo.id = nextId
		todo.url = "http://localhost:8080/\(nextId)"
		nextId += 1
		todo.save(completion)
```
This code is similar to the original but uses `todo.save(completion)` which saves the `todo` object to the database and then calls the methods completion handler.
### Step 8 - deleteOne(), deleteAll(), getOne(), getAll()
These methods are easy to update as their logic can happen with one call using the ORM! 
```swift
	func deleteAllHandler(completion: @escaping (RequestError?) -> Void ) {
		ToDo.deleteAll(completion)
	}

	func deleteOneHandler(id: Int, completion: @escaping (RequestError?) -> Void ) {
		ToDo.delete(id: id, completion)
	}
	
	func getAllHandler(completion: @escaping ([ToDo]?, RequestError?) -> Void ) {
		ToDo.findAll(completion)
	}
	
	func getOneHandler(id: Int, completion: @escaping(ToDo?, RequestError?) -> Void ) {
		ToDo.find(id: id, completion)
	}
```
Each one now uses one call to the database to accomplish it's task. The code is also easy to understand just from reading the calls happening inside the methods.

### Step 9 - updateHandler()  

This one has a little more going on as we need to fetch the version of the `ToDo` currently stored in our database, assess the differences between this version of the `ToDo` and the patched version being passed into the method, make the changes and then commit the modified `ToDo` back to the database.

```swift
	func updateHandler(id: Int, new: ToDo, completion: @escaping (ToDo?, RequestError?) -> Void ) {

		ToDo.find(id: id) { (preExistingToDo, error) in
			if error != nil {
				return completion(nil, .notFound)
			}
			
			guard var oldToDo = preExistingToDo else {
				return completion(nil, .notFound)
			}
			
			guard let id = oldToDo.id else {
				return completion(nil, .internalServerError)
			}
			
			oldToDo.user = new.user ?? oldToDo.user
			oldToDo.order = new.order ?? oldToDo.order
			oldToDo.title = new.title ?? oldToDo.title
			oldToDo.completed = new.completed ?? oldToDo.completed
			
			oldToDo.update(id: id, completion)
			
		}
	}
```

Again, the logic is similar to before but with some added error handling for fetching from the database incase the `id` doesn't exist. 

The final step is now to remove the `todoStore` variable from the top of the `App` Class.

### Step 10 - Running the Tests

That's it! We have now modified all we need to have the ORM running inside our project. Run the Xcode project with ⌘R and use Terminal to launch the Test suite:

```bash
open ~/todo-backend-js-spec/index.html
```

Set the test target root to `http://localhost:8080/` and you should see them all pass, just as they did before we added a database connection.

Congratulations! We have removed the projects dependency on a volatile, local storage option and updated it to use a persistent and accessible database, using Swift 4s Codable feature to maintain our ToDo type at all times.