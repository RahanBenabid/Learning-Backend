# Vapor
server side framework, apparently can also do frontend, you can create a project simply like this:
Ensure Vapor is installed

```sh
brew install vapor
vapor -h
```

Create you first project

```sh
mkdir vapor && cd vapor
vapor new HelloVapor
```

then simply open your project with Xcode, it will download the dependencies

## Swift Package Manager
similar to CocoaPods in IOS, notice how you can run the Xcode project without the Xcode project template, because when using SPM it will create a workspace in a hidden folder, the dependencies are declared in the `Package.swift` file with the target, and how they link together

setting up endpoints is mostly the same as for node, here is a code that takes a parameter from the url, and handles errors then, returns a string 

```swift
app.get("hello", ":name") { req -> String in 
  guard let name = req.parameters.get("name") else {
	throw Abort(.internalServerError)
  }
  return "Hello \(name)!"
}
```

now the endpoint `localhost:8080/hello/rahan` will return `Hello, rahan!`, get request also handle almost the same, but instead we use the built in `Codable` protocol in swift, here is a small example of a POST request

```swift
struct InfoData: Content {
  let name: String
  let age: Int
  let single: Bool
}
struct InfoResponse: Content {
  let request: String
}

app.post("info") { req -> InfoResponse in 
  let data = try req.content.decode(InfoData.self)
  return InfoResponse(request: data)
```

## TroubleShooting
there are some ways to do that in vapor here are the recommended ones
- updating the packages using `swift package update`, or doing it in the menu bar in Xcode
- cleaning and re-building the project in Xcode and the `.build`, `.swiftpm`, `Package.resolved` and `DerivedData` folders

# HTTP
HyperText Transfer Protocol, is the core of the web, it is sent to the browser whenever you visit a web page (when you load a page, it sends a `GET` request for that page), here are the structure of an HTTP request:
- the request line, or the method, you know them, not all tho
- the host, is needed when multiple servers are hosted at the same address
- Other request Headers, like the authorisation, Accept, Cache Control… they are nothing more than key-value pairs, they make the server side more *robust*
- Optional request Data, if they are required by the HTTP method
### The HTTP Responses
They consist of:
- the status line, or code
  - `1XX` are not frequent, they mean information response
  - `2XX` are success responses
  - `3XX` are redirection responses
  - `4XX` are client error responses
  - `5XX` mean server side error, either a bug or resource exhaustion
- Response header, they are similar to the request headers seen earlier, with the same type of content
- Optional response body, could contain something like an *image file, JSON*, and even *response codes*, `204` means no content
### REST
Representational State Transfer, the core of many web applications, lets you define such endpoints:
- `GET /api/acronyms/` get all acronyms
- `POST /api/acronyms` create a new acronym
- `GET /api/acronyms/1` get the acronym with the ID 1
- `PUT /api/acronyms/1` update the acronym with the ID 1
- `DELETE /api/acronyms/1` delete the acronym with the ID 1

# Async
This concept is important because it lets us run multiple threads at a time, instead of trying to fetch something and it puts the whole website on pause, in an asynchronous server, these requests that take time will be put to side until the fetching is complete. 
To put it aside until it resolves, we must put wrap it in a **promise**, here is some Vapor code of a synchronous vs async operation:

```swift
// sync operation
func getAllUsers() -> [User] {
  // the database query
}

// async operation
func getAllUsers() -> EventLoopFuture<[User]> {
  // the database query
}
```

What `EventLoopFuture` means you will return *something* in the future, and at that point there might be nothing

### Futures
take this code

```swift
// fetch all users from the db
return database.getAllUsers().flatMap { users in
  // updates the user
  let user = users[0]
  user.name = "Bob"
  // saves the update to the database
  return user.save(on: req.db).map { user in
	// returns the appropriate HTTPStatus
	return .noContent
  }
}
```

it uses `flatMap(_:)` since the top level promise returns an `EventLoopFuture` while the inner promise uses `map(_:)` since it returns a non-future `HTTPSatus`.

### Transform
used when you only care that the future completed and not the result

```swift
return database.getAllUsers().flatMap { users in
  let user = users[0] 
  user.name = "Bob" 
  return user 
	.save(on: req.db) 
	.transform(to: HTTPStatus.noContent) 
}
```

This makes the code easier to read

### Flatten
when you wait for a number of futures to finish, like when saving multiple models in a database

```swift
static func save(_ users: [User], request: Request) -> EventLoopFuture<HTTPStatus> {
    // 1
    var userSaveResults: [EventLoopFuture<User>] = []
    // 2
    for user in users {
        userSaveResults.append(user.save(on: request.db))
    }
    // 3
    return userSaveResults
      .flatten(on: request.eventLoop)
      .map { savedUsers in
        // 4
        for users in savedUser {
            print("Saved \(user.username)")
        }
        // 5
        return .created
    }
}
```

- you can use `and(:_)` to chain some number of futures that do not rely on each other

### Create a future
You might end up in a weird situation, for example an `if` statement that returns a future while the else returns a non-future, to fix this you can write code that converts non-future into `EvenLoopFuture` using `request.eventLoop.future(:_)`

```swift
// define a method that creates a TrackingSession from the request
func createTrackingSession(for request: Request) -> EventLoopFuture<TrackingSession> {
    return request.makeNewSession()
}
// defines a method that gets a tracking session from the request
func getTrackingSession(for request: Request) -> EventLoopFuture<TrackingSession> {
    // attempt to create a tracking session using the request's key
    let session: TrackingSession? = TrackingSession(id: request.getKey())
    // make sure the session was created successfully
    guard let createdSession = session else {
        return createTrackingSession(for: request)
    }
    // this returns the future on the request's EventLoop
    return request.eventLoop.future(createdSession)
}
```

