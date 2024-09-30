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

Now, another security issues, is that the password should never be returned in the API request, instead we make it so that we return a *Public View* of the user, so we create a public class inside the `/User.swift` to create the class that will be sent back in the API request, we create extensions to handle converting the user model (with the password) to a public model (without password) by adding extensions to our user model like this

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
this way, you just exchange their credentials for a token they can use, it’s generated by the server upon successful authentication so that they don’t need to provide their credentials again. We simply start by creating the Token model in the `/Models` folder, it contains the `id`, `value`, and the `userID` fields, we do the rest of the work in the `/Migration` folder and `/Configure.swift` file, the usual for any model, now…
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

This way we make sure that only an authenticated user will be able to create a new Acronym and Category. and for even more security, we will add it to the `/UsersController.swift` that way only an authenticated user could create another user, but we will be stuck in case we reset a new database, that why we will **Seed The Database**, and create a user *when* the app first boots up, we do it using a migration

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

> Don’t forget updating the tests after all this work, basically the `/Models+Testable.swift`, `/AcronymTests.swift` and `/UserTest.swift` files, all the changes have been committed to in Github

# Web authentication
After we added the authentication, we did not implement it yet in the web interface, so any user can do anything for now, and unlike the iOS app, it’s not as simple as just sending the token and credentials in the api. So to work around that, browsers use **Cookies**, they’re just a *small amount of data* that the the app sends to the browser *to store locally on the computer* then when the user makes a request, the browser attaches them. They help remember user preferences, track sessions, and manage authentication.
To authenticate users, you need to combine them with **Sessions**, they make it so that you can persist state across requests. This allows the server to recognise and authenticate users as they navigate the site.
In Vapor, they assign each cookie with a *unique ID*, and that is done using a middleware `SessionMiddlware` , so in `/configure.swift` we add the following lines

```swift
app.middleware.add(app.sessions.middleware)
```

now we add these two

```swift
// conform user to ModelSessionAuthenticatable to be able to save and retreive user as part of the session
extension User: ModelSessionAuthenticatable {}
// allows vapor to authenticate users with a username and password when they login, already implemented so nothing to ædd here
extension User: ModelCredentialsAuthenticatable {}
```

Now for the routes, the `/login` route should accept *POST* and one for showing the page, we add a structure for creating a context for the login page

```swift
struct LoginContext: Encodable {
	let title = "Log In"
	let loginError: Bool

	init(loginError: Bool = false) {
		self.loginError = loginError
	}
}
```

this provide the *title* and * error flag*, next we add a route handler

```swift
func loginHandler(_ req: Request) -> EventLoopFuture<View> {
	let context: LoginContext
	if let error = req.query[Bool.self, at: "error"], error {
		context = LoginContext(loginError: true)
	} else {
		context = LoginContext()
	}
	return req.view.render("login", context)
}
```

we define the route, if the request contains the error param (which means there’s an error) than set `loginError` to true, or else just render the context like normal, now we create the `/login.leaf`, not gonna dive into that, just an HTML file with some Bootstrap, however for handling the *POST* request for it, we need to write a new handler

```swift
func loginPostHandler(_ req: Request) -> EvenLoopFuture<Response> {
	if req.auth.has(User.self) {
	return req.eventLoop.future(req.redirect(to: "/"))
	} else {
	let context = LoginContext(loginError: true)
	return req
		.view
		.render("login", context)
		.encodeResponse(for: req)
	}
}
```

This makes sure that the user is authenticated (using the middleware), and redirects him to the home page, if it fails, then it goes back to the login page.
Next we add the routes of course

```swift
routes.get("login", use: loginHandler)
// this middleware checks if there are credentials in the request, and authenticates the request, the method is (i'm sure) built-in
let credentialsAuthRoutes = routes.grouped(User.credentialsAuthenticator())
credentialsAuthRoutes.post("login", use: loginPostHandler)
```

To make the user available for all these pages, we just add a group

```swift
let authSessionsRoutes = routes.grouped(User.sessionAuthenticator())
```

and we create another group that includes redirects for users, they are used for the routes that require protection

```swift
let protectedRoutes = authSessionsRoutes.grouped(User.redirectMiddlware(path: "login"))
```

underneath this one, we register the routes that require protection such as ** creating, editing and deleting acronyms**
If the user tries to connect
### Updating the site
now since the users who can create and delete and all must already be authenticated, we no longer need to include the `userID` in `CreateAcronymData`, we remove it and replace the content of `createAcronymPostHandler(_:data:)` and the `editAcronymPostHandler()`, so in both we add the lines

