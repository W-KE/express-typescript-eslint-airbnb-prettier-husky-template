# express-typescript-eslint-airbnb-prettier-husky-template

Template Express project

# The folder structure

```
src
│   app.js          # App entry point
└───api             # Express route controllers for all the endpoints of the app
└───config          # Environment variables and configuration related stuff
└───jobs            # Jobs definitions for agenda.js
└───loaders         # Split the startup process into modules
└───models          # Database models
└───services        # All the business logic is here
└───subscribers     # Event handlers for async task
└───types           # Type declaration files (d.ts) for Typescript
```

# ☠️ Don’t put your business logic inside the controllers!! ☠️

You may be tempted to just use the express.js controllers to store the business logic of your application, but this
quickly becomes spaghetti code, as soon as you need to write unit tests, you will end up dealing with complex mocks for
req or res express.js objects.

It’s complicated to distingue when a response should be sent, and when to continue processing in `background`, let’s say
after the response is sent to the client.

Here is an example of what not to do.

```typescript
route.post('/', async (req, res, next) => {

  // This should be a middleware or should be handled by a library like Joi.
  const userDTO = req.body;
  const isUserValid = validators.user(userDTO)
  if (!isUserValid) {
    return res.status(400).end();
  }

  // Lot of business logic here...
  const userRecord = await UserModel.create(userDTO);
  delete userRecord.password;
  delete userRecord.salt;
  const companyRecord = await CompanyModel.create(userRecord);
  const companyDashboard = await CompanyDashboard.create(userRecord, companyRecord);

  // ...whatever...

  // And here is the 'optimization' that mess up everything.
  // The response is sent to client...
  res.json({ user: userRecord, company: companyRecord });

  // But code execution continues :(
  const salaryRecord = await SalaryModel.create(userRecord, companyRecord);
  eventTracker.track('user_signup', userRecord, companyRecord, salaryRecord);
  intercom.createUser(userRecord);
  gaAnalytics.event('user_signup', userRecord);
  await EmailService.startSignupSequence(userRecord)
});
```

# Use a service layer for your business logic 💼

This layer is where your business logic should live.

It’s just a collection of classes with clear purposes, following the **SOLID** principles applied to node.js.

In this layer there should not exist any form of **SQL query**, use the data access layer for that.

* Move your code away from the express.js router
* Don’t pass the req or res object to the service layer
* Don’t return anything related to the HTTP transport layer like a status code or headers from the service layer.

```typescript
route.post('/',
           validators.userSignup, // this middleware take care of validation
           async (req, res, next) => {
             // The actual responsability of the route layer.
             const userDTO = req.body;

             // Call to service layer.
             // Abstraction on how to access the data layer and the business logic.
             const { user, company } = await UserService.Signup(userDTO);

             // Return a response to client.
             return res.json({ user, company });
           });
```

Here is how your service will be working behind the scenes.

```typescript
import UserModel from '../models/user';
import CompanyModel from '../models/company';

export default class UserService {

  async Signup(user) {
    const userRecord = await UserModel.create(user);
    const companyRecord = await CompanyModel.create(userRecord); // needs userRecord to have the database id 
    const salaryRecord = await SalaryModel.create(userRecord, companyRecord); // depends on user and company to be created

    // ...whatever...

    await EmailService.startSignupSequence(userRecord)

    // ...do more stuff...

    return { user: userRecord, company: companyRecord };
  }
}
```

# Dependency Injection 💉

D.I. or inversion of control (IoC) is a common pattern that will help the organization of your code, by ‘injecting’ or
passing through the constructor the dependencies of your class or function.

By doing this way you will gain the flexibility to inject a ‘compatible dependency’ when, for example, you write the
unit tests for the service, or when the service is used in another context.

**Code with no D.I**

```typescript
import UserModel from '../models/user';
import CompanyModel from '../models/company';
import SalaryModel from '../models/salary';

class UserService {
  constructor() {
  }

  Sigup() {
    // Calling UserModel, CompanyModel, etc
  }
}
```

**Code with manual dependency injection**

