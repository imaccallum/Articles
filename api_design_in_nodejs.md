# API Design in Node.js

## API Architecture
### What is an API

### Components of an API
0. Service - basically wrappers around any 3rd party library (data stores, mailing service, push notification services, etc…). 
1. Repo —  abstractions around a single resource type. Uses various services to perform operations on resources.
2. Controller — sits between the repo (and optionally the service layer) and the router. Extracts information from requests and uses the repo/service layer to formulate responses.
3. Router — The router is the Interface in API. It specifies the routes that your application client should hit.
4. Application — listens on a port for requests then forwards them to the router.

### What are we trying to avoid?
Something like this:

```javascript
app.get('/users/:id', function (req, res) {
    
    User.findById(req.params.id, function (err, user) {
        if (err) return res.status(500).send(":(");
        res.status(200).send(user);
    });
});
```


The responsibilities of our app, router, controller, and repo are all intertwined. We have no way to test a particular segment of this logic without doing an end-to-end test.As we add functionality to our application, we may want to reuse the functionality that fetches the user. So what do we do? Copy and paste this same block of code, perhaps? Not very DRY. Now say we only want to retrieve certain properties from the user (ie. we don’t want to return the user’s password when we fetch the user in this context). We have to find everywhere this functionality exists in our code and update it. There is not a unified place where this functionality exists. Unless absolutely necessary, we should avoid interacting with third party libraries directly. Instead, we should decide how we intend to use them and create an interface by which to interact with them. The underlying business logic for a use case that requires that we look up a user and send an email to that user because they forgot their password should be independent of both the mail service and the database. This is where the repo layer comes into play.
	
### Architecture Goals
Above all, our 
There are many different ways you can structure your API, but after much trial and error this seems to be a decent way of doing it
A basic API consists of 5 layers
1. Repo — abstractions built on top of the libraries we will use

Heres a quick overview of what we will be doing. First, we’ll setup the Repo Layer so we can provide an interface by which we interact with our data stores (Redis and Sequelize). Then we’ll create the Controller Layer to bridge the gap between routes and our repo. Finally, we’ll setup the Router layer for our application to use and bring it all together.

## Overview

We’re going to make an application that does two very simple things: create and fetch users.

Before we get started, let’s define a few interfaces for a User object.

```typescript
// Really basic user object
export interface User {
	id: string,
	name: string
}

// Used for constructing a new user
export interface UserBuilder {
	name: string
}
```




## Part 1: the Repo Layer
The repo layer is collection of classes that you use for interacting with your underlying data store(s). You should not interact directly with a database in a controller for several reason. First, code is not reusable. When you change code in one controller function, such as the properties to fetch, it is not updated across all the other times you access that entity. 

Our use cases are to create and fetch users so we’ll define an interface for our User Repo:

```typescript
export interface UserRepo {
	createUser(builder: UserBuilder): Promise<User>
	getUser(id: String): Promise<User>
}
```

Next, we’ll want to make a concrete implementation of this to utilize a Sequelize data store. We’ll inject a Sequelize db instance and implement the functions.

```typescript
export class UserRepoSequelize implements UserRepo {
	
	constructor(private db) {
		// Probably define an interface for a sequelize db
	}

	public async getUser(id: User): Promise<User> {
		const user = await this.db.User.findOne({
			where: { id }
		})

		return user
	}

	public async createUser(builder: UserBuilder): Promise<User> {
		const user = await this.db.User.create(builder)
		return user
	}
}
```

We could test this and it should work. This is an extremely easy piece of code to test because all it requires is a db instance which we could mock if we wanted to or just use a test database. Furthermore, if we wanted to switch to MongoDB/Mongoose instead of PostgreSQL/Sequelize, we just create a UserRepoMongoose implementation.

Now we could move on to the controller right now and that would work alright, but let’s say we are getting tons of requests so we wish to implement a caching layer using Redis to alleviate the traffic to our PostgreSQL database. Using the decorator pattern, this is trivial. We make another implementation of UserRepo that wraps another instance of UserRepo.

For the `getUser` function, we want to first check if the user with the id is in the cache. If the user is cached in redis, we will return it. If not, we will retrieve it from our wrapped UserRepo instance (in this case Sequelize), cache it, and then return it.

