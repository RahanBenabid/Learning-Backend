# Some side notes for my Vapor project

## Questions
- need to understand the token thingy better, why include it in the web app and not in the RapidAPI yet it still works (i think just the /api endpoint makes it work like normal)
- same question with the login
- custom validations

## ToDo
- [ ] please fix the cookies footer
- [ ] fix the register showing up when you’re not in the home folder, and you’re still logged in
- [ ] categories in acronym creation not working
- [ ] retrieve the user profile picture using google and github oauth and store it alongside the user default picture in case he has one
- [x] i still have a lot of issues with the google OAuth, there are  problems in the redirect, that i need to fix later on. check [console.developers.google.com/apis/credentials][1] for that
- [ ] replace the all the extensions values in the models and migrations
- [ ] being able to delete the user pfp, using the NIO of course
## Side Notes
- the whole frontend is pretty much the `/WebsiteController.swift` file mixed with the `/Resources` folder
- there are both the handlers that return a **View** and post handlers that return a **Response**
- URL vs URI, URL is a specific URI that tells you how to access the resource, where URI is a string that identifies a resource, URI identifies a resource by either *Location (URL)* or *name (URN)*, the terms between URL and URI are interchangeable since all URLs are URIs
- *Flaws in the app:* 
	- if the user that signed in using GitHub, somehow has a login the same as the google email what would happen, what also would happen if i already have an the same as the google one and use the google OAuth, in fact, i’ll go check right now… the username cannot contain `@` hehe, i knew it ofc, and i def did not forget the rules I wrote myself



[1]:	console.developers.google.com/apis/credentials