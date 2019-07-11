# Building a "ToDo" Backend with Kitura

<p align="center">
<img src="https://www.ibm.com/cloud-computing/bluemix/sites/default/files/assets/page/catalog-swift.svg" width="120" alt="Kitura Bird">
</p>

<p align="center">
<a href= "http://swift-at-ibm-slack.mybluemix.net/">
    <img src="http://swift-at-ibm-slack.mybluemix.net/badge.svg"  alt="Slack">
</a>
</p>

# Workshop Table of Contents:

1. [Build your Kitura app](https://github.com/IBM/ToDoBackend/blob/master/DeployingToKube.md)
2. [Connect it to a SQL database](https://github.com/IBM/ToDoBackend/blob/master/Workshop.md)
3. [Build your app into a Docker image and deploy it on Kubernetes.](https://github.com/IBM/ToDoBackend/blob/master/DatabaseWorkshop.md)
4. [Enable monitoring through Prometheus/Graphana](https://github.com/IBM/ToDoBackend/blob/master/DeployingToKube.md)

## Run the tests
In order to implement a ToDo Backend, a server is required that provides support for storing, retrieving, deleting and updating "to do" items. The ToDoBackend project doesn't provide a specification for how the server must respond, rather it provides a set of tests which the server must pass. The "todo-backend-js-spec" project provides those tests.

### 1. Run the ToDo-Backend Tests:

1. Open the tests in a web browser: `open ~/todo-backend-js-spec/index.html`
2. Set a "test target root" of `http://localhost:8080`
3. Click "run tests".

All the tests should fail. The first error reported should be as follows:

:x: `the api root responds to a GET (i.e. the server is up and accessible, CORS headers are set up)`

```
AssertionError: expected promise to be fulfilled but it was rejected with [Error:

GET http://localhost:8080/
FAILED

The browser failed entirely when make an AJAX request.
```

This shows that the tests made a `GET` request to `http://localhost.com:8080`, but it failed with no response. This is expected as there is no server running yet - we're going to fix that in a moment!

In the instructions below, reloading the page will allow you to re-run the ToDo-Backend tests.

## Building a Kitura server

Implementing a compliant ToDo Backend is an incremental task, with the aim being to pass more of the testsuite at each step. The first step is to build and run your Kitura server so it can respond to requests.

### 1. Run your Kitura server

1. Open the ToDoServer project in Xcode.
    ```
    cd ~/ToDoBackend/ToDoServer  
    open ToDoServer.xcodeproj
    ```

2. Run your Kitura server in Xcode:
    1) Change the selected target from "ToDoServer-Package" to the "ToDoServer > MyMac".
    2) Press the `Run` button or use the `⌘+R` key shortcut.
    3) Select "Allow incoming network connections" if you are prompted.

3. Check that some of the standard Kitura URLs are running:
    * Kitura splash screen: [http://localhost:8080/](http://localhost:8080/)
    * Kitura monitoring dashboard: [http://localhost:8080/swiftmetrics-dash/](http://localhost:8080/swiftmetrics-dash/)
    * Kitura health API: [http://localhost:8080/health](http://localhost:8080/health)

### 2. Add support for OpenAPI

**Important**: If you started your project by choosing the "OpenAPI" tile in the Kitura desktop app, you can skip to Section 3.

[OpenAPI](https://www.openapis.org/) is the most popular way to document RESTful web services. The OpenAPI ecosystem provides a broad range of tools and services for developers across the API lifecycle.

Kitura provides a package which makes it easy to add OpenAPI support to your application. Let's add OpenAPI.

1. Open the `ToDoServer` > `Package.swift` file
2. Add the following to the end of the dependencies section of the `Package.swift` file:
    ```swift
        .package(url: "https://github.com/IBM-Swift/Kitura-OpenAPI.git", from: "1.0.0")
    ```
3. Update the target dependencies for the "Application" target to the following (note the lack of hyphen in KituraOpenAPI):
    ```swift
    .target(name: "Application", dependencies: ["KituraOpenAPI", "Kitura", "CloudEnvironment", "Health", "SwiftMetrics" ]),
    ```

In order for Xcode to pick up the new dependency, the Xcode project now needs to be regenerated.

1. Close Xcode.
2. Regenerate the Xcode project and reopen:
    ```
    cd ~/ToDoBackend/ToDoServer  
    swift package generate-xcodeproj
    open ToDoServer.xcodeproj
    ```

Now we need to check if OpenAPI is enabled in our Kitura server.

1. Open the `Sources` > `Application` > `Application.swift` file
2. Check if there is an import for the KituraOpenAPI library to the start of the file:
   ```swift
   import KituraOpenAPI
   ```
3. Make sure if there is the following code into the end of the `postInit()` function after the call to `initializeHealthRoutes()`:  
   ```swift
   KituraOpenAPI.addEndpoints(to: router)
   ```

4. Re-run the server project in Xcode:
    1) Edit the scheme again and select a Run Executable of "ToDoServer".
    2) Run the project, then "Allow incoming network connections" if prompted.

### 3. Try out OpenAPI in Kitura

Now, you can open [http://localhost:8080/openapi](http://localhost:8080/openapi) and view the live OpenAPI specification of your Kitura application in JSON format.

You can also open [http://localhost:8080/openapi/ui](http://localhost:8080/openapi/ui) and view SwaggerUI, a popular API development tool. You will see one route defined: the GET `/health` route you visited earlier. Click on the route to expand it, then click "Try it out!" to query the API from inside SwaggerUI.

You should see a Response Body in JSON format, like:

```
{
  "status": "UP",
  "details": [],
  "timestamp": "2018-06-04T16:03:17+0000"
}
```

and a Response Code of 200.

Congratulations, you have added OpenAPI support to your Kitura application and used SwaggerUI to query a REST API!

### 4. Add Cross Origin Resource Sharing (CORS) Support

Re-run the ToDo-Backend tests by reloading the test page in your browser.

The first test should still fail with the following:

:x: `the api root responds to a GET (i.e. the server is up and accessible, CORS headers are set up)`

```
AssertionError: expected promise to be fulfilled but it was rejected with [Error:

GET http://localhost:8080
FAILED

The browser failed entirely when make an AJAX request.
Either there is a network issue in reaching the url, or the
server isn't doing the CORS things it needs to do.
```

This test is still failing, even though the server is responding on `localhost:8080`. This is because Cross Origin Resource Sharing (CORS) is not enabled.

By default, web servers only serve content to web pages that were served by that web server. In order to allow other web pages, such as the ToDo-Backend test page, to connect to the server, [Cross Origin Resource Sharing (CORS)](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) must be enabled.

Kitura provides a package which makes it easy to enable CORS in your application. Let's add CORS to your project.

1. Open the `ToDoServer` > `Package.swift` file
2. Add the following to the end of the dependencies section of the `Package.swift` file:
    ```swift
        .package(url: "https://github.com/IBM-Swift/Kitura-CORS.git", from: "2.1.0"),
    ```
3. Update the target dependencies for the "Application" target to the following (note the lack of hyphen in KituraCORS):
    ```swift
    .target(name: "Application", dependencies: ["KituraCORS", "KituraOpenAPI", "Kitura", "CloudEnvironment", "Health", "SwiftMetrics" ]),
    ```

In order for Xcode to pick up the new dependency, the Xcode project now needs to be regenerated.

1. Close Xcode.
2. Regenerate the Xcode project and reopen:
    ```
    cd ~/ToDoBackend/ToDoServer  
    swift package generate-xcodeproj
    open ToDoServer.xcodeproj
    ```

Now we need to enable CORS in our Kitura server.

1. Open the `Sources` > `Application` > `Application.swift` file
2. Add an import for the CORS library to the start of the file:
   ```swift
   import KituraCORS
   ```
3. Add the following code at the start of the `postInit()` function:  
   ```swift
   let options = Options(allowedOrigin: .all)
   let cors = CORS(options: options)
   router.all("/*", middleware: cors)
   ```

4. Re-run the server project in Xcode  
    1) Edit the scheme again and select a Run Executable of "ToDoServer". 2) Run the project, then "Allow incoming network connections" if you are prompted.

5. Re-run the tests by reloading the test page in your web browser.

The first test should now be passing!  But the second test is failing:

:x: `the api root responds to a POST with the todo which was posted to it`

In order to fix this, we need to implement a `POST` request that saves a ToDo item.

### 5. Add Support for handling a POST request on `/`

REST APIs typically consist of an HTTP request using a verb such as `POST`, `PUT`, `GET` or `DELETE` along with a URL and an optional data payload. The server then handles the request and responds with an optional data payload.

A request to store data typically consists of a POST request with the data to be stored, which the server then handles and responds with a copy of the data that has just been stored. This means we need to define a `ToDo` type, register a  handler for POST requests on `/`, and implement the handler to store the data.

1. Define a data type for the ToDo items:
   1. Select the Application folder in the left hand explorer in Xcode
   2. Select `File` > `New` > `File...` from the pull down menu
   3. Select `Swift File` and click `Next`
   4. Name the file `Models.swift`, change the `Targets` from `ToDoServerPackageDescription` to `Application`, then click `Create`
   5. Add the following to the created file:
   ```swift
   public struct ToDo : Codable, Equatable {
       public var id: Int?
       public var user: String?
       public var title: String?
       public var order: Int?
       public var completed: Bool?
       public var url: String?

       public static func ==(lhs: ToDo, rhs: ToDo) -> Bool {
           return (lhs.title == rhs.title) && (lhs.user == rhs.user) && (lhs.order == rhs.order) && (lhs.completed == rhs.completed) && (lhs.url == rhs.url) && (lhs.id == rhs.id)
       }
   }
   ```
   This creates a struct for the ToDo items that uses Swift 4's `Codable` capabilities.

2. Create an in-memory data store for the ToDo items
   1. Open the `Sources` > `Application` > `Application.swift` file
   2. Add `todoStore`, `nextId` and `workerQueue` properties into the App class. On the line below `let cloudEnv = CloudEnv()` add:
   ```swift
   private var todoStore: [ToDo] = []
   private var nextId: Int = 0
   private let workerQueue = DispatchQueue(label: "worker")
   ```
   3. To be able to use `DispatchQueue` on Linux, add the following `import` statement to the start of the file:
   ```swift
   import Dispatch
   ```
   4. Add a helper method at the end of the class, before the last closing brace
   ```swift
    func execute(_ block: (() -> Void)) {
       workerQueue.sync {
           block()
       }
    }
   ```
   This will be used to make sure that access to the todoStore is serialized, so the app does not crash on concurrent requests.

3. Register a handler for a `POST` request on `/` that stores the ToDo item data.
   1. Add the following into the `postInit()` function:
   ```swift
   router.post("/", handler: storeHandler)
   ```
   2. Implement the `storeHandler()` that receives a ToDo, and returns the stored ToDo.
   Add the following as a function in the App class:
   ```swift
    func storeHandler(todo: ToDo, completion: (ToDo?, RequestError?) -> Void ) {
        var todo = todo
        if todo.completed == nil {
            todo.completed = false
        }
        todo.id = nextId
        todo.url = "http://localhost:8080/\(nextId)"
        nextId += 1
        execute {
            todoStore.append(todo)
        }
        completion(todo, nil)
    }
   ```
   This expects to receive a ToDo struct from the request, sets `completed` to false if it is `nil` and adds a `url` value that informs the client how to retrieve this ToDo item in the future.  
   The handler then returns the updated ToDo item to the client.

4.  Run the project and rerun the tests by reloading the test page in the browser.

The first three tests should now pass.

Open SwaggerUI again at [http://localhost:8080/openapi/ui](http://localhost:8080/openapi/ui) and expand the new POST route on `/`. Paste the following JSON into the "input" text box:

```
{ "title": "mow the lawn" }
```

Click "Try it out!" and view the response body below.  You should see a JSON object representing the new ToDo item you created in the store:

```
{
  "id": 0,
  "title": "mow the lawn",
  "completed": false,
  "url": "http://localhost:8080/0"
}
```

Congratulations, you have successfully added a ToDo item to the store using SwaggerUI!

Going back to the testsuite webpage, the next failing test says this:

:x: `after a DELETE the api root responds to a GET with a JSON representation of an empty array`

In order to fix this, handlers for `DELETE` and `GET` requests are needed.

### 6. Add Support for handling a DELETE request on `/`

A request to delete data typically consists of a DELETE request. If the request is to delete a specific item, a URL encoded identifier is normally provided (eg. '/1' for the item with ID 1). If no identifier is provided, it is a request to delete all of the items.

In order to pass the next test, the ToDoServer needs to handle a `DELETE` on `/` resulting in removing all stored ToDo items.

Register a handler for a `DELETE` request on `/` that empties the ToDo item data.

1. Add the following into the `postInit()` function:
```swift
router.delete("/", handler: deleteAllHandler)
```

2. Implement the `deleteAllHandler()` that empties the todoStore  
Add the following as a function in the App class:  
```swift
func deleteAllHandler(completion: (RequestError?) -> Void ) {
    execute {
        todoStore = []
    }
    completion(nil)
}
```

Build and run your application again, then reload SwaggerUI to see your new DELETE route. Expand the route and click "Try it out!" to delete the contents of the store. You should see a Response Code of 204, indicating that the server successfully fulfilled the request.

### 7. Add Support for handling a GET request on `/`

A request to load all of the stored data typically consists of a `GET` request with no data, which the server then handles and responds with an array of all the data in the store.

1. Register a handler for a `GET` request on `/` that loads the data  
   Add the following into the `postInit()` function:  
   ```swift
	router.get("/", handler: getAllHandler)
   ```
2. Implement the `getAllHandler()` that responds with all of the stored ToDo items as an array.      
   Add the following as a function in the App class:
   ```swift
    func getAllHandler(completion: ([ToDo]?, RequestError?) -> Void ) {
        completion(todoStore, nil)
    }
   ```
3.  Run the project and re-run the tests by reloading the test page in the browser.

The first seven tests should now pass, with the eighth test failing:  
:x: `each new todo has a url, which returns a todo`

```
GET http://localhost:8080/0
FAILED

404: Not Found (Cannot GET /0.)
```

Refresh SwaggerUI again and view your new GET route. Clicking "Try it out!" will return the empty array (because you just restarted the application and the store is empty), but experiment with using the POST route to add ToDo items then viewing them by running the GET route again. REST APIs are easy!

### 8. Add Support for handling a `GET` request on `/:id`

The next failing test is trying to load a specific ToDo item by making a `GET` request with the ID of the ToDo item that it wishes to retrieve, which is based on the ID in the `url` field of the ToDo item set when the item was stored by the earlier `POST` request. In the test above the reqest was for `GET /0` - a request for id 0.

Kitura's Codable Routing is able to automatically convert identifiers used in the `GET` request to a parameter that is passed to the registered handler. As a result, the handler is registered against the `/` route, with the handler taking an extra parameter.

1. Register a handler for a `GET` request on `/`:
   ```swift
    router.get("/", handler: getOneHandler)
   ```

2. Implement the `getOneHandler()` that receives an `id` and responds with a ToDo item:
    ```swift      
    func getOneHandler(id: Int, completion: (ToDo?, RequestError?) -> Void ) {
        guard let todo = todoStore.first(where: { $0.id == id }) else {
            return completion(nil, .notFound)
        }
        completion(todo, nil)
    }
    ```

3.  Run the project and re-run the tests by reloading the test page in the browser.

The first nine tests now pass. The tenth fails with the following:  
:x: `can change the todo's title by PATCHing to the todo's url`  

```
PATCH http://localhost:8080/0
FAILED

404: Not Found (Cannot PATCH /0.)
```

Refresh SwaggerUI and experiment with using the POST route to create ToDo items, then using the GET route on `/{id}` to retrieve the stored items by ID.

### 9. Add Support for handling a `PATCH` request on `/:id`

The failing test is trying to `PATCH` a specific ToDo item. A `PATCH` request updates an existing item by updating any fields sent as part of the `PATCH` request. This means that a field by field update needs to be done.

1.  Register a handler for a `PATCH` request on `/`:
   ```swift
   router.patch("/", handler: updateHandler)
   ```
2. Implement the `updateHandler()` that receives an `id` and responds with the updated ToDo item:
   ```swift
    func updateHandler(id: Int, new: ToDo, completion: (ToDo?, RequestError?) -> Void ) {
        guard let index = todoStore.index(where: { $0.id == id }) else {
            return completion(nil, .notFound)
        }
        var current = todoStore[index]
        current.user = new.user ?? current.user
        current.order = new.order ?? current.order
        current.title = new.title ?? current.title
        current.completed = new.completed ?? current.completed
        execute {
            todoStore[index] = current
        }
        completion(current, nil)
    }
   ```
3.  Run the project and rerun the tests by reloading the test page in the browser.

Twelve tests should now be passing, with the thirteenth failing as follows:
:x: `can delete a todo making a DELETE request to the todo's url`

```
DELETE http://localhost:8080/0
FAILED

404: Not Found (Cannot DELETE /0.)
```

Refresh SwaggerUI and experiment with using the POST route to create ToDo items, then using the PATCH route to update an existing item. For example, if you have a ToDo item at `http://localhost:8080/0` with a title of "mow the lawn", you can change its title by issuing a PATCH with id 0 and this JSON input:

```
{ "title": "wash the dog" }
```

You should see a response code of 200 with a response body of:

```
{
  "id": 0,
  "title": "wash the dog",
  "completed": false,
  "url": "http://localhost:8080/0"
}
```

### 10. Add Support for handling a DELETE request on `/:id`

The failing test is trying to `DELETE` a specific ToDo item. To fix this you need an additional route handler for `DELETE` that this time accepts an ID as a parameter.

1. Register a handler for a `DELETE` request on `/`:
   ```swift
   router.delete("/", handler: deleteOneHandler)
   ```
2. Implement the `deleteOneHandler()` that receives an `id` and removes the specified ToDo item:
   ```swift
    func deleteOneHandler(id: Int, completion: (RequestError?) -> Void ) {
        guard let index = todoStore.index(where: { $0.id == id }) else {
            return completion(.notFound)
        }
        execute {
            todoStore.remove(at: index)
        }
        completion(nil)
    }
   ```
3.  Run the project and rerun the tests by reloading the test page in the browser.

All sixteen tests should now be passing!

### Congratulations, you've built a Kitura backend for the [Todo-Backend](https://www.todobackend.com) project!

## Next Steps

### 1. Try out the ToDo-Backend web client

Now try visiting [https://todobackend.com/client/](https://todobackend.com/client/) in your browser to view the ToDo-Backend web client. Enter an API root of `http://localhost:8080/` and use the website to interact with your REST API. You can add, remove and update ToDo items as you wish.

### 2. Attach a database to your project
Our [ORM tutorial](https://github.com/IBM/ToDoBackend/blob/master/DatabaseWorkshop.md) builds upon the project created in this Workshop and replaces storing ToDos in an Array with a database running locally on your machine, using an ORM (Object Relational Mapper) and PostgreSQL.
