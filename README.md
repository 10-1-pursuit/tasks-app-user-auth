# Back End App w/ User Authentication

## Part 1: Server setup & creating a Task Model

- Set up new node project:
- mkdir tasks-backend
- cd tasks-backend
- touch server.js .env .gitignore
- npm init -y
- touch app.js
- npm i express pg-promise cors dotenv
- Open up your project in VSCode
- Add node_modules and .env to .gitignore file


### Set up the Server

#### app.js
```js
const express = require('express')
const cors = require('cors')
const app = express()

// Middleware
app.use(cors())
app.use(express.json())

app.get('/', (req, res) => {
    res.json({ index: "This is the index page" })
})

module.exports = app
```

#### server.js
```js
const app = require('./app')
require('dotenv').config()

const PORT = process.env.PORT

app.listen(PORT, () => {
    console.log(`Server is running on port: ${PORT}`)
})
```

### Set up the Database
- `mkdir db`
- `touch db/schema.sql db/dbConfig.js db/seed.sql` 


#### schema.sql
```sql
DROP DATABASE IF EXISTS task_app;

CREATE DATABASE task_app;

\c task_app;

CREATE TABLE tasks (
  task_id SERIAL PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  completed BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Add in your seed data
```sql
\c task_app

INSERT INTO tasks (title, description, created_at) VALUES
  ('Study for Interviews', 'Review common interview questions and practice problem-solving.', '2024-01-25 10:00:00'),
  ('Complete 10 LeetCode Problems', 'Solve a set of coding problems on LeetCode for skill improvement.', '2024-01-25 12:30:00'),
  ('Build a CRUD App', 'Create a basic CRUD (Create, Read, Update, Delete) application for hands-on experience.', '2024-01-26 09:15:00'),
  ('Learn a New Programming Language', 'Explore the fundamentals of a programming language you are not familiar with.', '2024-01-26 14:45:00');
```

### Set up the Database configuation in Node

#### dbConfig.js
```js
const pgp = require('pg-promise')()
require('dotenv').config()

const cn = {
    host: process.env.PG_HOST,
    port: process.env.PG_PORT,
    database: process.env.PG_DATABASE,
    user: process.env.PG_USER,
    password: process.env.PG_PASSWORD
}

const db = pgp(cn)

module.exports = db
```

Now in your `package.json` file let's add some custom scripts to make development easier:

Add these underneath line 8, and don't forget the comma on line 8!
```json
"db:init": "psql -U postgres -f db/schema.sql",
"db:seed": "psql -U postgres -f db/seed.sql"
```

Now we're ready to run our SQL files so we have our db set up and seeded w some data. In the terminal, run:
- `npm run db:init`
- `npm run db:seed`

If you go into your Postgres shell and connect to the `task_app` db, you can see the seed data in your db.



### Set up your Tasks Controller & Queries
- `mkdir controllers queries`
- `touch controllers/tasksController.js`
- `touch queries/tasks.js`

We'll start inside of `/queries/tasks.js` with our GET ALL query
```js
const db = require('../db/dbConfig')

const getTasks = async () => {
    try {
        const tasks = await db.any("SELECT * FROM tasks")
        return tasks
    } catch (err){
        return err
    }
}

module.exports = { getTasks }
```

Then we can set up our GET request inside of `/controllers/tasksController.js`
```js
// importing Express
const express = require('express')
// Creating an instance of a Router
const tasks = express.Router()
// Importing db query functions
const { getTasks } = require('../queries/tasks')

//GET all tasks
tasks.get('/', async (req, res) => {
    try {
        const tasks = await getTasks()
        res.status(200).json(tasks)
    } catch (err) {
        res.status(404).json({ error: err })
    }
})

module.exports = tasks
```

Now you can go back and update your `app.js` file to import your controller and test your first route.
- At the top, import your task controller: `const tasksController = require('./controllers/tasksController')`
- Under your middleware add: `app.use('/tasks', tasksController)`

Your `app.js` file should look like this:
```js
const express = require('express')
const cors = require('cors')
const app = express()
const tasksController = require('./controllers/tasksController')

// Middleware
app.use(cors())
app.use(express.json())
app.use('/tasks', tasksController)


app.get('/', (req, res) => {
    res.json({ index: "This is the index page" })
})

module.exports = app
```

Let's test out the route on Postman. You should be able to see your seed data!


Now we'll work on creating the rest of our CRUD routes for getting a single task, creating, updating, and deleting tasks.

Don't forget to `import the functions` we write in our `tasks.js` file into our `tasksController.js`


### Get a single task
#### /queries/tasks.js
```js
const getTask = async (id) => {
    try {
        const task = await db.one("SELECT * FROM tasks WHERE task_id=$1", id)
        return task
    } catch (err) {
        return err
    }
}
```
#### /controllers/tasksController.js
```js
tasks.get('/:id', async (req, res) => {
    const { id } = req.params
    try {
        const task = await getTask(id)
        res.status(200).json(task)
    } catch (err) {
        res.status(404).json({ error: err })
    }
})
```


### Create a new task
#### /queries/tasks.js
```js
const createTask = async (task) => {
    try {
        const completed = task.completed || false
        const newTask = await db.one("INSERT into tasks (title, description, completed, created_at) VALUES ($1, $2, $3, $4) RETURNING *", [task.title, task.description, completed, new Date()]);

        return newTask;
    } catch (err) {
        return err;
    }
};
```

#### /controllers/tasksController.js
```js
tasks.post('/', async (req, res) => {
    try {
        const createdTask = await createTask(req.body)
        res.status(201).json(createdTask)
    } catch (err) {
        res.status(500).json({ error: "Internal Server Error" })
    }
})
```

### Update a task
#### /queries/tasks.js
```js
const updateTask = async (id, task) => {
    try {
        const { title, description, completed, created_at } = task 
        const updatedTask = await db.one("UPDATE tasks SET title=$1, description=$2, completed=$3, created_at=$4 WHERE task_id=$5 RETURNING *", [title, description, completed, created_at, id])
        return updatedTask
    } catch (err) {
        return err
    }
}
```

#### /controllers/tasksController.js
```js
tasks.put('/:id', async (req, res) => {
    try {
        const { id } = req.params
        const updatedTask = await updateTask(id, req.body)
        res.status(200).json(updatedTask)
    } catch (err) {
        res.status(404).json({ error: err})
    }
})
```

### Delete a task
#### /queries/tasks.js
```js
const deleteTask = async (id) => {
    try {
        const deletedTask = await db.one("DELETE FROM tasks WHERE task_id=$1 RETURNING *", id)
        return deletedTask
    } catch (err) {
        return err
    }
}
```
#### /controllers/tasksController.js
```js
tasks.delete('/:id', async (req, res) => {
    try {
        const { id } = req.params
        const deletedTask = await deleteTask(id)
        res.status(200).json({ success: "Successfully deleted task"})
    } catch (err) {
        res.status(404).json({ error: err })
    }
})
```

## Part 2: [Adding a User Model & Authentication]()
