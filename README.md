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
used to simplify the request, and basically represents what a client should send or receive, good to use when you’re dealing with complex data structure
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