For the `createUser` function, the user does not exist in our database yet so there is no way it could be cached already. So we’ll create it and then cache it.

```typescript
export class UserRepoCacheRedis implements UserRepo {
	
	constructor(
		private inner: UserRepo, 
		private redisClient: RedisClient) {
		// Let's assume RedisClient has getAsync and setAsync
	}

	public async getUser(id: User): Promise<User> {
		
		// Attempt to retrieve user for the cache
		const cachedUser = await this.redisClient.getAsync(`user:${id}`)

		if (cachedUser) {
			// The user was already cached, return it
			return cachedUser
		} else {
			// Fetch from db, then cache, then return it
			const user = await this.inner.getUser(id)
			await this.redisClient.setAsync(`user:${user.id}`, user)
			return user
		}
	}

	public async createUser(builder: UserBuilder): Promise<User> {
		// Just forward the request because there is 
		// nothing to cache
		const user = await this.inner.createUser(builder)
		
		if (user) {
			// Cache the user if 
			await this.redisClient.setAsync(`user:${user.id}`, user)
		}
		
		return user
	}
}
```

Using the UserRepo looks something like this:
```typescript
// Create our db repo and wrap it with our cache
const userRepoDb = new UserRepoSequelize(db)
const userRepo = new UserRepoCacheRedis(userRepoDb, redisClient)

const newUser: UserBuilder = {
	name: 'Abc'
}

// Create, cache, and return user
const createdUser = await userRepo.createUser(newUser) 

// Fetch user, should fetch user from cache and never touch db
const user = await this.userRepo.getUser(createdUser.id)
```

This should provide us with a simple way of accessing our database. We could create other decorators for logging, validation, sanitization, etc… and it would not take much effort on our end.



## Part 1: the Controller Layer
```typescript
// controllers/user/user.controller.ts
export interface UserController {
	getUser(req, res, next)
	createUser(req, res, next)
}

//controllers/user/user.controller.default.ts
import UserController from './user.controller'
import { Request, Response, NextFunction } from "express";

export class UserControllerDefault implements UserController {
	
	constructor(private repo: UserRepo) {}

	public async getUser(
		req: Request, 
		res: Response, 
		next: NextFunction) {

		try {
			const { userId } = req.params
			if (!userId) throw new Error('userId not defined')
			const user = await this.repo.getUser(id)
			res.json(user)		
		} catch (err) {
			next(err)
		}
	}

	public createUser(
		req: Request, 
		res: Response, 
		next: NextFunction) {

		try {
			const userBuilder = req.body
			if (! userBuilder) throw new Error('no body')
			const user = await this.repocreateUser(userBody)
			res.json(user)
		} catch (err) {
			next(err)
		}
	}
}

```






```javascript

import express from 'express'

// We'll get to this later
const userController = new UserController(userRepo)

// Create our router
const router = express.Router()

// Add some routes
router.get('/users/:userId', userController.getUser)
router.post('/users', userController.createUser)


// Create our application
const app = express()

// Use our router
app.use(router)

module.exports = app
```

```javascript
// server.js

import app from './app'

const port = process.env.PORT

const server = app.listen(port, () => {
	console.log(`App is running on port ${port}.`)
});

export = server;

```

```javascript
// app.js
import express from 'express'
	
// Setup our sequelize and redis clients
const db = //
const redisClient = //
	
// Create UserRepo instance for the controller
const userRepoDb = new UserRepoSequelize(db)
const userRepo = new UserRepoCacheRedis(userRepoDb, redisClient)

// Create a UserController instance from repo
const userController = new UserController(userRepo)

// Create our router
const router = express.Router()

// Add some routes
router.get('/users/:userId', userController.getUser)
router.post('/users', userController.createUser)

// Create our application
const app = express()
	
// Use our router
app.use(router)

// Use an error handler her

module.exports = app
```







Extra Layer

2. Service Layer (Optional) — combines multiple repos to define advanced business logic (ie. uses the UserRepo to fetch a user and a MailRepo to send them an email). The more business logic we can keep out of our controllers, the better. 