since `createTrackingSession(for:)` returns a `EventLoopFuture<TrackingSession>` you have turn `createdSession` into a future to make the compiler happy

There are many other things to handle that you can check later, here’s the list, they’re all available in the book
- dealing with errors
- dealing with future errors
- chaining futures
- the `.always` method (always executes after the future)
- the `.wait()` method, self explanatory

# Fluent
This is Vapor’s ORM, it is set between the app and the database, the positives are that you don’t have to interact with the DB directly, so that you don’t have to write queries as strings that are less safe and more of a pain in the ass, to describe a model throughout vapor we use **Models**, and leverages wrappers to provide database integration. (such as `@ID` or `@Field@` (should only be used in non-optional props))
these will also be used in relationships.

```swift
import Fluent
import Vapor

final class Todo: Model {
    static let schema = "todos"
    
    @ID(key: .id)
    var id: UUID?
    
    @Field(key: "title")
    var title: String
    
    @Field(key: "isComplete")
    var isComplete: Bool
    
    init() { }
    
    init(id: UUID? = nil, title: String, isComplete: Bool = false) {
        self.id = id
        self.title = title
        self.isComplete = isComplete
    }
}

extension Acronym: Content {}
```

### Saving a model in a DB
can be done using **migration**, to make flexible changes to your db, they are also used for making table descriptions. The migration protocol has two required methods `prepare` and `revert`, it handles database schema creation and deletion.

```swift
import Fluent

struct CreateTodo: Migration {
    func prepare(on database: Database) -> EventLoopFuture<Void> {
        database.schema("todos")
            .id()
            .field("title", .string, .required)
            .field("isComplete", .bool, .required)
            .create()
    }

    func revert(on database: Database) -> EventLoopFuture<Void> {
        database.schema("todos").delete()
    }
}
```

So how do we actually save a model?
It is done using the wrapper `Content`, and creating the endpoint obviously, and what makes it easy is that `Content` conforms to `Codable`, so the user can just send a JSON and we can decode it easily into `Content`, an example POST request:

```swift
 app.post("api", "acronyms") { req -> EventLoopFuture<Acronym> in
    // using Codable, we can decode the request JSON data into an 			Acronym
    let acronym = try req.content.decode(Acronym.self)
    // save the model
    return acronym.save(on: req.db).map {
        // returns the acronym when the save is complete and here we 			use .map
        acronym
    }
}
```

# Databases
The database used for the project is specified in the `package.swift` and the database configuration happens in `Sources/App/configure.swift` 
- first import the driver
- configure it depending on your db, for example an in-memory SQLite, or setting up the host password, user name and all

```swift
app.databases.use(.mysql(
    hostname: Environment.get("DATABASE_HOST") ?? "localhost",
    username: Environment.get("DATABASE_USERNAME")
      ?? "vapor_username",
    password: Environment.get("DATABASE_PASSWORD")
      ?? "vapor_password",
    database: Environment.get("DATABASE_NAME")
      ?? "vapor_database",
    tlsConfiguration: .forClient(certificateVerification: .none)
  ), as: .mysql)
```

i finally made this shitty code work, MySQL uses a **TLS connection** by default, so when you run the app on docker it generates a self-signed certificate, however in the case of a production app, you must provide the certificate to trust

# CRUD
The basic operations you need in every backend, setting up the endpoints isn’t really complex once you already set one already, here is a simple `GET` endpoint that communicates with the database:

```swift
app.get("api", "acronyms") { req -> EventFutureLoop<[Acronym]> in
  // query to the db
  Acronym.query(on: req.db).all()
}
```

in my Xcode project there are the rest of the operations with all the comments you need.

### A little advanced query
Here is some Fluent query that is even more powerful, this one uses the `OR` statement

```swift
app.get("api", "acronyms", "search") {
  req -> EventLoopFuture<[Acronym]> in
  // retrieve the search term from the URL query string
  guard let searchTerm = req.query[String.self, at: "term"] else {
   throw Abort(.badRequest)
  }
  // find all the acronyms whos short prop matches
  return Acronym.query(on: req.db).group(.or) { or in
   or.filter(\.$short == searchTerm)
   or.filter(\.$long == searchTerm)
  }.all()
}
```

The use of `\.$short` here is an example of the property wrapper projection. `$short` is the projected value of the `@Field` property wrapper for the short property of the Acronym model.

# Controllers
The more routes you add to `routes.swift` the more it’s going to be cluttered especially in big projects, so we use controllers to manage our routes and models more efficiently, used (respectfully) controllers and RESTful controllers.

What controllers do is handle interactions from a client, it’s good practice to have them, and it can go even further, you can use controllers to control different (older) versions of your API, so it could be used to keep your code separated. inside a controller, here is what it looks like:

```swift
// we define the controller
struct AcronymsController: RouteCollection {
  func boot(routes: RoutesBuilder) throws {
	// route defintions
	let acronymdRoutes = routes.grouped("api", "acronyms")

	// define the handler functions and mark them as @Sendable for safety
	@Sendable func createHandler(_ req: Request) throws ->
	  EvenLoopFuture<Acronym> {
	  //implementation
	}
	/*
	  more handlers
	*/

	// register the route
	acronymsRoutes.get(use: getAllHandler)
	/*
	  routes to the corresponding handler
	*/
  }
}
```

and to use all of what you wrote you need to correctly setup `routes.swift` like this:

```swift
let acronymsController = AcronymsController()
try app.register(collection: acronymsController)
```
 
