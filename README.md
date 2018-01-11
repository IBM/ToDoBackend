# Building a "To Do" Backend with Kitura

<p align="center">
<img src="https://www.ibm.com/cloud-computing/bluemix/sites/default/files/assets/page/catalog-swift.svg" width="120" alt="Kitura Bird">
</p>

<p align="center">
<a href= "http://swift-at-ibm-slack.mybluemix.net/"> 
    <img src="http://swift-at-ibm-slack.mybluemix.net/badge.svg"  alt="Slack"> 
</a>
</p>

This tutorial teaches how to create a Kitura backend for the [Todo-Backend](https://www.todobackend.com) project, which provides tests and a web client for a "To Do List" application. 

## Pre-Requisites:
**Note:** This workshop has been developed for Swift 4, Xcode 9.x and Kitura 2.x.

1. Install the [Kitura CLI](https://github.com/ibm-swift/kitura-cli):  
   1. Configure the Kitura homebrew tap  
   `brew tap ibm-swift/kitura`  
   2. Install the Kiura CLI from homebrew  
   `brew install kitura`

2. Clone this project from GitHub to your machine (don't use the Download ZIP option):
   ```
   cd ~
   git clone http://github.com/IBM/ToDoBackend
   ```

3. Clone the ToDo Backend tests from GitHub to your machine (don't use the Download ZIP option): 
   ```
   cd ~
   git clone http://github.com/TodoBackend/todo-backend-js-spec
   ```

## Getting started
In order to implement a To Do Backend, a server is required that provides support for storing, retrieving, deleting and updating "to do" items. The To Do Backend project doesn't provide a specification as such for how the server must respond, rather it provides a set of tests which the server must pass. The "todo-backend-js-spec" project provides those tests.

### 1. Run the ToDo Backend Tests:
The following steps allow you to run the tests:
1. Open the tests in a web browser:
   ```
   cd ~/todo-backend-js-spec
   open index.html
   ```
   This should open your browser with the test page open.  
2. Set a "test target root" of http://localhost:8080  
3. Click "run tests"  

The first error reported should be as follows:  
:x: `the api root responds to a GET (i.e. the server is up and accessible, CORS headers are set up)`
```
AssertionError: expected promise to be fulfilled but it was rejected with [Error: 

GET http://localhost:8080
FAILED

The browser failed entirely when make an AJAX request.
```

This shows that the tests made a `GET` request to `http://localhost.com:8080`, but that it failed with no response, which is expected as there is no server.

In the instructions below, reloading the page will allow you to re-run the ToDo Backend tests.


## Building a Kitura Backend
Implementing a compliant ToDo Backend is an incremental task, with the aim at each step to pass more tests. The first step is to create a Kitura server to response on requests.

### 1. Initialize a Kitura Server Project
1. Create a directory for the server project 
   ```
   cd ~/ToDoBackend
   mkdir ToDoServer
   cd ToDoServer
   ```  

2. Create a Kitura starter project  
   ```
   kitura init
   ```  
   The Kitura CLI will now create and build an starter Kitura application for you. This includes adding best-practice implementations of capabilities such as configuration, health checking and monitoring to the application for you.

   More information about the [project structure](http://kitura.io/en/starter/generator/project_layout_reference.html) is available on kitura.io. 

3. Open the ToDoServer project in Xcode  
   ```swift
   cd ~/ToDoBackend/ToDoServer  
   open ToDoServer.xcodeproj
   ```

1. Run the server project in Xcode
    1. Change the selected target from "ToDoServer-Package" to the "TodoServer" executable.
    2. Press the Run button or use the ⌘+R key shortcut.
    3. Select "Allow incoming network connections" if you are prompted.

2. Check that some of the standard Kitura URLs are running:
    * Kitura Monitoring: http://localhost:8080/swiftmetrics-dash/
    * Kitura Health check: http://localhost:8080/health
    * Kitura splash screen: http://localhost:8080/

6. Rerun the tests by reloading the test page in the browser.  

The first test should fail with the following:  
:x: `the api root responds to a GET (i.e. the server is up and accessible, CORS headers are set up)`
```
AssertionError: expected promise to be fulfilled but it was rejected with [Error: 

GET http://localhost:8080/
FAILED

The browser failed entirely when make an AJAX request.
Either there is a network issue in reaching the url, or the
server isn't doing the CORS things it needs to do.
```

### 2. Add Cross Origin Resource Sharing (CORS) Support
This test is still failing, even though the server is responding on localhost:8080. This is because Cross Origin Resource Sharing (CORS) is not enabled.

By default, web servers only serve content to web pages that were served by that web server. In order to allow other web pages, such as the ToDo Backend test page, to connect to the server, [Cross Origin Resource Sharing (CORS)](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) must be enabled.

1. Add the CORS library to the ToDoServer > Package.swift file

   Add the following to the end of the dependencies section of the Package.swift file:
   ```
      .package(url: "https://github.com/IBM-Swift/Kitura-CORS", .upToNextMinor(from: "2.0.0")),
   ```
   and update the dependencies line for the Application target to the following:
   ```
      .target(name: "Application", dependencies: [ "Kitura", "KituraCORS", "Configuration", "CloudEnvironment", "Health" , "SwiftMetrics",
     ]),
   ```
   NOTE:- In order for Xcode to pick up the new dependency, the Xcode project now needs to be regenerated.  
2. Close Xcode, regenerate the Xcode project and reopen:
   ```
   cd ~/ToDoBackend/ToDoServer  
   swift package generate-xcodeproj
   open ToDoServer.xcodeproj
   ```
   
3. Open the Sources > Application > Application.swift file
4. Add an import for the CORS library to the start of the file:
   ```swift
   import KituraCORS
   ```
5. Add the following into the start of the `postInit()` function:  
   ```swift
        let options = Options(allowedOrigin: .all)
        let cors = CORS(options: options)
        router.all("/*", middleware: cors)
   ```

6. Re-run the server project in Xcode  
   1. Edit the scheme and select a Run Executable of “ToDoServer”  
   2. Run the project, then "Allow incoming network connections" if you are prompted.

7. Rerun the tests by reloading the test page in the browser. 

The first test should now be passing but the second test is failing:  
:x: `the api root responds to a POST with the todo which was posted to it`

In order to fix this, we need to implement a `POST` request that saves a todo item.

### 3. Add Support for handling a POST request on '/'
REST APIs typically consist of a HTTP request using a verb such as POST, PUT, GET or DELETE along with a URL and an optional data payload. The server then handles the request and responds with an optional data payload.

A request to store data typically consists of a POST request with the data to be stored, which the server then handles and responds with a copy of the data that has just been stored. This means we need to define a ToDo type, register a  handler for POST requests on "/", and implement the handler to store the data.

1. Define a data type for the ToDo items:
   1. Select the Application folder in the left hand explorer in Xcode
   2. Select File > New > File... from the pull down menu
   3. Select Swift File and click Next
   4. Name the file `Models.swift`, change the "Targets" from "ToDoServerPackageDescription" to "Application", then click Create
   5. Add the following to the created file:
   ```swift
   public struct ToDo : Codable {
       public var id: Int?
       public var user: String?
       public var title: String?
       public var order: Int?
       public var completed: Bool?
       public var url: String?
   }
   ```
   This creates a struct for the ToDo items that uses Swift 4's `Codable` capabilities.

2. Create an in-memory data store for the ToDo items
   1. Open the Sources > Application > Application.swift file
   2. Add a "todoStore",  "nextId" and a "workerQueue" into the App class. On the line below `let cloudEnv = CloudEnv()` add:
   ```swift
   private var todoStore = [ToDo]()
   private var nextId :Int = 0
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
   This will be used to make sure that access to shared resources is serialized so the app does not crash on concurrent requests.

3. Register a handler for a `POST` request on `/` that stores the ToDo item data  
   1. Add the following into the `postInit()` function:
   ```swift
   router.post("/", handler: storeHandler)
   ```
   2. Implement the storeHandler that receives a ToDo, and returns the stored ToDo    
   Add the following as a function in the App class:  
   ```swift
    func storeHandler(todo: ToDo, completion: (ToDo?, RequestError?) -> Void ) {
        var todo = todo
        if todo.completed == nil {
            todo.completed = false
        }
        let todo.id = nextId
        todo.url = "http://localhost:8080/\(todo.id)"
        nextId += 1
        execute {
            todoStore.append(todo)
        }
        completion(todo, nil)
    }
   ``` 
   This expects to receive a ToDo struct from the request, sets `completed` to false if it is nil and adds a `url` value that informs the client how to retrieve this todo item in the future.  
   The handler then returns the updated ToDo item to the client.

4.  Run the project and rerun the tests by reloading the test page in the browser. 

The first three tests should now pass and the fourth fails:  
:X: `after a DELETE the api root responds to a GET with a JSON representation of an empty array`

In order to fix this, a handler for DELETE requests and a subsequent GET handler to return the stored ToDo items.

### 4. Add Support for handling a DELETE request on '/'
A request to delete data typically consists of a DELETE request. If the request is to delete a specific item, a URL encoded identifier is normally provided (eg. '/1' for the item with ID 1). If no identifier is provided, it is a request to delete all of the items.

In order to pass the next test, the ToDoServer needs to handle a DELETE on / resulting in removing all stored ToDo items.

1. Register a handler for a `DELETE` request on `/` that empties the ToDo item data
   1. Add the following into the `postInit()` function:
   ```swift
   router.delete("/", handler: deleteAllHandler)
   ```
   2. Implement the deleteAllHandler empties the todoStore  
   Add the following as a function in the App class:  
   ```swift
    func deleteAllHandler(completion: (RequestError?) -> Void ) {
        execute {
            todoStore = [ToDo]()
        }
        completion(nil)
    }
   ```

### 5. Add Support for handling a GET request on '/'
A request to load all of the stored data typically consists of a GET request with no data, which the server then handles and responds with an array of the data that has just been stored.

1. Register a handler for a `GET` request on `/` that loads the data  
   Add the following into the `postInit()` function:  
   ```swift
	router.get("/", handler: getAllHandler)
   ```
2. Implement the getAllHandler that responds with all of the stored ToDo items as an array.      
   Add the following as a function in the App class:
   ```swift
    func getAllHandler(completion: ([ToDo]?, RequestError?) -> Void ) {
        completion(todoStore, nil)
    }
   ```
3.  Run the project and rerun the tests by reloading the test page in the browser. 

The first seven tests should now pass, with the eighth test failing:  
:x: `each new todo has a url, which returns a todo`

```
GET http://localhost:8080/0
FAILED

404: Not Found (Cannot GET /0.)
```

### 6. Add Support for handling a GET request on '/:id'
The failing test is trying to load a specific ToDo item by making a GET request with the ID of the ToDo item that it wishes to retrieve, which is based on the ID in the `url` field of the ToDo item set when the item was stored by the earlier POST request. In the test above the reqest was for `GET /0` - a request for id 0.

Kitura's Codable Routing is able to automatically convert identifiers used in the GET request to a parameter that is passed to the registered handler. As a result, the handler is registered against the "/" route, with the handler taking an extra parameter.

1. Register a handler for a `GET` request on `/`:
   ```swift
    router.get("/", handler: getOneHandler)
   ```

2. Implement the `getOneHandler` that receives an `id` and responds with a ToDo item:
   ```swift        
    func getOneHandler(id: Int, completion: (ToDo?, RequestError?) -> Void ) {
        completion(todoStore.first(where: {$0.id == id }), nil)
    }
   ```
3.  Run the project and rerun the tests by reloading the test page in the browser. 

The first nine tests now pass. The tenth fails with the following:  
:x: `can change the todo's title by PATCHing to the todo's url`  

```
PATCH http://localhost:8080/0
FAILED

404: Not Found (Cannot PATCH /0.)
```

### 7. Add Support for handling a PATCH request on '/:id'
The failing test is trying to `PATCH` a specific ToDo item. A `PATCH` request updates an existing item by updating any fields sent as part of the PATCH request. This means that a field by field update needs to be done.

1.  Register a handler for a `PATCH` request on `/`:
   ```swift
   router.patch("/", handler: updateHandler)
   ```
2. Implement the `updateHandler` that receives an `id` and responds with the updated ToDo item:
   ```swift
    func updateHandler(id: Int, new: ToDo, completion: (ToDo?, RequestError?) -> Void ) {
        guard let idMatch = todoStore.first(where: { $0.id == id }), let idPosition = todoStore.index(of: idMatch) else { return }
        var current = todoStore[idPosition]
        current.user = new.user ?? current.user
        current.order = new.order ?? current.order
        current.title = new.title ?? current.title
        current.completed = new.completed ?? current.completed
        execute {
            todoStore[idPosition] = current
        }
        completion(todoStore[idPosition], nil)
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

### 8. Add Support for handling a DELETE request on '/:id'
The failing test is trying to `DELETE` a specific ToDo item. This means registering an additonal handler for `DELETE` that this time accepts an ID as a parameter.

1. Register a handler for a `DELETE` request on /:
   ```swift
   router.delete("/", handler: deleteOneHandler)
   ```
2. Implement the `deleteOneHandler` that receives an `id` and removes the specified ToDo item:
   ```swift
    func deleteOneHandler(id: Int, completion: (RequestError?) -> Void ) {
        guard let idMatch = todoStore.first(where: { $0.id == id }), let idPosition = todoStore.index(of: idMatch) else { return }
        todoStore.remove(at: idPosition)
        completion(nil)
    }
   ```
3.  Run the project and rerun the tests by reloading the test page in the browser. 

All sixteen tests should now be passing!

### Congratulations, you've built a Kitura backend for the [Todo-Backend](https://www.todobackend.com) project!

## Next Step

### An iOS application for the ToDo Backend
This tutorial has helped you build a ToDo Backend for the web tests and web client from the [Todo-Backend](https://www.todobackend.com) project, but one of the great values of Swift is end to end development between iOS and the server. Clone the [iOSSampleKituraKit](https://github.com/IBM-Swift/iOSSampleKituraKit) repository and open the `iOSKituraKitSample.xcworkspace` to see a iOS app client for the ToDo-Backend project.

   ```
   cd ~
   git clone https://github.com/IBM-Swift/iOSSampleKituraKit.git
   cd iOSSampleKituraKit/KituraiOS
   open iOSKituraKitSample.xcworkspace/
   ```

Run (⌘+R) the iOS application. You should be able to use the app to add, change and delete ToDo items!