```typescript
export default class UserService {
  constructor(userModel, companyModel, salaryModel) {
    this.userModel = userModel;
    this.companyModel = companyModel;
    this.salaryModel = salaryModel;
  }

  getMyUser(userId) {
    // models available throug 'this'
    const user = this.userModel.findById(userId);
    return user;
  }
}
```

Now you can inject custom dependencies.

```typescript
import UserService from '../services/user';
import UserModel from '../models/user';
import CompanyModel from '../models/company';

const salaryModelMock = {
  calculateNetSalary() {
    return 42;
  }
}
const userServiceInstance = new UserService(userModel, companyModel, salaryModelMock);
const user = await userServiceInstance.getMyUser('12346');
```

The amount of dependencies a service can have is infinite, and refactor every instantiation of it when you add a new one
is a boring and error-prone task.

That’s why dependency injection frameworks were created.

The idea is you declare your dependencies in the class, and when you need an instance of that class, you just call the
‘Service Locator’.

Let’s see an example using `typedi` an npm library that brings D.I to node.js

```typescript
// services/user.ts

import { Service } from 'typedi';

@Service()
export default class UserService {
  constructor(
    private userModel,
    private companyModel,
    private salaryModel
  ) {
  }

  getMyUser(userId) {
    const user = this.userModel.findById(userId);
    return user;
  }
}
```

Now `typedi` will take care of resolving any dependency the UserService require.

```typescript
import { Container } from 'typedi';
import UserService from '../services/user';

const userServiceInstance = Container.get(UserService);
const user = await userServiceInstance.getMyUser('12346');
```

# Using Dependency Injection with Express.js in Node.js

Using D.I. in express.js is the final piece of the puzzle for this node.js project architecture.

**Routing layer**

```typescript
route.post('/',
           async (req, res, next) => {
             const userDTO = req.body;

             const userServiceInstance = Container.get(UserService) // Service locator

             const { user, company } = userServiceInstance.Signup(userDTO);

             return res.json({ user, company });
           });
```

# A unit test example 🕵🏻

By using dependency injection and these organization patterns, unit testing becomes really simple.

You don’t have to mock req/res objects or require(…) calls.

Example: Unit test for signup user method

```typescript
// tests/unit/services/user.ts

import UserService from '../../../src/services/user';

describe('User service unit tests', () => {
  describe('Signup', () => {
    test('Should create user record and emit user_signup event', async () => {
      const eventEmitterService = {
        emit: jest.fn(),
      };

      const userModel = {
        create: (user) => {
          return {
            ...user,
            _id: 'mock-user-id'
          }
        },
      };

      const companyModel = {
        create: (user) => {
          return {
            owner: user._id,
            companyTaxId: '12345',
          }
        },
      };

      const userInput = {
        fullname: 'User Unit Test',
        email: 'test@example.com',
      };

      const userService = new UserService(userModel, companyModel, eventEmitterService);
      const userRecord = await userService.SignUp(teamId.toHexString(), userInput);

      expect(userRecord).toBeDefined();
      expect(userRecord._id).toBeDefined();
      expect(eventEmitterService.emit).toBeCalled();
    });
  })
})
```

# Configurations and secrets 🤫

