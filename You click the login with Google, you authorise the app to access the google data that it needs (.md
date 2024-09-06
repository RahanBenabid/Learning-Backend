![][image-1]

You click the login with Google, you authorise the app to access the google data that it needs (everything that it will access will be displayed), once you authorise it, google will give the App a *token*, which the app will use to authenticate requests to google APIs, this is pretty much what we’re going to implement

The heavy lifting here will be done using a package called **Imperial**, it not only has Google API integration but also *Facebook* and *GitHub*, and more, to set up Imperial we just include it in the dependencies and then the `/routes.swift`

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
- create a project in [console.developers.google.com/apis/credentials][1]
- create an OAuth credential
- set the email and all that shit
- set the *scopes* of the app… what it will ask of the user. This gives you access to the user’s email and profile which you need to create an account in the TIL app.
- next when setting the URI, add [http://localhost:8080/oauth/google][2]
- save the **client ID** and **client secret**
Now integrating Imperial

[1]:	console.developers.google.com/apis/credentials
[2]:	http://localhost:8080/oauth/google

[image-1]:	images/sequenceDiagram%202024-09-05%20at%2010.35.38.png