```swift
let user = try req.auth.require(User.self)
```

to make sure not only the user is *authenticated* but also be able to get his ID, then we replace them in their respective places.
Finally, replace `acronym.$user.id = updateData.userID` with the following:

```swift
acronym.$user.id = userID
```

 next we remove where the user is mentioned (the `<div>` in `/createAcronym.leaf` and the `createAcronymContext` function

> Note that the session right now is kept in memory, so every time you restart the project, you need to login again

### Log Out
that one is really simple, we create the handler, the route and the `.leaf` template in top if `/base.leaf`

```swift
func logoutHandler(_ req: Request) -> Response {
	req.auth.logout(User.self)
	return req.redirect(to: "/")
}

authSessionRoutes.post("logout", use: logoutHandler)
```

we add the button in the `/base.leaf` file, once again, not diving into that, just note that it checks the `userLoggedIn` variable to display or not, we implement this variable next inside the `IndexContext`  class

```swift
let userLoggedIn: Bool
```

and then we pass the result to the **flag** in  `IndexContext`

```swift
let usedLoggedIn = req.auth.has(User.self)
let context = IndexContext(
	title: "Home Page"
	acronyms: acronyms
	userLoggedIn: userLoggedIn)
```

That’s all there is to it

# Cookies
Cookies are an essential part of web development, often used for tasks such as authentication and displaying cookie consent messages. Below is a summary of how to manually handle cookies in a Vapor-based web application.

#### Adding a Cookie Consent Message
To add a cookie consent message to your site:
1. **Edit the Template**: In `base.leaf`, add a conditional block that checks if the `showCookieMessage` flag is set. If true, it displays a footer with a cookie consent message.
	```html
	#if(showCookieMessage):
	<footer id="cookie-footer">
	    <div id="cookieMessage" class="container">
	        <span class="muted">
	            This site uses cookies! To accept this, click
	            <a href="#" onclick="cookiesConfirmed()">OK</a>
	        </span>
	    </div>
	</footer>
	<script src="/scripts/cookies.js"></script>
	#endif
	```

2. **Include Stylesheet**: Before the `<title>` tag in `base.leaf`, add a link to your custom stylesheet.
	```html
	<link rel="stylesheet" href="/styles/style.css">
	```

3. **Create and Style the Footer**: In `Public/styles/style.css`, add styling to pin the cookie message to the bottom of the page.
	```css
	#cookie-footer {
	    position: absolute;
	    bottom: 0;
	    width: 100%;
	    height: 60px;
	    line-height: 60px;
	    background-color: #f5f5f5;
	}
	```

4. **Create JavaScript Function**: In `Public/scripts/cookies.js`, add a function that sets a cookie when the user clicks "OK", hiding the message and storing the consent for a year.
	```javascript
	function cookiesConfirmed() {
	    $('#cookie-footer').hide();
	    var d = new Date();
	    d.setTime(d.getTime() + (365*24*60*60*1000));
	    var expires = "expires=" + d.toUTCString();
	    document.cookie = "cookies-accepted=true;" + expires;
	}
	```

#### Managing the Cookie Consent Message

In `WebsiteController.swift`:

1. **Add a Flag**: Add `let showCookieMessage: Bool` to `IndexContext` to determine if the consent message should be displayed.
2. **Check for Cookie**: In `indexHandler(_:)`, set the `showCookieMessage` flag based on whether the `cookies-accepted` cookie exists.
	```swift
	let showCookieMessage = req.cookies["cookies-accepted"] == nil
	let context = IndexContext(
	    title: "Home page",
	    acronyms: acronyms,
	    userLoggedIn: userLoggedIn,
	    showCookieMessage: showCookieMessage
	)
	```

This process ensures that the cookie consent message only appears when the user hasn't previously accepted the cookies.

# Sessions
we add this for mainly security measures, here is how it works:
what we do is create a **Cross Site Request Forgery (CSRF)** token and send it in the form, here is a better explanation:
-  the site generates a token for each form
- the token is included in the form in a hidden field
- the website checks if the token it generated matches the one being submitted, if they don’t match then the request will be rejected
- we do that by generating a token and storing it in the **User Session**
- we pass it to the template **to be included in the form**

we need to include the token in the acronym creation and then create it using a generator

```swift
struct CreateAcronymContext: Encodable {
	// the other values
	let csrfToken: String
}

// inside createAcronymHandler
let token= [UInt8].random(count: 16).base64
let context = CreateAcronymContext(csrfToken: token)
// we save the token into the request's session data under the `CSRF_TOKEN` key
req.session.data["CSRF_TOKEN"] = token
```

Now after the creation, we hold it in a hidden field, then extract it and compare it when submitted

```html
#if(csrfToken):
	<input type="hidden" name="csrfToken" value="#(csrfToken)">
#endif
```

we compare the token (and reset it ofc, since each one is created in every request)

```swift
// in the CreateAcronymFormData, we should include the

let expectedToken = req.session.data["CSRF_TOKEN"]
req.session.data["CSRF_TOKEN"] = nil
guard
	let csrfToken = data.csrfToken,
	expectedToken == csrfToken
else {
	throw Abort(.badRequest)
}
```

# Validation
Now it’s time to make a register page, and validate the infos the users sent
first off, we create the sign in page

```html
#extend("base"):
	#export("content"):
		<h1>#(title)</h1>
		<form method="post">
			<div class="form-group">
			<label for="name">Name</label>
			<input type="text" name="name" class="form-control"
			id="name"/>
			</div>
		</form>
		<!-- same for the other fields (username, pass, confirm pass) -->
	#endexport
#endextend
```

now we create a new context structure, a *handler* and a *POST* struct context and handler, in order

```swift
// structure for the context
struct RegisterContext: Encodable {
	let title = "Register"
}

//
func registerHandler(_ req: Request) -> EventLoopFuture<View> {
	let context = RegisterContext()
	return req.view.render("register", context)
}

//
struct RegisterData: Content {
	let name: String
	let username: String
	let password: String
	let confirmPassword: String
}

//
func RegisterPostHandler(_ req: Request) -> EventLoopFuture<Response> {
	let data = req.content.decode(RegisterData.self)
	let password = try Bcrypt.hash(data.password)
	let user = User(
		name: data.name,
		username: data.username,
		password: password)
	return user.save(on: req.db).map {
		// authenticate the user, and automatically logs him in
		req.auth.login(user)
		return req.redirect(to: "/")
	}
}
```

we register the routes, obviously

```swift
authSessionsRoutes.get("register", use: registerHandler)
authSessionsRoutes.post("register", use: regsiterPostHandler)
```

now to show the register button in the nav-bar or not depending on the value of `userLoggedIn`

```html
#if(!userLoggedIn):
	<li class="nav-item #if(title == "Register"): active #endif">
		<a href="/register" class="nav-link">Register</a>
	</li>
#endif
```

## Basic Validation
Basically we create an extension that sets some rules and validate it later in the handler

```swift
extension RegisterData: Validitable {
	static func validations(_ validations: inout Validations) {
		validations.add("name", as: String.self, is: .ascii)
		validations.add("username", as: String.self, is: .alphanumeric && .count(3...))
		validations.add("password", as: String.self, is: .count(8...))
	}
}

// validation later in the registerPostHandler
do {
	try RegisterData.validate(content: req)
} catch {
	return req.eventLoop.future(req.redirect(to: "/register"))
}
```

custom validation are too complex for me for now, but here’s what the book does in general:
1. First, they're creating a new type of validation result specifically for zip codes. This result will simply say whether a given string is a valid zip code or not.
2. They're then adding some helpful messages to this result, like "is a valid zip code" for when it passes, and "is not a valid zip code" for when it fails.
3. Next, they're creating the actual validator. This validator uses a regular expression (a special pattern) to check if a string matches the format of a US zip code.
4. The regular expression they're using allows for two formats:
	- Five digits (like 12345)
	- Five digits, followed by a hyphen or space, then four more digits (like 12345-6789 or 12345 6789)
5. They're then setting up the validator to use this regular expression. If a string matches the pattern, it's considered a valid zip code. If not, it's invalid.
6. Finally, they're adding this new zip code validation to a list of validations for some kind of registration data. They're making it optional, meaning it won't cause an error if no zip code is provided, but if one is provided, it must be valid.

## Display the error
now that we made validation, we need to display what’s wrong when the user does something wrong, we just create an alert in the `/register.leaf`

```html
#if(message):
  <div class="alert alert-danger" role="alert">
    Please fix the following errors: <br />
    #(message)
  </div>
#endif
```

add the message to the `RegisterContext` struct

```swift
let message: String?

init(message: String? = nil) {
	self.message = message
}
```

then we include it in the *handler* and *post handler*

```swift
let context: RegisterContext
if let message = req.query[String.self, at: "message"] {
	context = RegisterContext(message: message)
} else {
	context = RegisterContext()
}
```

if there is an error message, then include it in the context for the leaf View, next, when including it in the post handler, we extract the value of the message

```swift
catch let error as ValidationsError {
	let message =
		error.description
		.addingPercentEncoding(
		withAllowedCharacters: .urlQueryAllowed
		) ?? "Unknown error"
	let redirect =
		req.redirect(to: "/register?message=\(message)")
	return req.eventLoop.future(redirect)
}
```

The error message is included in the URL, if the description is not then it provides a default message

# OAuth
## Google OAuth
just for better user experience, since some users don’t like to sign up (me included), so including OAuth in your website is essential, here is how the OAuth works… pretty much

![][image-1]

You click the login with Google, you authorise the app to access the google data that it needs (everything that it will access will be displayed), once you authorise it, google will give the App a *token*, which the app will use to authenticate requests to google APIs, this is pretty much what we’re going to implement

The heavy lifting here will be done using a package called **Imperial** (all the things you see in the graph above, they will be simplified), it not only has Google API integration but also *Facebook* and *GitHub*, and more, to set up Imperial we just include it in the dependencies and then the `/routes.swift`

```swift
.package(url: "https://github.com/vapor-community/Imperial.git", from: "1.2.0")
/*
*/
.product(name: "ImperialGoogle", package: "Imperial")

// in routes.swift
let imperialController = ImperialController()
try app.register(collection: imperialController)
```

We create its controller that is empty for now, now the annoying part, we set up the app with Google
- create a project in [console.developers.google.com/apis/credentials][7]
- create an OAuth credential
- set the email and all that shit
- set the *scopes* of the app… what it will ask of the user. This gives you access to the user’s email and profile which you need to create an account in the TIL app.
- next when setting the URI, add [http://localhost:8080/oauth/google][8]
- save the **client ID** and **client secret**

Now integrating Imperial, we set up the function that handler the Google login, this one runs after the Google login and accord, it will be a simple redirect for now, we will change it later

```swift
@Sendable func processGoogleLogin(request: Request, token: String) throws -> EventLoopFuture<ResponseEncodable> {
	request.eventLoop.future(request.redirect(to: "/"))
}
```

now we set up the route

```swift
guard let googleCallbackURL =
	Environment.get("GOOGLE_CALLBACK_URL") else {
		fatalError("Google callback URL not set")
}
try routes.oAuth(
	from: Google.self,
	authenticate: "login-google",
	callback: googleCallbackURL,
	scope: ["profile", "email"],
	completion: processGoogleLogin)
```

next we provide the `clientID` and `client secret`, using an environment variable, for making this work, you need to set a custom directory in the Xcode settings, and then write all your secret variables inside the `/.env` file, now for the logic and a good UX, you need to create a new user when he logs in with his google account
Now we setup two things, the content that we want from the Google API, and the function that **connects** to the google api

```swift
struct GoogleUserInfo: Content {
	let email: String
	let name: String
}

extension Google {
	static func getUser(on request: Request) throws -> EventLoopFuture<GoogleUserInfo> {
		var headers = HTTPHeaders()
		headers.bearerAuthorization = try BearerAuthorization(token: request.accessToken())
		let googleAPIURL: URI = "https://www.googleapis.com/oauth2/v1/userinfo?alt=json"
		return request
			.client
			.get(googleAPIURL, headers: headers)
			.flatMapThrowing { response in
				guard response.status == .ok else {
					if response.status == .unauthorized {
						throw Abort.redirect(to: "/login-google")
					} else {
						throw Abort(.inernalServerError)
					}
				}
				return try response.content.decode(GoogleUserInfo.self)
			}
	}
}
```

This function will fetch the user infos using the access token, and handles the possible errors during the process, now back to the function that we wrote earlier 

```swift
try Google
	.getUser(on: Request)
	.flatMap { userInfo in
		User
			.query(on: request.db)
			.filter(\.$username == userInfo.email)
			.first()
			.flatMap { foundUser in
				guard let existingUser = foundUser else {
					let user = User(
					name: userInfo.name,
					username: userInfo.email,
					password: UUID().uuidString)
					return user.save(on: request.db).map {
						request.session.authenticate(user)
						return request.rediret(to: "/")
					}
				}
				request.session.authenticate(existingUser)
				return request.eventLoop. future(request.redirect(to: "/")
			}
	}
```

This function is very simple, it executes after the authentication, and the user gave permission, then it does one of two things, match the user in the database, then if it matches, simply authenticate and redirect him in the `request.session.authenticate(existingUser)` line, or create a new user, set the name as the name that you got from the API, the username as the username you got from the API, and set a random password using `UUID()`, the password will be unknown so that we make sure the user does not login using it.

## GitHub OAuth
This one is pretty much the same as the google one, we set the dependency, we import it, then define the  `processGitHubLogin()` routes, and the `struct` and `extension` in the `/ImperialController.swift` and set the **Client ID**  and **Client Secret** inside the `/.env` file. after that we add the `<a>` tag inside the Leaf file and TADA! everything works, the code is almost as similar as the Google one, except the struct which looks like this

```swift
struct GitHubUserInfo: Content {
	let name: String
	let login: String
}
```

and the header that is more specific

```swift
var headers = HTTPHeaders()
try headers.add(
	name: .authorization,
	value: "token \(request.accessToken())")
headers.add(name: .userAgent, value: "vapor")
```

(Don’t forget to check the iOS app too, I updated it but it’s still the same thing as the Google one

## Apple OAuth
When using the other OAuth, it’s just good to also add the apple authentication, since it has its privacy benefits, like hiding the real name or email, the downside is that for setting that up i **need a paid apple developer account**, i’m gonna write the code nonetheless even if it won’t work, here’s how the process works:
- use the `ASAuthorizationAppleIDButton` to show the button, clicking it will make you got through the sign in process 
- it will return a `ASAuthorizationAppleIDCredential` that contains a **JWT** 
- it will send the JWT to the server that should validate it
- if valid, it will either create a new user or login an existing one
- the last step is returning a Token to the iOS app

#### JWT
are a way of transmitting data between parties, they contain *JSON*, so any info can be sent within them, the side who created the JWT sign the token with a secret, it also contains a header, using these two you can verify the integrity of your token, if it’s real and valid, in our example, the server gets the token and the public key to verify the token is valid

So knowing all of this, now we should add the module that handles the JWT in vapor, in the `/Package.swift` we add a new package this time 

```swift
.package(url: "https://github.com/vapor/jwt.git", from: "4.2.2")
// later in the file
.product(name: "JWT", package: "jwt")
```

next we add the *Sign In With Apple identifier* in the User model, and this field will be optional, since we don’t want the user to only Sign In using apple

```swift
@OptionalField(key: "siwaIdentifier")
var siwaIdentifier: String?

// add it to the init later
init(..., siwaIdentifier: String? = nil) {
	/*...*/
	self.siwaIdentifier = siwaIdentifier
}
```

also in the `/CreateUser.swift`

```swift
.field("siwaIdentifier", .string)
```

> WILL LEAVE IT FOR LATER BECAUSE THE PROCESS IS LONG AND I CAN’T EVEN INTEGRATE IT WITHOUT BREAKING MY APP (i don’t have a paid apple dev account)

# Email & Password Reset
Not only we will add emails in the database, we will also add a service to send emails to users, later… for now we will be able to *reset passwords*, so first off we change the *User* structure in the database

```swift
@Field(key: "email")
var email: String
```

And include it in the **initialiser** and not the public one, because it’s always good practice to keep the email hidden, unless required

```swift
init(..., email: String? = nil) {
	/*...*/
	self.email = email
}

// in CreateUser.swift
.field("email", .string, .required)
.unique(on: "email")

// in CreateAdminUser.swift
let user = User(
	name: "Admin",
	username: "admin",
	password: passwordHash,
	email: "admin@localhost.local")
```

After that, we easily fix the following to better match the new user model:

- `/WebsiteController.swift`:
	- `RegisterData` 
	- `RegisterPostHandler(data)`
	- the validation extension to require the email
- `/ImperialController.swift`
	- `processGoogleLogin`
	-  routes in the Github part, as well as creating the `getEmails()` function that is very similar to the `getUser()` one
	- modify the `processGitHubLogin` function, this one is interesting because you chain two function using the `.and(EventLoopFuture<OtherValue>)` method
- fix the tests, not interested 
- fix the iOS app, also not interested
Don’t forget resetting the *Docker* db

```bash
docker rm -f postgres
docker run --name postgres \
	-e POSTGRES_DB=vapor_database \
	-e POSTGRES_USER=vapor_username \
	-e POSTGRES_PASSWORD=vapor_password \
	-p 5432:5432 -d postgres
```

## Sending Emails to the user
Now to be able to reset the password, you need to be able to send an email to the user, so we need to create that process using *SendGrid*

## Profile Picture
This chapter is unique because this time, instead of sending data like JSON or simple text, we’ll learn how to send files in the request, in this case to be able to upload a profile picture

> in this case, we’ll upload the files to the server where the Vapor app resides, for a real app, the files need to be sent to a storage service, one example would be AWS S3 or Heroku (not recommended) 

For starters, we just need to add a field to the user model, this one needs to be optional

```swift
@OptionalField(key: "profilePicture")
var profilePicture: String?
```

We add it to the initialised and all, it will contain the filename of the pfp, note that the GitHub and Google API can be used to retrieve the user pfp, also the pfp upload will be separate from the register. Now we add the field to create user, without adding the unique constraint

```swift
.field("profilePicture", .string)
```

then handle everything in `/WebsiteController.swift`

```swift
func addProfilePictureHandler(_ req: Request) -> EventLoopFuture<View> {
	User.find(req.parameters.get("userID"), on: req.db)
		.unwrap(or: Abort(.notFound)).flatMap { user in
			req.view.render("addProfilePicture", ["title": "Add Profile Picture", "username": user.name])
		}
}
```

Now we add the route in `boot()` and create the leaf file, nothing crazy here, next we modify the `UserContext`, since we now need a new link for the users to be able to access the new form
Now, what we need to do is, when the user visits his page and is authenticated, we need to show a *add profile picture* input, to be able to either upload or update his pfp, we change the `UserContext`

```swift
let authenticatedUser: User?
```

change the context in the `userHandler()`

```swift
// get the authenticated user from the Request's authentication cash
let loggedInUser = req.auth.get(User.self)
// pass it to the context, it's optional
let context = UserContext(..., authenticatedUser: loggedInUser)
```

then in the `/user.leaf` file, we show the **Update/Upload** button if the user is logged in
Now for the image upload part, it’s not that hard since Swift will do the heavy-lifting 

We create a folder that will contain all the profile pictures

```bash
# in the project root
mkdir ProfilePictures
touch ProfilePictures/.keep
```

then create the struct that will contain the image

```swift
struct ImageUploadData: Content {
	var picture: Data
}
```

then define the image folder

```swift
let imageFolder = "ProfilePictures/"
```

and now of course, the post handler, what it will do is, decode the image, then store it using the `NIO` file functionality in the folder we created, then with the same name create a field for the user.

```swift
@Sendable func addProfilePicturePostHandler(_ req: Request) throws ->  EventLoopFuture<Response> {
	// decode the request body (out image) into ImageUploadData
	let data = try req.content.decode(ImageUploadData.self)
	// search for the user
	return User.find(req.parameters.get("userID"), on: req.db)
		.unwrap(or: Abort(.notFound))
		.flatMap { user in
			// get the userID and handle the errors
			let userID: UUID
			do {
				userID = try user.requireID()
			} catch {
				return req.eventLoop.future(error: error)
			}
			// create a name for your file
			let name = "\(userID)-\(UUID()).jpg"
			// setup the file using the root directory + image folder + name we just created
			let path = req.application.directory.workingDirectory + imageFolder + name
			// save the file on the disk
			return req.fileio
				.writeFile(.init(data: data.picture), at: path)
				.flatMap {
					// update the profile picture name in the user field
					user.profilePicture = name
					// save the user then redirect
					let redirect = req.redirect(to: "/users/\(userID)")
					return user.save(on: req.db).transform(to: redirect)
				}
		}
}
```

and now a very weird part, when registering the route, we’re not gonna use `.post` or `.get`, instead we’ll use `.on`

```swift
protectedRoutes.on(.POST, "users", "userID", "addProfilePicture", body: .collect(maxSize: "10mb"), use: addProfilePicturePostHandler)
```

it still is a POST request, but vapor limits the body collection to 16KB, and  this gets rid of this problem and allows a limit of 10MB  instead

This is all there is to **save** the image, now to **serve** it is another story, jk it’s not that hard, just create a handler, its route and modify the `/user.leaf`
The handler will just get the picture using the filename which is already stored in the user field

```swift
func getUsersProfilePictureHandler( _ req: Request) -> EventLoopFuture<Response> {
	User.find(req.parameters.get("userID"), on: req.db)
		.unwrap(or: Abort(.notFound))
		.flatMapThrowing { user in
		guard let filename = user.profilePicture else {
			throw Abort(.notFound)
		}
		let path = req.application.directory
			.workingDirectory + imageFolder + filename
		return req.fileio.streamFile(at: path)
	}
}
```

```swift
authSessionsRoutes.get(
	"users"
	":userID"
	"profilePicture"
	use: getUsersProfilePictureHandler)
```

```swift
#if(user profilePicture):
	<img src="/users/#(user.id)/profilePicture"
	alt="#(user.name) ">
#endif
```

# DB Migration & Versioning
One problem that we’re facing now is that whenever we change a model, we end up deleting the whole database to reset, unless we initialise with null like we did in the previous chapter, right now we don’t have a data, which isn’t a massive issue, but once we have a dataset of users and acronyms it’s going to be an issue, instead of resetting, we need to modify the database using **migrations**, in this chapter, we’re going to:
- make the modifications unique
- add a field in User model for a twitter handle
- the creation of the admin user will only be made in development or testing mode

## Reminder on migrations
When fluent runs, it creates a special table that tracks the migration that have been ran, and runs the migrations in the *orders* you added them in. And the migration only runs once, not to cause conflict, so when changing a migrations, since fluent already ran it, it will not execute agains, so the solution is reseting the whole database, and you don’t want to do that and lost all your data, that’s why the solution is a **migration protocol**

> even tho you can modify using these migrations, it’s a good idea to make a backup of the DB and test everything, it’s still a risky procedure
> When creating the new migration, each one should have a clear name that states its version and is easy to find, ex: YY-MM-DD-FriendlyName.swift.

Now let’s write one, it’s usually written as a `struct` and contains the `prepare` (used for creating a new table or modify an existing one) and `revert` (made to undo the prepare function) methods 

## Updating a Model
When creating a new field, here are the steps:
- add an extension below with the new field in create `CreateModel.swift`
- create a new file in the `/Migrations` folder name it like stated before
- add the new `prepare` and `revert` functions in the struct with the new field
- the `revert` function will have to update and delete the field instead of deleting the whole table
- When adding the migration to the migration list in `/configure.swift`, it must be below the User and Admin user model, since they are they need to be executed first then updated

here is an example of a migration that adds a twitterURL to the user model, without having to reset the database, not that this will be set to null when initialised as it’s not required when creating an account

```swift
// in CreateUser.swift
// note that all the other user values are initialised above this one by the extension name of v20240929 aka the version before this one
enum v20240930 {
	static let twitterURL = FieldKey(stringLiteral: "twitterURL")
}

// add to User.swift
@OptinalField(key: User.v20240930.twitterURL)
var twitterURL: String? // optional string
// initialse the init funtion below too, nothing crazy
```

Now to creating the new migration, we create a new file in the `/Migrations` folder and add the following

```swift
import Fluent

struct AddTwitterToUser: Migration {
	// define the two required funcitons
	func prepare(on database: any Database) -> EventLoopFuture<Void> {
		// notice the schema is from the old version and the twitterURL is from the newer version
		database.schema(User.v20240929.schemaName)
			.field(User.v20240930.twitterURL, .string)
			.update()
	}
	
	func revert(on database: any Database) -> EventLoopFuture<Void> {
		database.schema(User.v20240929.schemaName)
		// this time in revert instead of deleting the table, we will just delete the field
			.deleteField(User.v20240930.twitterURL)
			.update()
	}
}
```

When adding the migration to the `configure.swift` file, we make sure we add it **before** the Admin migration so that it works when creating a new database

```swift
app.migrations.add(AddTwitterURLToUser())
```

Now the user model has a new null `twitterURL` field

## Versioning the API

[1]:	http://localhost:8080
[2]:	http://127.0.0.1:8080
[3]:	http://127.0.0.1:8080
[6]:	http://127.0.0.1:8080/api/users/login
[7]:	console.developers.google.com/apis/credentials
[8]:	http://localhost:8080/oauth/google

[image-1]:	images/sequenceDiagram%202024-09-05%20at%2010.35.38.png