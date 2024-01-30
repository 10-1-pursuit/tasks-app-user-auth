# Part 4: Authenticated CRUD for Tasks & Protected Routes
- [Part 3 React Code](https://github.com/10-1-pursuit/tasks-front)
    - Clone, run `npm i` then `npm run dev` to start the server

First, let's start by updating our Home component to look how it did originally. We used a ternary to display the username when we were testing, but now we'll build out an actual Tasks component.
```jsx
import { Container } from 'react-bootstrap'

const Home = () => {
    return (
        <Container className="mt-5 text-center">
                <h2>World Class Task Management</h2>
        </Container>
    );
};

export default Home;
```
Let's also update the `Nav.Link` in the `NavBar` that displays the user's `username` so it takes us to `/tasks` instead of `/profile`.

### Create the ProtectedRoute component (HOC: [Higher-Order Component](https://legacy.reactjs.org/docs/higher-order-components.html))
This component will take in some props: `user` and `token` of course, as well as an `element` which will represent a component that we want to render, and `isAuthenticated` which will be a boolean.

This component also imports `Navigate` which is another component similar to `useNavigate` but note the difference is that `useNavigate` is a hook that we use as a function, and `Navigate` is used as a component.


#### Pages/ProtectedRoute.jsx
```jsx
import { Navigate } from 'react-router-dom'

const ProtectedRoute = ({ element: Component, isAuthenticated, user, token }) => {
    return isAuthenticated ? (
        <Component user={user} token={token}/>
    ) :
    (
        <Navigate to="/login" replace />
    )
};

export default ProtectedRoute;
```

Now go to your `<Signup/>` and `<Login/>` components and make it so they redirect to `/tasks` instead of the home page. We'll build this component next but set this part up.

Import `useNavigate` in your `<Signup/>`, then initialize a variable called `navigate` and use the hook, then use the navigate variable in your `handleSubmit` function. (refer to the Login page and the handleLogin function if you forget how)


Back in `App.jsx`, import your <ProtectedRoute/> and let's use it.

```jsx
<Route 
    path="/tasks"
    element={
        <ProtectedRoute 
            element={Tasks}
            isAuthenticated={!!user && !!token}
            user={user}
            token={token}
        />
    }
/>
```
Let's break down what is happening. We created a route that will render our ProtectedRoute component when the path in the URL is `/tasks`. We hand it down the Tasks component(that we'll make next) as well as the other props we talked about.

`isAuthenticated` gets the value of `true` or `false` based on whether both of the state variables `user` and `token` exist or not.

Don't forget to import `Tasks` at the top of your `App.jsx` so it doesn't throw an error


In your `<Login />` let's update where we navigate to after we log in. Instead of directing us home: `/`, let's redirect to view our `/tasks` component. We'll build out the component for tasks next.


### Create the Tasks component
```jsx
import React, { useState, useEffect } from 'react';
import { Table, Button, Container } from 'react-bootstrap';

const Tasks = ({ user, token }) => {
  const API = import.meta.env.VITE_BASE_URL
  const [tasks, setTasks] = useState([])


  const completeTask = (id) => {
    const task = tasks.find(task => task.task_id === id)
    const updatedTask = { ...task, completed: !task.completed }
    fetch(`${API}/users/${user.user_id}/tasks/${id}`, {
        method: "PUT",
        body: JSON.stringify(updatedTask),
        headers: {
            "Content-Type": "application/json",
            "Authorization": token
        }
    })
        .then(res => res.json())
        .then(res => {
            // Map over the previous state of tasks to create a new array w the updated task
            setTasks((prevTasks) => prevTasks.map(task => task.task_id === id ? updatedTask : task))

        })
        .catch(err => console.log(err))
  }

  useEffect(() => {
    fetch(`${API}/users/${user.user_id}/tasks`, {
        headers: {
            "Authorization": token
        }
    })
        .then(res => res.json())
        .then(res => setTasks(() => res))
  }, [user, tasks])

  // Sort the tasks by ID number so they keep the same order when updating them
  const sortedTasks = tasks.sort((a , b) => a.task_id < b.task_id ? -1 : 1)

  return (
    <Container>
      <h2>Task List</h2>
      <Table striped bordered hover responsive className="table-sm">
        <thead>
          <tr>
            <th>Title</th>
            <th>Description</th>
            <th>Status</th>
            <th>Action</th>
          </tr>
        </thead>
        <tbody>
          {tasks.length > 0 && sortedTasks.map((task) => (
            <tr key={task.id}>
              <td>{task.title}</td>
              <td>{task.description}</td>
              <td>{task.completed ? 'Completed' : 'Incomplete'}</td>
              <td>
                <Button
                  variant={task.completed ? 'light' : 'dark'}
                  onClick={() => completeTask(task.task_id)}
                >
                  {task.completed ? 'Undo' : 'Complete'}
                </Button>
              </td>
            </tr>
          ))}
        </tbody>
      </Table>
    </Container>
  );
};

export default Tasks;
```

This page is pretty large and there is a lot going on. First import the what we need at the top, and then create the `table`.

Inside the table body is where we'll map over our tasks and create a table row for each task. We'll need to make a `GET request` to our server to see the tasks, but only after adding some tasks via `Postman` since we don't have a Create Task page yet.

Next add the useEffect so we can make a call to the server when this page is rendered. Make sure to import the base url from .env and create a state variable to hold the tasks after we get them from our request.

Then, you can work on the `completeTask` method which will act as a `toggler` to complete the app or undo the completion.

Finally write a function to sort the tasks by `task_id` so they stay in the same order instead of moving, every time you complete one.