This way your code is more organised, and your app will stay rideable even if your app gets bigger since everything is separated

# Parent-Child Relationships
for example we create a new `User` type after the acronym one, like the usual steps
- create the **User Model** in `/Models`
- create the migration file in `/Migration` (a file that defines the changes to be made to the database schema, create, delete, update the db)
- add to the migration list
- configure the controller (add the endpoints and routes)
Now let’s set the relationships between two models, we add a new property:

```swift
@Parent(key: "userId")
var: user: User
```

it adds a User property to the model and it creates a link between the two models, it **is not** optional, and in the initialiser, we do as such

```swift
init( /* other props */ , userID: User.IDValue) {
  /*
  other props
  */
  self.$user.id = userID
}
```

and go to the migration, add the following:

```swift
.field("userID", .uuid, .required)
```

This adds the new column for user using the key provided to the @Parent property
wrapper. The column type, uuid, matches the ID column type from CreateUser.

## Domain-Transfer-Object
used to simplify the request, and basically represents what a client should send or receive, good to use when you’re dealing with complex data structure, it’s just a simple way of sending and receiving data between the server and client, it’s just a property that holds data. it packages (compresses multiple pieces of data together so that we can only transfer what’s important.
change the controller and add the DTO 

```swift
struct CreateAcronymData: Content {
	let short: String
	let long: String
	let userID: UUID
}
```

and change the model create handler for the database *(don’t forget the update one too if you implemented it)*

```swift
@Sendable func createHandler(_ req: Request) throws -> EventLoopFuture<Acronym> {
		let data = try req.content.decode(CreateAcronymData.self)
		
		let acronym = Acronym(
			short: data.short,
			long: data.long,
			userID: data.userID)
		return acronym.save(on: req.db).map { acronym }
}
```

using this we can add a new handler that can fetch the user ID

### Childen
can also be fetched, the difference isn’t very big, just check the project

## Foreign Keys
are an important concept, they link the two table together, and they ensure
- you can’t create the child with a parent that doesn’t exist
- you can’t delete a User if you didn’t delete his acronyms
- you can’t the User table before the Acronyms table
to set them up, just go to the migrations, and add something such as the following example:

```swift
.field("userID", .uuid, .required, .references("users", "id"))
```

this adds a reference  from the `userID` column to the `id` column in the Users Table
> Don’t forget creating the Users Table before the Acronyms one

# Sibling Relationships
also known as many to many relationships, in this example, we’ll add a category to our acronyms, so an acronyms can be part of many categories and a category can have many acronyms, the steps are like usual:
- create the model
- create the migration
- add the migration to configure
- set the controller
- add the route to `/routes.swift`
now, what differentiate this relationships with the previous one, is in its query, let’s say we have an array of acronyms inside a category, and we want to search for all the categories of an acronym, in that case, we should inspect every single category. So in fluent we’ll use something called a **pivot**, a separate model to hold the relationship between them, what it does is *contains the relationship*. we create a whole Model and migration for it.

```swift
// we define the two properties that link the 2 models together
@Parent(key: "acronymID")
var acronym: Acronym

@Parent(key: "categoryID")
var category: Category
```

and the migration:

```swift
.field("acronymID", .uuid, .required, .references("acronyms", "id", onDelete: .cascade))
.field("categoryID", .uuid, .required, .references("categories", "id", onDelete: .cascade))
```

note that it is good practice to use **foreign key** constraints with sibling relationships too. if we don’t we can for example, delete acronyms and categories that are still linked by the pivot and the
relationship will remain, without flagging an error.
now we add to the migration list 

```swift
app.migrations.add(CreateAcronymCategoryPivot())
```

and now… most importantly, to create the actual relationship between the two models, we’ll use the pivot, let’s take one of the siblings:

```swift
@Siblings(
	through: AcronymCategoryPivot.self,
	from: \.$acronym,
	to: \.$category)
var categories: [Category]
```

this will take the following parameters:
- the pivot’s model type
- the key path from pivot which references the root (original) model
- the key path from pivot which references the related model
> You don’t need to initialise an instance, `@Sibling` has to join between the two different models, but Fluent abstracts this part

so here is what the `Handler` will look like:

```swift
@Sendable func addCategoriesHandler(_ req: Request) -> EventLoopFuture<HTTPStatus> {
	let acronymQuery = Acronym.find(req.parameters.get("acronymID"), on: req.db)
		.unwrap(or: Abort(.notFound))
	let categoryQuery = Category.find(req.parameters.get("categoryID"), on: req.db)
		.unwrap(or: Abort(.notFound))
	return acronymQuery.and(categoryQuery)
		.flatMap { acronym, category in
			acronym
				.$categories
				.attach(category, on: req.db)
				.transform(to: .created)
		}
}
```

don’t forget to add the route lol

What we did till now was successfully create the relationship, but you can’t view them, so let’s query them:

```swift
@Sendable func getCategoriesHandler(_ req: Request) -> EventLoopFuture<[Category]> {
	Acronym.find(req.parameters.get("acronymID"), on: req.db)
		.unwrap(or: Abort(.notFound))
		.flatMap { acronym in
			// Use the new property wrapper to get the categories. Then use a Fluent query to return all the categories.
			acronym.$categories.query(on: req.db).all()
		}
}
```

and a **Very Important** part of course, is doing the same thing to the other sibling:
- adding the `@Sibling` to the Model
- create the handler for the database request
- add the route

## Deleting the RelationShip
is pretty much the same thing, we add a handler that gets both the `IDs` but uses `detach(_:on)` instead

# Testing
I’m dreading this part so much, i literally hate TDD, however i must admit that it is important, apparently testing is as old as development, what testing does is give you **confidence** that your application works as intended, to be able to **modify it** without running into issues, what you do is write tests before you write code, in vapor, you define the test target in `Package.swift`, first we create the Test File inside the directory specified in the `Package.swift` inside `/Tests` in the root

```swift
@testable import App
import XCTVapor

final class UserTests: XCTestCase {	 
}
```

this will be used to test the Users, we import the necessary modules, now we write the real tests, like getting the users from the API, here is a very lengthy test function

```swift
func testUserCanBeRetrievedFromAPI() throws {
	let expectedName = "Alice"
	let expectedUsername = "alice"
	
	let app = Application(.testing)
	
	defer { app.shutdown() }
	
	try configure(app)
	
	let user = User(
		name: expectedName,
		username: expectedUsername)
	try user.save(on: app.db).wait()
	try User(name: "Luke", username: "lukes")
		.save(on: app.db)
		.wait()
	
	try app.test(.GET, "/api/users", afterResponse: { response in
		
		XCTAssertEqual(response.status, .ok)
		
		let users = try response.content.decode([User].self)
		
		XCTAssertEqual(users.count, 2)
		XCTAssertEqual(users[0].name, expectedName)
		XCTAssertEqual(users[0].username, expectedUsername)
		XCTAssertEqual(users[0].id, user.id)
	})	
}  
```

these are a bunch of basic tests that will create 2 users and see if they get saved, now we need to set up the `configure.swift` to use a different environment, one in running the app and one in testing

```swift
if (app.environment == .testing) {
	databaseName = "vapor-test"
	databasePort = 5433
} else {
	databaseName = "vapor-database"
	databasePort = 5432
}
```

and we will use these variables inside the `app.database.use(...)`, and now a part just as important is setting up a different docker container for it

```bash
docker run --name postgres-test \
	-e POSTGRES_DB=vapor-test \
	-e POSTGRES_USER=vapor_username \
	-e POSTGRES_PASSWORD=vapor_password \
	-p 5433:5432 -d postgres
```

however, in this case the test will only run once, since on the second run, the app will have 4 users, so we’ll need to reset the database

```swift
// reset the database on every run then run the migration again
try app.autoRevert().wait()
try app.autoMigrate().wait()
```

now logically we should test all the endpoints, so instead of writing that MASSIVE chunk of code each time, we create a reusable piece of code, see the Xcode project, we built `Application+Testable.swift` and `Models+Testable.swift` to be able to reuse the code later, than created a test for each endpoint. 
In `Models+Testable.swift` we create all the models we’re going to use for testing, for example when retrieving an acronym from the user, we should create a new Model inside it like such

```swift
extension Acronym {
	static func create(
		short: String = "TIL",
		long: String = "Today I Learned",
		user: User? = nil,
		on database: Database
	) throws -> Acronym {
		var acronymUser = user
		
		if acronymUser == nil {
			acronymUser = try User.create(on: database)
		}
		
		let acronym = Acronym(short: short, long: long, userID: acronymUser!.id!)
		try acronym.save(on: database).wait()
		return acronym
	}
}
```

The Xcode project goes in detail about testing all our models.
> To learn more about testing in linux, which does not interest me yet, check the book

# Building An iOS App
what we’re going to do is have two separate apps, Our previous app we built all this time then the iOS App we will talk to, What the iOS app has for now is just a skeleton that interacts with the *TIL* API.

To create an iOS app that performs the crud operations that we built:
- We create a model structure that matches ours, the fact that we built the backend in the same language helps immensely, this was (just like in Vapor) made inside the `/Models` folder
- create the network service that handles the API using the `Foundation` module 
- create a way to fetch all instances of a particular resource type (we implemented the `getAll()` function inside the `ResourceRequest.swift`), it gets all the values from the API, here is the function:

```swift
// this function gets the values of the resource typr from the API
func getAll(
	completion: @escaping
	(Result<[ResourceType], ResourceRequestError>) -> Void ) {
		// create a variable with the resource URL
		let dataTask = URLSession.shared
			.dataTask(with: resourceURL) { data, _, _ in
				// decodes the response
				guard let jsonData = data else {
					completion(.failure(.noData))
					return
				}
				do {
					let ressource = try JSONDecoder()
						.decode([ResourceType].self, from: jsonData)
					completion(.success(ressource))
				} catch {
					completion(.failure(.decodingError))
				}
			}
```

> this creates a ressource Request, we will be able to provide it with the type, and API path
- we create a table and the request

```swift
var acronyms: [Acronym] = []
let acronymsRequest = ResourceRequest<Acronym>(resourcePath: "acronym")
```

- then on every refresh of the `acronymsRequest` will call the `.getAll` method.
- the next step is simply displaying the table in our UI

```swift
let acronym = acronyms[indexPath.row]
cell.textLabel?.text = acronym.short
cell.detailTextLabel?.text = acronym.long
```

when trying to fetch something else like an array of users, we create another `ViewController`.

let’s implement creating a user in the iOS app:

- Set Up Your iOS Project
- Create a new iOS project in Xcode.
- Configure App Transport Security
- Modify the Info.plist file to allow HTTP requests by adding the appropriate settings.
- Create a Network Manager
- Create a NetworkManager class to handle network requests to your Vapor API.
- Set Up Your ViewController
- Modify the main ViewController to utilise the NetworkManager for fetching data when the view loads.
- Update the User Interface
- Add a `UILabel` in the storyboard to display the fetched data and create an outlet for it in the ViewController.
- Handle JSON Data (if applicable)
- If your API returns JSON, define a `Codable` model to represent the data.
- Update the NetworkManager to decode the JSON response into your model.

## Implement a Delete Operation
Here is a step by step on how to implement a Delete operation in the App:
First we need to make sure the `AcronymRequest.swift` handles the delete request, like the other request we may have implemented

```swift
func delete() {
	// create urlRequest and set the Method
	var urlRequest = URLRequest(url: resource)
	urlRequest.httpMethod = "DELETE"
	// Create a data task for the request using the shared URLSession and send the request
	let dataTask = URLSession.shared.dataTask(with: urlRequest)
	dataTask.resume()
}
```

Next we enable the deletion of the table row

```swift
override func tableView(
	_ tableView: UITableView,
	commit editingStyle: UITableViewCell.EditingStyle,
	forRowAt indexPath: IndexPath
) {
	if let id = acronyms[indexPath.row].id {
		// in case of valid ID, create an AcronymRequest, and call the delete()
		let acronymDetailRequester = AcronymRequest(acronymID: id)
		acronymDetailRequester.delete()
	}
	
	// remove the acronym from the local array of acronyms
	acronyms.remove(at: indexPath.row)
	// remove the acronym row from the table view
	tableView.deleteRows(at: [indexPath], with: .automatic)
}
```

Ta-Da, done as simple as this, just one interface config and one API call

## Creating a Category
We first need to implement the `save(_:)`:

```swift
@IBAction func save(_ sender: Any) {
	// Check if the name text field has a value and is not empty
	guard
		let name = nameTextField.text, // Get the text from the nameTextField
		!name.isEmpty // Ensure the text is not empty
	else {
		// If the name is empty, show an error message
		ErrorPresenter.showError(
			message: "You must specify a name", // Error message
			on: self) // Present the error on the current view controller
		return // Exit the function if the name is invalid
	}
	
	// Create a new Category object with the specified name
	let category = Category(name: name)
	
	// Create a ResourceRequest for the "categories" endpoint and save the new category
	ResourceRequest<Category>(resourcePath: "categories")
		.save(category) { [weak self] result in // Save the category and handle the result
			switch result {
			case .failure:
				// If saving fails, show an error message
				let message = "There was a problem saving the category"
				ErrorPresenter.showError(message: message, on: self) // Present the error
			case .success:
				// If saving is successful
				DispatchQueue.main.async { [weak self] in
					// Navigate back to the previous view controller
					self?.navigationController?
						.popViewController(animated: true) // Animate the transition back
				}
			}
		}
}
```

note this:
1. `DispatchQueue.main.async `: This line ensures that the following code runs on the main thread. UI updates (like navigating between view controllers) should always be done on the main thread to maintain a responsive user interface.
2. `self?.navigationController?` : This accesses the navigation controller associated with the current view controller. A navigation controller is responsible for managing a stack of view controllers, allowing you to navigate back and forth between them.
3. `popViewController(animated: true)`: This method is called on the navigation controller to remove the current view controller from the navigation stack and go back to the previous one. The animated: true parameter means that the transition will include a visual animation, making the navigation feel smoother.
In summary, the .success case indicates that the category was saved successfully. The code then navigates back to the previous screen in the app, allowing the user to see the updated list of categories or return to the previous context.

Now we must add the ability to add an acronym to the category:

```swift
func loadData() {
	// Create a ResourceRequest for the "categories" endpoint
	let categoriesRequest = ResourceRequest<Category>(resourcePath: "categories")
	
	// Perform a request to get all categories
	categoriesRequest.getAll { [weak self] result in
		// Handle the result of the request
		switch result {
		case .failure:
			// If the request fails, show an error message
			let message = "There was an error getting the categories"
			ErrorPresenter.showError(message: message, on: self) // Present the error on the current view controller
			
		case .success(let categories):
			// If the request is successful, store the retrieved categories
			self?.categories = categories
			
			// Update the UI on the main thread
			DispatchQueue.main.async { [weak self] in
				// Reload the table view to display the new categories
				self?.tableView.reloadData()
			}
		}
	}
}
```

now we handle the POST request:

```swift
func add(
  category: Category,
  completion: @escaping (Result<Void, CategoryAddError>) -> Void) {
  // Ensure the category has a valid ID
  guard let categoryID = category.id else {
    completion(.failure(.noID)) // Return an error if no ID is found
    return
  }
  
  // Construct the URL for the POST request
  let url = resource
    .appendingPathComponent("categories") // Add "categories" to the path
    .appendingPathComponent("\(categoryID)") // Add the category ID to the path
  
  // Create a URLRequest for the specified URL
  var urlRequest = URLRequest(url: url)
  urlRequest.httpMethod = "POST" // Set the HTTP method to POST
  
  // Create a data task to send the request
  let dataTask = URLSession.shared
    .dataTask(with: urlRequest) { _, response, _ in
      // Ensure the response is valid and has a status code of 201 (Created)
      guard
        let httpResponse = response as? HTTPURLResponse,
        httpResponse.statusCode == 201
      else {
        completion(.failure(.invalidResponse)) // Return an error for invalid response
        return
      }
      
      // If the request is successful, call the completion handler with success
      completion(.success(()))
    }
  
  // Start the data task
  dataTask.resume()
}
```

 and now the final part:  

```swift
extension AddToCategoryTableViewController {
  // Override the method that is called when a row in the table view is selected
  override func tableView(
    _ tableView: UITableView,
    didSelectRowAt indexPath: IndexPath
  ) {
    // 1: Get the selected category from the categories array using the selected row index
    let category = categories[indexPath.row]
    
    // 2: Check if the acronym has a valid ID
    guard let acronymID = acronym.id else {
      // If the acronym has no ID, show an error message
      let message = """
        There was an error adding the acronym
        to the category - the acronym has no ID
      """
      ErrorPresenter.showError(message: message, on: self) // Present the error on the current view controller
      return // Exit the function if the acronym ID is invalid
    }
    
    // 3: Create an AcronymRequest object to manage the request for adding the acronym to the category
    let acronymRequest = AcronymRequest(acronymID: acronymID)
    
    // Call the add method on the acronymRequest to add the selected category
    acronymRequest.add(category: category) { [weak self] result in
      // Handle the result of the add operation
      switch result {
        // 4: If the addition is successful
      case .success:
        DispatchQueue.main.async { [weak self] in
          // Navigate back to the previous view controller
          self?.navigationController?
            .popViewController(animated: true)
        }
        
        // 5: If there was a failure in adding the acronym to the category
      case .failure:
        // Show an error message indicating the failure
        let message = """
          There was an error adding the acronym
          to the category
        """
        ErrorPresenter.showError(message: message, on: self) // Present the error on the current view controller
      }
    }
  }
}
```

This method allows the user to select a category from a table view and attempts to add an acronym to that category. It includes error handling for cases where the acronym ID is missing or the addition fails, ensuring a smooth user experience.
Now, we just add this line of code to `makeAddToCategoryController(_:)` to return a `AddToCategoryTableViewController` created with the current acronym and its categories

```swift
AddToCategoryTableViewController(
	coder: coder,
	acronym: acronym,
	selectedCategories: categories)
```

# Leaf
Vapor’s templating language, it allows you to pass infos to a page to generate HTML, also to avoid **duplication**, instead of creating multiple, you just create a template and set the properties that should be displayed, so when you change your code, you only change it in one place, it also allows you to embed templates into other templates.
You can either create a new project and select using leaf, or just add it to your dependencies `Package.swift` file.
The templates should be inside the `/Resources/Views` folder inside your project, and **ONLY THE VIEWS YOU DUMBASS**
First things first, some routes should be created, let’s create `WebsiteController.swift` inside the `Controllers` folder like the other controllers, the routes handling is very similar to the `/Controllers` routes, the `boot()` function and `@Sendable` also, here’s what the function looks like:

```swift
@Sendablec func indexHandler(_ req: Request) -> EventLoopFuture<View> {
  return req.view.render("index")
}
```

This renders the index template and return the result. This will try to render `index.leaf`, which is like a classic `.html` file:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8"/>
        <title>Hello World</title>
    </head>
    <body>
        <h1>Hello World</h1>
    </body>
</html>
```

and now register the route, like all the other from the backend, in `routes.swift`

```swift
let websiteController = WebsiteController()
try app.register(collection: websiteController)
```

and in the `configure.swift`, write 

```swift
app.views.use(.leaf)
```

To tell vapor to use Leaf when rendering pages, and configure the custom working directory, now just open the [localhost][1] link, But this just shows a normal static page, what if we want to display real data inside it? We can write something like this

```html
<title>#(title) | Acronyms</title>
```

and then set the title variable inside the `WebsiteController.swift`

create the class:

```swift
struct IndexContext: Encodable {
  let title: String
}
```

set it then pass it:

```swift
@Sendable func indexHandler(_ req: Request) -> EventLoopFuture<View> {
  // create an Encodable IndexTitle containing the title we want
  let context = IndexContext(title: "Home Page")
  // pass it as a second parameter
  return req.view.render("index", context)
}
```

let’s go even further and query the database to display a list of Acronyms

```swift
struct IndexContext: Encodable {
	let title: String
	let acronyms: [Acronym]?
}

@Sendable func indexHandler(_ req: Request) -\> EventLoopFuture\<View\> {
	Acronym.query(on: req.db).all().flatMap { acronyms in
	let acronymsData = acronyms.isEmpty ? nil : acronyms
	let context = IndexContext(title: "Home Page", acronymsData)
	return req.view.render("index", context)
	}
}
```

This fetches the list of Acronyms from the database and passes them to the View, inside the view we’re gonna write a bit of HTML with some syntax:

```html
#if(acronyms):
	<table>
		<thead>
			<tr>
				<th>Short</th>
				<th>Long</th>
			</tr>
		</thead>
		<tbody>
			#for(acronym in acronyms):
				<tr>
					<td>#(acronym.short)</td>
					<td>#(acronym.long)</td>
				</tr>
			#endfor
		</tbody>
	</table>
#else:
		<h2>There aren't any acronyms yet!</h2>
#endif
```

Look up the Xcode project for even more fancy stuff

> Notice that when testing the endpoint in RapidAPI, you can send request to [http://127.0.0.1:8080 ][2] and [http://127.0.0.1:8080/api][3] separately, one returns HTML, the other return json data respectively

# Embedding Templates
Is needed so that you only make changes in one place, the changes might be styling for example, so that you don’t have to change your style in every file, let’s start by creating `base.leaf` in the `/Views` folder:

```html
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="utf-8"/>
		<title>#(title) | Acronyms</title>
	</head>
	<body>
		#import("content")
	</body>
</html>
```

what this does is:  Every file you make now will have this already without writing it all over again, you just need to add:

```html
#extend("base"):
	#export("content"):
	<!-- add all the additions specific to your file -->
	#endexport
#endextend
```

we extend our file and replace our `"content"` variable, This takes the HTML specific to index.leaf and wraps it in an #export tag. When Leaf renders base.leaf as required by index.leaf, it takes content and inserts it into the base template. let’s take our `acronym.leaf`, here’s what it looks like now, just a couple of lines

```html
#extend("base"):
	#export("content"):
		<h1>#(acronym.short)</h1>
		<h2>#(acronym.long)</h2>
		<p>Created by #(user.name)</p>
	#endexport
#endextend
```

Go check the project for the file configuration using **Bootstrap**.

### Serving Content
What we’ve been doing till now is serve data such as *JSON*, but what about images or style sheets, first off we add the middleware that is able to serve files:

```swift
app.middleware.use(
	FileMiddleware(publicDirectory: app.directory.publicDirectory)
)
```

This serves files in the `/Public` directory.

### Sharing templates
in the project, we ended up creating the acronym table twice, in the Front page and the user page, so when we need to modify it, we’ll end up doing it twice, instead we should create a separate file called `acronymsTable.leaf`, write the whole table inside it and go to where the table was written, and simply inset

```swift
#extend("acronymsTable")
```

# Creating/Modifying models
The goal now is to create an acronym IN the interface, for that we should implement *two routes*: 
- handle a **GET** request to display the form the user will fill
- handle the **POST** request after the user submits their acronym
For the GET part, we need to display the list of users, so that the user can select which one created the acronym, pretty dumb but we’ll so it like this for now, just create a handler that sends a list of users
Now for the POST request… the code handles the creation of a new acronym by decoding the request data, creating an Acronym instance, saving it to the database, and redirecting the user to the newly created acronym's page:

```swift
@Sendable func createAcronymPostHandler(_ req: Request) throws -> EventLoopFuture<Response> {
    // Decode the request content into a CreateAcronymData struct
    let data = try req.content.decode(CreateAcronymData.self)
    
    // Create a new Acronym instance with the data from the request
    let acronym = Acronym(
        short: data.short,
        long: data.long,
        userID: data.userID
    )
    
    // Save the new acronym to the database
    return acronym.save(on: req.db).flatMapThrowing {
        // Ensure the ID is set after saving, or else throw a server error
        guard let id = acronym.id else {
            throw Abort(.internalServerError)
        }
        
        // Redirect the user to the newly created acronym's page
        return req.redirect(to: "/acronyms/\(id)")
    }
}
```

We’ll do the same for editing the acronyms of course, now the real issue is with the **DELETE**, browsers have no simple way of sending DELETE requests, so what we’re gonna do is send a POST request a delete a route, we’ll registers a route at [/acronyms/\<ACRONYM ID\>/delete]() to accept POST requests and call `deleteAcronymHandler(\_:)`

```swift
func deleteAcronymHandler(_ req: Request) -> EventLoopFuture<Response> {
	Acronym
	  .find(req.parameters.get("acronymID"), on: req.db)
	  .unwrap(or: Abort(.notFound)).flatMap { acronym in
		acronym.delete(on: req.db)
		.transform(to: req.redirect(to: "/"))
	}
}

routes.post("acronyms", ":acronymID", "delete", use: deleteAcronymHandler)
```

### Add Acronyms to Categories
The issue here is that in web applications, they need to accept all the infos *in one request*, so knowing this, we create an **extension** to `Category.swift` that:
- checks for the category that we submitted
- creates the relationship if the category exists
- if the category doesn’t exist, then create the category AND the relationship

# Password
To give users some privacy (big corps should learn from me hehe). We just create a new field and initialise it, but it’ll be more complex than just creating a normal field

```swift
@Field(key: "password")
var password: String

/*
...
*/

init(id: UUID? = nil, name: String, username: String, password: String) {
	self.name = name
	self.username = username
    self.password = password
}
```

now we **shouldn’t** store the user password in plain text. So in the user creation controller, we just hash our password using `Bcrypt`

```swift
user.password = try Bcrypt.hash(user.password)
```

As simple as it gets, of course, after changes like these in the database, we must reset it, and thankfully, since it’s contained in docker, we just reset it like

```bash
docker stop postgres

docker rm postgres

docker run --name postgres -e POSTGRES_DB=vapor_database \
	-e POSTGRES_USER=vapor_username \
	-e POSTGRES_PASSWORD=vapor_password \
	-p 5432:5432 -d postgres
```

Now, another security issues, is that the password should never be returned in the API request, instead we make it so that we return a *Public View* of the user, so we create a public class inside the `User.swift` to create the class that will be sent back in the API request, we create extensions to handle converting the user model (with the password) to a public model (without password) by adding extensions to our user model like this

```swift
// represents the object that will be sent in the GET API
extension User {
	func convertToPublic() -> User.Public {
		return User.Public(id: id, name: name, username: username)
	}
}

// this will help to reduce nesting, it will call convertToPublic() on EventLoopFuture<User>, [User] and EventLoopFuture<[User]>, they allow you to change your route to handle returning a public user

extension EventLoopFuture where Value: User {
	func convertToPublic() -> EventLoopFuture<User.Public> {
		return self.map { user in
			return user.convertToPublic()
		}
	}
}

extension Collection where Element: User {
	func convertToPublic() -> [User.Public] {
		return self.map { $0.convertToPublic() }
	}
}

extension EventLoopFuture where Value == Array<User> {
	func convertToPublic() -> EventLoopFuture<[User.Public]> {
		return self.map { $0.convertToPublic() }
	}
}
```
 
then in all the handlers, we will return a `User.Public` and use the `. convertToPublic()` method

```swift
// now this function will use the extension EventLoopFuture<[User]> for the return type
@Sendable func getAllHandler(_ req: Request) -> EventLoopFuture<[User.Public]> {
	User.query(on: req.db).all().convertToPublic()
}
```

> do not forget to change the return type wherever the user is mentioned, in the Acronyms for example

# Authentification
We need to generate the token for that, and using the HTTP standardised method, we **combine** the username and password, then **Base64-encode** the result.

```bash
# combine
rahan:password

# B64-encode
cmFoYW46cGFzc3dvcmQ=
```

the header will be

```bash
Authorization: Basic dGltYzpwYXNzd29yZA==
```

Vapor, already has built-in HTTP Auth, first we add method will be using to conform the User to the HTTP Auth protocol, we will tell Vapor which fields to use when making authentication 

```swift
// The ModelAuthenticable will allow Fluent models to use the HTTP Auth
extension User: ModelAuthenticatable {
	
	// tells fluents the username and password path
	static let usernameKey = \User.$username
	static let passwordHashKey = \User.$password
	
	// very the hash here (you hash the input password and compare the result with the db hash)
	func verify(password: String) throws -> Bool {
		try Bcrypt.verify(password, created: self.password)
	}
}
```

next we set up a **middleware** to ensure that only authenticated users can access certain routes. It groups routes together under this protection. Finally, it defines a POST route that will only be accessible if the user is authenticated.

```swift
// creates an instance of ModelAuthenticator and GuardAuthenticationMiddlware to use basic HTTP auth and checks if the credentials are valid, and then ensures that the request contains an authenticated user
let basicAuthMiddleware = User.authenticator()
let guardAuthMiddleware = User.guardMiddleware()
// create a group
let protected = acronymsRoutes.grouped(
	basicAuthMiddleware,
	guardAuthMiddleware)
// connects the create acronym path to createHandler(), through this middleware
protected.post(use: createHandler)


// we delete
acronymsRoutes.post(use: createHandler)
```

> Middleware allows you to intercept requests and responses in your application. In this example, basicAuthMiddleware intercepts the request and authenticates the user supplied.

Now, a POST request to [http://localhost:8080/api/acronyms]() will respond with a **401 unauthorised**, unless you add an Auth **username and password**, and now, only authenticated users can create an acronym. And now for better quality of life, we should let the users 

# Login
this way, you just exchange their credentials for a token they can use, it’s generated by the server upon successful authentication so that they don’t need to provide their credentials again. We simply start by creating the Token model in the `/Models` folder, it contains the `id`, `value`, and the `userID` fields, we do the rest of the work in the `/Migration` folder and `Configure.swift` file, the usual for any model, now…
We add an **extension** to the model that will generate that token

```swift
extension Token {
	static func generate(for user: User) throws -> Token {
		let random = [UInt8].random(count: 16).base64
		return try Token(value: random, userID: user.requireID())
	}
}
```

This is nothing crazy, just a token generator, now the loginHandler

```swift
func loginHandler(
_ req: Request) throws
-> EventLoopFuture<Token> {
    let user = try req.auth.require(User.self)
    let token = try Token.generate(for: user)
    return token.save(on: req.db).map { token }
}
```

this handles user login and token generation.

```swift
let basicAuthMiddleware = User.authenticator()
let basicAuthGroup = usersRoute.grouped(basicAuthMiddleware)
basicAuthGroup.post("login", use: loginHandler)
```

This block configures a route for user login, applying middleware to handle authentication.
Now sending a POST request to [http://127.0.0.1:8080/api/users/login][6] with the Basic Auth, will return back a token, example:

```json
{
	"id":"3F332D00-F76A-4E59-B878-61974B790083",
	"user":{
		"id":"F55FFEA3-6077-4B96-88C9-063F54C82C5C"
	},
	"value":"bWfypO22XAZl52dANGCvGw=="
}
```

now since we require Authentication, we no longer need to provide the `userID` when creating an Acronym, so now instead, we remove the expected ID from the **DTO**, and in the handler, we get the authenticated user, and create a new acronym with the data fetched from that user (replace the Update one too)

```swift
let user = try req.auth.require(User.self)
let acronym = try Acronym(
	short: data.short,
	long: data.long,
	userID: user.requireID())
```

now, sending a request as the following will result in a successful acronym creation

```HTTP
POST /api/acronyms HTTP/1.1
Authorization: Bearer pYIyt4j0ao17Ut8lYSDvdw==
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Host: 127.0.0.1:8080
Connection: close
User-Agent: RapidAPI/4.1.5 (Macintosh; OS X/15.0.0) GCDHTTPRequest
Content-Length: 28

short=LOL&long=Lord+Of+Laugh
```

to make the app even more complete, and we add the token handling thingy to every controller (User, Acronym and Category)

```swift
let tokenAuthMiddleware = Token.authenticator()
let guardAuthMiddleware = User.guardMiddleware()
let tokenAuthGroup = usersRoute.grouped(
	tokenAuthMiddleware,
	guardAuthMiddleware)
	tokenAuthGroup.post(use: createHandler)
```

This way we make sure that only an authenticated user will be able to create a new Acronym and Category. and for even more security, we will add it to the `UsersController.swift` that way only an authenticated user could create another user, but we will be stuck in case we reset a new database, that why we will **Seed The Database**, and create a user when the app first boots up, we do it using a migration

```swift
struct CreateAdminUser: Migration {
	func prepare(on database: any Database) -> EventLoopFuture<Void> {
		let passwordHash: String
		do {
			passwordHash = try Bcrypt.hash("password")
		} catch {
			return database.eventLoop.future(error: error)
		}
		
		let user = User(
			name: "Admin",
			username: "admin",
			password: passwordHash)
		return user.save(on: database)
	}
	
	func revert(on database: any Database) -> EventLoopFuture<Void> {
		User.query(on: database)
			.filter(\.$username == "admin")
			.delete()
	}
}
```


[1]:	http://localhost:8080
[2]:	http://127.0.0.1:8080
[3]:	http://127.0.0.1:8080
[6]:	http://127.0.0.1:8080/api/users/login