Following the battle-tested concepts of [Twelve-Factor](https://12factor.net/) App for node.js the best approach to
store API Keys and database string connections, it’s by using dotenv.

Put a `.env` file, that must never be committed (but it has to exist with default values in your repository) then, the
npm package `dotenv` loads the .env file and insert the vars into the `process.env` object of node.js.

That could be enough but, I like to add an extra step. Have a `config/index.ts` file where the `dotenv` npm package
loads the `.env` file and then I use an object to store the variables, so we have a structure and code autocompletion.

```typescript
// config/index.ts

const dotenv = require('dotenv');
// config() will read your .env file, parse the contents, assign it to process.env.
dotenv.config();

export default {
  port: process.env.PORT,
  databaseURL: process.env.DATABASE_URI,
  paypal: {
    publicKey: process.env.PAYPAL_PUBLIC_KEY,
    secretKey: process.env.PAYPAL_SECRET_KEY,
  },
  mailchimp: {
    apiKey: process.env.MAILCHIMP_API_KEY,
    sender: process.env.MAILCHIMP_SENDER,
  }
}
```

This way you avoid flooding your code with `process.env.MY_RANDOM_VAR` instructions, and by having the autocompletion
you don’t have to know how to name the env var.

# Loaders 🏗️

The idea is that you split the startup process of your node.js service into testable modules.

Let’s see a classic express.js app initialization

```typescript
const mongoose = require('mongoose');
const express = require('express');
const bodyParser = require('body-parser');
const session = require('express-session');
const cors = require('cors');
const errorhandler = require('errorhandler');
const app = express();

app.get('/status', (req, res) => {
  res.status(200).end();
});
app.head('/status', (req, res) => {
  res.status(200).end();
});
app.use(cors());
app.use(require('morgan')('dev'));
app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json(setupForStripeWebhooks));
app.use(require('method-override')());
app.use(express.static(__dirname + '/public'));
app.use(session({ secret: process.env.SECRET, cookie: { maxAge: 60000 }, resave: false, saveUninitialized: false }));
mongoose.connect(process.env.DATABASE_URL, { useNewUrlParser: true });

require('./config/passport');
require('./models/user');
require('./models/company');
app.use(require('./routes'));
app.use((req, res, next) => {
  var err = new Error('Not Found');
  err.status = 404;
  next(err);
});
app.use((err, req, res) => {
  res.status(err.status || 500);
  res.json({
             'errors': {
               message: err.message,
               error: {}
             }
           });
});

// ... more stuff 

// ... maybe start up Redis

// ... maybe add more middlewares

async function startServer() {
  app.listen(process.env.PORT, err => {
    if (err) {
      console.log(err);
      return;
    }
    console.log(`Your server is ready !`);
  });
}

// Run the async function to start our server
startServer();
```

As you see, this part of your application can be a real mess.

Here is an effective way to deal with it.

```typescript
const loaders = require('./loaders');
const express = require('express');

async function startServer() {

  const app = express();

  await loaders.init({ expressApp: app });

  app.listen(process.env.PORT, err => {
    if (err) {
      console.log(err);
      return;
    }
    console.log(`Your server is ready !`);
  });
}

startServer();
```

Now the loaders are just tiny files with a concise purpose

```typescript
// loaders/index.ts

import expressLoader from './express';
import mongooseLoader from './mongoose';

export default async ({ expressApp }) => {
  const mongoConnection = await mongooseLoader();
  console.log('MongoDB Initialized');
  await expressLoader({ app: expressApp });
  console.log('Express Initialized');

  // ... more loaders can be here

  // ... Initialize agenda
  // ... or Redis, or whatever you want
}
```

The express loader

```typescript
// loaders/express.ts

import * as express from 'express';
import * as bodyParser from 'body-parser';
import * as cors from 'cors';

export default async ({ app }: { app: express.Application }) => {

  app.get('/status', (req, res) => {
    res.status(200).end();
  });
  app.head('/status', (req, res) => {
    res.status(200).end();
  });
  app.enable('trust proxy');

  app.use(cors());
  app.use(require('morgan')('dev'));
  app.use(bodyParser.urlencoded({ extended: false }));

  // ...More middlewares

  // Return the express app
  return app;
}
```

The mongo loader

```typescript
// loaders/mongoose.ts

import * as mongoose from 'mongoose'

export default async (): Promise<any> => {
  const connection = await mongoose.connect(process.env.DATABASE_URL, { useNewUrlParser: true });
  return connection.connection.db;
}
```

# Conclusion

We deep dive into a production tested node.js project structure, here are some summarized tips:

1. Use a 3 layer architecture.
2. Don’t put your business logic into the express.js controllers.
3. Have dependency injection for your peace of mind.
4. Never leak your passwords, secrets and API keys, use a configuration manager.
5. Split your node.js server configurations into small modules that can be loaded independently.

## [See the example repository here](https://github.com/santiq/bulletproof-nodejs)
