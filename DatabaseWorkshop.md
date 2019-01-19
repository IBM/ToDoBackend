# Adding persistance to ToDoBackend with Swift-Kuery-ORM

So far, our ToDoBackend has been storing, retrieving, deleting and updating "to do" items in a local in-memory `Array`.

Now we will show you how to use our ORM (Object Relational Mapping) library, called Swift-Kuery-ORM, to store the "to do" items in a PostgreSQL database. This allows you to simplify the persistence of model objects with your server.

## Pre-Requisite

This is a follow on tutorial to our [ToDoBackend tutorial](https://github.com/IBM/ToDoBackend). Please complete that before proceeding with this tutorial.

## Installing PostgreSQL

In this tutorial you'll be using the [Swift Kuery PostgreSQL plugin](https://github.com/IBM-Swift/Swift-Kuery-PostgreSQL), so you will need PostgreSQL running on your local machine. You can install and start PostgreSQL as follows:

```
brew install postgresql
brew services start postgresql
```

## Persistance over local storage
Adding a database means your "todo" items are saved even after you close your server down. Also, depending on where your database is stored, different applications on different machines can access the data (for instance, in a cloud application).

### 1. Create a Database
Create a new PostgreSQL database for the project called `tododb`:
```
createdb tododb
```

### 2. Updating Package.swift
Your `Package.swift` needs two new Packages for the PostgreSQL and ORM to work. 
1. Open the `ToDoServer` > `Package.swift` file in Xcode.
2. Add the following two lines to the end of the dependencies section of the `Package.swift` file:
```swift
.package(url: "https://github.com/IBM-Swift/Swift-Kuery-ORM", from: "0.4.1"),
.package(url: "https://github.com/IBM-Swift/Swift-Kuery-PostgreSQL", from: "2.1.0"),
```
3. Declare these new packages in the "Application" target (note the lack of hyphens):
```swift
.target(name: "Application", dependencies: ["SwiftKueryPostgreSQL", "SwiftKueryORM", "KituraCORS", "KituraOpenAPI", "Kitura", "CloudEnvironment","SwiftMetrics","Health",]),
```
In order for Xcode to pick up the new dependencies, the Xcode project now needs to be regenerated.

1. Close Xcode
2. Rengerate the Xcode project and reopen:

```bash
cd ~/ToDoBackend/ToDoServer
swift package generate-xcodeproj
open ToDoServer.xcodeproj
```

### 3. Importing into the Application
1. In Xcode, open `Sources` > `Application` > `Application.swift`
2. Add imports for the two new libraries below the other import statements at the top of the file:
```swift
import SwiftKueryORM
import SwiftKueryPostgreSQL
```
### 4. Extending ToDo to conform to Model
The `Model` protocol is the key to using the ORM. The `model` protocol provides the functionality for saving and retrieving items to and from the database.

Earlier, we declared our `ToDo` struct, located in `Application` > `Model.swift`, to be Codable to simplify our RESTful routes for these objects on our server. The Model protocol extends what Codable does to work with the ORM, so now you also need the `ToDo` struct to conform to the `Model` protocol.

At the end of your `Application.swift` file (after the final `}`) extend your object by adding the following:

```swift
extension ToDo: Model {
}
```
This exposes new methods to ToDo for saving, finding, deleting etc. from a database.

### 5. Creating a connection with the database
Add the following code, to set up your database connection pool, to the end of the `Application.swift` file, below the final curly brace:

```swift
class Persistence {
	static func setUp() {
		let pool = PostgreSQLConnection.createPool(host: "localhost", port: 5432, options: [.databaseName("tododb")], poolOptions: ConnectionPoolOptions(initialCapacity: 10, maxCapacity: 50))
		Database.default = Database(pool)
	}
}
```
Add the following code to call the `setUp()` method and create a database table (which will be called ToDos) to the `postInit()` method within the `Application.swift` file:

```swift
Persistence.setUp()
do {
	try ToDo.createTableSync()
} catch let error {
	print("Table already exists. Error: \(String(describing: error))")
}
```
To check that a database table called `ToDos` has been created, run your Kitura server and then use `psql` from the command-line as follows:

```sql
psql tododb
SELECT * FROM "ToDos";
```
This should print the column names of the `ToDos` table with no data in (i.e. no rows).

Now we will start modifying the methods which handle requests so that they no longer use the in-memory `todoStore` array and instead store and retrieve data from the database table.

### 6. Adding  @escaping
All of the handlers now need to have `@escaping` added to their function signature, as the ORM uses an escaping method. Add `@escaping` to each method, after `completion:`. For example:
```swift
func storeHandler(todo: ToDo, completion: @escaping (ToDo?, RequestError?) -> Void ) { ...
func deleteAllHandler(completion: @escaping (RequestError?) -> Void ) { ...
```
### 7. storeHandler() updates
We will first update storeHandler() to use the database. Remove all of the code between the method's curly braces and paste in the following:
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
This code is similar to the original but uses `todo.save(completion)` which saves the `todo` object to the database and then calls the method's completion handler.
### 8. deleteOne(), deleteAll(), getOne(), getAll()
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
Each one now uses one call to the database to accomplish its task. The code is now simpler and easier to understand because each method uses only one call to the database to accomplish its task, and the ORM handles all the complexity of saving and retrieving items in the database.

### 9. updateHandler()  

This method is a little more complex as we need to fetch the `ToDo` object that is currently stored in our database, assess the differences between this version of the `ToDo` object and the patched version being passed into the method, make the changes and then commit the modified `ToDo` object to the database

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

Again, the logic is similar to before but with some added error handling for fetching from the database in case the `id` doesn't exist. 

### 10. Remove the todoStore property

The final step is to remove the `todoStore` property definition from the top of the `App`class within `Application.swift` as it is no longer needed.

### 11. Running the Tests

That's it! We have now modified all we need to have the ORM running inside our project. Run the Xcode project with ⌘R and use Terminal to launch the Test suite:

```bash
open ~/todo-backend-js-spec/index.html
```

Set the test target root to `http://localhost:8080/` and you should see them all pass, just as they did before we added a database connection.

You can use `psql` to check that the todo items are being correctly stored in the database:

```sql
psql tododb
SELECT * FROM "ToDos";
```

Congratulations! We have removed the project's dependency on a non-persistent storage option and updated it to use a persistent and accessible database, using Swift 4's Codable feature along side our ORM to maintain our ToDo type.

**Ready to learn about Docker and Kubernetes? [This guide](https://github.com/IBM/ToDoBackend/blob/master/DeployingToKube.md) will take you, step-by-step, through setting up a Kubernetes cluster using Docker for Desktop, building a Docker image of your Kitura app, and deploying releases using both remote and local Helm charts.**
