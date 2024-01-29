# Part 2: Adding a Users Model and Setting Up Authentication


Fork and Clone if you need the starter code: [Task Backend / One Model](https://github.com/10-1-pursuit/task-management-app)

First install the necessary packages:
- Run `npm i jsonwebtoken bcrypt`

Next let's update our `schema.sql` and `seed.sql` files to show the users table and the relationship between users and tasks, as well as update the seed data so each task belongs to a user:


#### schema.sql
```sql
DROP DATABASE IF EXISTS task_app;

CREATE DATABASE task_app;

\c task_app;

CREATE TABLE users (
  user_id SERIAL PRIMARY KEY,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tasks (
  task_id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  completed BOOLEAN DEFAULT false,
  user_id INT REFERENCES users(user_id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### seed.sql
```sql
\c task_app

INSERT INTO users (username, email, password_hash) VALUES
  ('john_doe', 'john@example.com', 'password123'),
  ('jane_smith', 'jane@example.com', 'letmein2024'),
  ('alex_wong', 'alex@example.com', 'securepassword'),
  ('sara_jones', 'sara@example.com', 'p@ssw0rd');

INSERT INTO tasks (title, description, completed, user_id, created_at) VALUES
  ('Study for Interviews', 'Review common interview questions and practice problem-solving.', false, 1, '2024-01-25 10:00:00'),
  ('Complete 10 LeetCode Problems', 'Solve a set of coding problems on LeetCode for skill improvement.', false, 2, '2024-01-25 12:30:00'),
  ('Build a CRUD App', 'Create a basic CRUD (Create, Read, Update, Delete) application for hands-on experience.', false, 3, '2024-01-26 09:15:00'),
  ('Learn a New Programming Language', 'Explore the fundamentals of a programming language you are not familiar with.', false, 4, '2024-01-26 14:45:00'),
  ('Implement RESTful API', 'Develop RESTful endpoints for your application.', false, 1, '2024-01-27 11:00:00'),
  ('Write Unit Tests', 'Create unit tests to ensure code quality and reliability.', false, 2, '2024-01-28 09:30:00'),
  ('Deploy Application to Heroku', 'Deploy your application to Heroku for production use.', false, 3, '2024-01-29 14:00:00'),
  ('Refactor Codebase', 'Optimize and improve the structure of your codebase.', false, 4, '2024-01-30 10:45:00'),
  ('Attend Tech Meetup', 'Participate in a local tech meetup to network and learn from others.', false, 1, '2024-01-31 18:00:00'),
  ('Review Design Patterns', 'Study common design patterns and their applications in software development.', false, 2, '2024-02-01 13:15:00');
```

### Set up the User Controller & Queries
- `touch controllers/usersController.js`
- `touch queries/users.js`

#### queries/users.js
```js
// Import db connection
const db = require('../db/dbConfig')
// Import library to hash password
const bcrypt = require('bcrypt')


const createUser = async (user) => {
    try {
        const { username, email, password_hash } = user
        const salt = 10
        const hash = await bcrypt.hash(password_hash, salt)
        const newUser = await db.one("INSERT INTO users (username, email, password_hash) VALUES($1, $2, $3) RETURNING *", [username, email, hash])
        return newUser
    } catch (err) {
        return err
    }
}

const getUsers = async () => {
    try {
        const users = await db.any("SELECT * FROM users")
        return users
    } catch (err) {
        return err
    }
}


module.exports = { createUser, getUsers }
```

#### controllers/usersController.js
```js
const express = require('express')
const users = express.Router()
require('dotenv').config()
// Package to generate tokens to authenticate users when sending requests
const jwt = require('jsonwebtoken')
// Secret string from .env used when function to create a token is called
const secret = process.env.SECRET
const { getUsers, createUser } = require('../queries/users')


users.get('/', async (req, res) => {
    try {
        const users = await getUsers()
        res.status(200).json(users)
    } catch (error) {
        res.status(500).json({ "error": "Internal Server Error" })
    }
});

users.post('/', async (req, res) => {
    try {
        const newUser = await createUser(req.body)
        // JWT package has a method called sign, to which we pass an object containing a user's id and the user's username, AND the 'secret' from your environment variables to create a 'token' that we assign to the variable token
        const token = jwt.sign({ userId: newUser.user_id, username: newUser.username }, secret);

        // We create a new object to send both the user and the token to the client
        res.status(201).json({ user: newUser, token });
    } catch (error) {
        res.status(500).json({ error: "Invalid information", info: error });
    }
});


module.exports = users
```

// Now back in app.js you can import the users controller and have your app use it
- import the user controller at the top of your file:
    - `const usersController = require('./controllers/usersController.js')`
- add the user controller to your middleware underneath the other middleware
    - `app.use('/users', usersController)`


Test out your ability to GET all users and then to POST a new user
(your client won't GET all users in this app, it's just so we can see that it was added to the db)

You may notice that we are generating a token when a user is created. That is purely for the benefit of the user(good UX). The user is automatically logged in when they successfully create an account, so there is no need to fill out the form twice(sign-up then log-in) This is NOT a requirement.

### Log a User in

#### queries/users.js
- Make sure to export the log-in function at the bottom of your file with the others!

```js
const logInUser = async (user) => {
    try {
        const loggedInUser = await db.oneOrNone("SELECT * FROM users WHERE username=$1", user.username)
        // console.log(loggedInUser)
        if(!loggedInUser){
            return false
        }

        const passwordMatch = await bcrypt.compare(user.password_hash, loggedInUser.password_hash)

        if(!passwordMatch){
            return false
        }
        return loggedInUser
    } catch (err) {
        return err
    }
}
```

#### controllers/usersController.js
```js
users.post('/login', async (req, res) => {
    try {
        const user = await logInUser(req.body);
        if(!user){
            res.status(401).json({ error: "Invalid username or password" })
            return // Exit the function
        }

        const token = jwt.sign({ userId: user.user_id, username: user.username }, secret);

        res.status(200).json({ 
            id: user.user_id, 
            user: user.username, 
            email: user.email, 
            token 
        });
    } catch (err) {
        res.status(500).json({ error: "Internal server error" })
    }
})
```

Now our users can log in and we can send the user info back to the client along with a token!

### Updating our Task routes and query functions
Ok now we are going to merge our URL params. Import the tasks controller into your usersController.js and have the users router use it:

#### controllers/usersController.js
```js
const tasksController = require('./tasksController')
users.use('/:user_id/tasks', tasksController)
```
We do this so we can get tasks related to only the user that is logged in, instead of seeing everyone's tasks. This is a task management app after all, not a social app.

Now we have to tell our tasks controller that we are merging URL params. Update your tasks router:

#### controllers/tasksController.js
```js
const tasks = express.Router({ mergeParams: true })
```

Great! Now that that part is set up, we can update our query functions and our routes.

Update your GET all tasks function to accept a user id as a parameter, and update your call to the database to only select tasks that the user created
#### queries/tasks.js
```js
const getTasks = async (userId) => {
    try {
        const tasks = db.any("SELECT * FROM tasks WHERE user_id=$1", userId)
        return tasks
    } catch (err){
        return err
    }
}
```
Now we have to update the route as well
#### controllers/tasksController.js
```js
tasks.get('/', async (req, res) => {
    try {
        const { user_id } = req.params
        const tasks = await getTasks(user_id)
        res.status(200).json(tasks)
    } catch (err) {
        res.status(404).json({ error: err })
    }
})
```

Now we can test this out. In Postman or the browser, try out this route `localhost:4000/users/1/tasks`

We can remove the tasksController from our app.js since it no longer makes sense to have a route to see all the tasks.

#### Update a single task query and route:
#### queries/tasks.js
```js
const getTask = async (id, userId) => {
    try {
        const task = db.one("SELECT * FROM tasks WHERE task_id=$1 AND user_id=$2", [id, userId])
        return task
    } catch (err) {
        return err
    }
}
```

#### controllers/tasksController.js
```js
tasks.get('/:id', async (req, res) => {
    const { id, user_id } = req.params
    try {
        const task = await getTask(id, user_id)
        res.status(200).json(task)
    } catch (err) {
        res.status(404).json({ error: err })
    }
})
```
Make sure to test each one of these routes out as you update them!

#### Update the Create task & Update task queries
#### queries/tasks.js
We add the `user_id` in as one of the columns to be inserted, and add `task.user_id` to the values array (Don't forget the $5)
```js
const createTask = async (task) => {
    try {
        const completed = task.completed || false
        const newTask = await db.one("INSERT into tasks (title, description, completed, created_at, user_id) VALUES ($1, $2, $3, $4, $5) RETURNING *", [task.title, task.description, completed, new Date(), task.user_id]);

        return newTask;
    } catch (err) {
        return err
    }
}

// Destructure user_id out of task and add it to the query method
const updateTask = async (id, task) => {
    try {
        const { title, description, completed, created_at, user_id } = task 
        const updatedTask = await db.one("UPDATE tasks SET title=$1, description=$2, completed=$3, created_at=$4, user_id=$5 WHERE task_id=$6 RETURNING *", [title, description, completed, created_at, user_id, id])
        return updatedTask
    } catch (err) {
        return err
    }
}
```

Now that our models are linked, we can finally add authentication!

<!-- create an auth folder and create an auth file -->
### Add authentication to your Task routes
- `mkdir auth`
- `touch auth/auth.js`

#### auth/auth.js
```js
const jwt = require('jsonwebtoken')
require('dotenv').config()
const secret = process.env.SECRET

const authenticateToken = (req, res, next) => {
    const token = req.header('Authorization')
    if(!token){
        return res.status(401).json({ "error": "Access Denied. No token provided" })
    }
    jwt.verify(token, secret, (err, user) => {
        if (err) return res.status(403).json({ error: 'Invalid token.' });
        req.user = user;
        next();
    })
}

module.exports = { authenticateToken }
```
Now we can import this authentication function into our tasks controller and use it in our routes as `middleware`

Example:
```js
tasks.get('/', authenticateToken, async (req, res) => {
    try {
        const { user_id } = req.params
        const tasks = await getTasks(user_id)
        res.status(200).json(tasks)
    } catch (err) {
        res.status(404).json({ error: err })
    }
})
```

Be sure to add this to all of your routes. 

#### That's it! 💪😎

[Completed Code](https://github.com/10-1-pursuit/tasks-auth)


