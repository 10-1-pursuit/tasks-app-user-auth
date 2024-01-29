# Part 3: Setting up the Front End

- Create the app `npm create vite@4 tasks-frontend`
- Change into the directory`cd tasks-frontend`
- Install packages `npm i`
- Install React Router DOM & Bootstrap: `npm i react-bootstrap bootstrap react-router-dom`
- Open in VSCode: `code .`
- `touch .env`
- Add `.env` to `.gitignore`
    - In your `.env` add `VITE_BASE_URL=http://localhost:4000`

Add these two imports to your `main.jsx` file
```js
import 'bootstrap/dist/css/bootstrap.min.css'
import { BrowserRouter as Router } from 'react-router-dom'
```
Our app will be using routes for our pages so while still in `main.jsx` wrap your `<App/>` component inside of `<Router>` tags

```js
<React.StrictMode>
    <Router>
      <App />
    </Router>
</React.StrictMode>
```


Empty out the fragment element(<> </>) in App.jsx
- Create Components Folder
- Create Pages Folder

Create a simple Home page, we'll update this later on:
#### Pages/Home.jsx
```jsx
import { Container } from 'react-bootstrap'

const Home = ({ user, token }) => {
    return (
        <Container className="mt-3 text-center">
            { !user ?
                <h2>World Class Task Management</h2>
                :
                <h2>{user.username}'s tasks</h2>
            }
        </Container>
    );
};

export default Home;
```

Create NavBar component:
Inside of `src/Components` create `NavBar.jsx`

#### Components/NavBar.jsx
```jsx
import { Container, Navbar, Nav, Button } from 'react-bootstrap'
import { Link } from 'react-router-dom'

const NavBar = () => {
    return (
        <Navbar>
            <Container>
                <Navbar.Brand>
                    <Nav.Link as={Link} to="/">Task Manager</Nav.Link>
                </Navbar.Brand>
                <Nav className='ms-auto'>
                    <Nav.Link as={Link} to="/login">Log in</Nav.Link>
                    <Nav.Link as={Link} to="/signup">Sign up</Nav.Link>
                </Nav>
            </Container>
        </Navbar>
    );
};

export default NavBar;
```

Import the NavBar component into your `App.jsx` and add it inside of your return.

You should see a nicely displayed NavBar that has functional buttons that toggle when you hover over or click on them.

#### Create the Log In & Sign Up Forms

Create a `Pages` folder inside of `src`
Inside of `src/Pages` create your `Signup.jsx`

Inside of your <App/> component, import `Routes` and `Route` from `react-router-dom` at the top of the file, Then after the <NavBar/> create a <Routes> component and add a <Route> for the Home and another for the Signup component:
#### src/App.jsx
```jsx
<div>
    <NavBar />
    <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/signup" element={<Signup />} />
    </Routes>
</div>
```

### Now create the Signup form
#### Pages/Signup.jsx
```jsx
import React, { useState } from 'react';
import { Form, Button, Row, Col, Container } from 'react-bootstrap'

const Signup = () => {
    const API = import.meta.env.VITE_BASE_URL
    const [formData, setFormData] = useState({
        username: '',
        email: '',
        password_hash: ''
    })

    const handleInputChange = (event) => {
        const { name, value } = event.target
        setFormData((prev) => ({
            ...prev,
            [name]: value
        }))
    }

    const handleSubmit = (event) => {
        event.preventDefault()
        console.log("State", formData)
        fetch(`${API}/users`, {
            method: "POST",
            body: JSON.stringify(formData),
            headers: {
                "Content-Type": "application/json"
            }
        })
            .then(res => res.json())
            .then(res => {
                console.log(res)
                setFormData((prev) => ({
                    username: '',
                    email: '',
                    password_hash: ''
                }))
            })
            .catch(err => console.log(err))
    }

    return (
        <Container style={{ marginTop: "50px" }}>
            <Row>
                <Col md={6}>
                    <Form onSubmit={handleSubmit}>
                        <Form.Group className="mb-3" controlId="username">
                            <Form.Label>Username</Form.Label>
                            <Form.Control
                            type="text"
                            placeholder="Enter your username"
                            name="username"
                            value={formData.username}
                            onChange={handleInputChange}
                            />
                        </Form.Group>

                        <Form.Group className="mb-3" controlId="email">
                            <Form.Label>Email</Form.Label>
                            <Form.Control
                            type="email"
                            placeholder="Enter your email"
                            name="email"
                            value={formData.email}
                            onChange={handleInputChange}
                            />
                        </Form.Group>

                        <Form.Group className="mb-3" controlId="password">
                            <Form.Label>Password</Form.Label>
                            <Form.Control
                            type="password"
                            placeholder="Enter your password"
                            name="password_hash"
                            value={formData.password_hash}
                            onChange={handleInputChange}
                            />
                        </Form.Group>
                        <Button variant="primary" type="submit">Submit</Button>
                    </Form>
                </Col>
            </Row>
        </Container>
    );
};

export default Signup;
```

This form might look brand new, but we've already done most of this before, the only thing that is new is that we are using **Bootstrap components** to create our form. 

These Bootstrap components come with some styling straight out of the box.
You can update the styles however you'd like by using the name of the component in a CSS file as the class name. For example you would select the `<Container />` component in CSS using `.container` or you can get more specific and use the inline-styling like we used on the container.

For now, the Form only logs the user to the console, we need to go back to our `<App/>` component to add a piece of state each for both the user and the token.


#### src/App.jsx
```jsx
const [user, setUser] = useState(null)
const [token, setToken] = useState(null)
```

Hand both of the functions down to your `<Signup/>` component as props inside of your `<Route>`, so you can update your state:
```jsx
<Route path="/signup" element={<Signup setUser={setUser} setToken={setToken} />} />
```

Now back in your `<Signup/>` receive them as props, and use them inside of your fetch call to update your state.

#### Components/Signup.jsx
```jsx
if(res.user.user_id){
    const { user, token } = res
    setUser(user)
    setToken(token)
    setFormData(() => ({
        username: '',
        email: '',
        password_hash: ''
    }))
} else {
    console.log(res)
}
```

Now when you sign up, you should see your state variables update. You can verify this in your React dev tools.


Great. Let's update the `<NavBar/>` so that it shows the user's username and a Logout button when a user is logged in:

In `<App/>` give the `<NavBar/>` the `user` `setUser` and `setToken` as props.

Then receive those props in your NavBar so we can use them.

Create a handleLogout function so when we click on the Logout button we can clear the user and the token out of state

#### Components/NavBar.jsx
```jsx
const handleLogout = () => {
    setUser(null)
    setToken(null)
}
```

Now under your `<Nav.Brand>`, we have out `<Nav>` that contains our Login and Signup button. We only want to see those buttons when a user is not signed in.

Update it so that they only render when there is no user saved in our App's state:
```jsx
</Navbar.Brand>
{ !user ? 
    <Nav className='ms-auto'>
        <Nav.Link as={Link} to="/login">Login</Nav.Link>
        <Nav.Link as={Link} to="/signup">Sign up</Nav.Link>
    </Nav>
    :
    <Nav className='ms-auto'>
        <Nav.Link as={Link} to="/profile">{user.username}</Nav.Link>
        <Button variant="outline-light" onClick={handleLogout} size="sm" style={{ color: "black" }}>Log Out</Button>
    </Nav>
}
</Container>
```

Let's create the Login page next:

### Pages/Login.jsx
```jsx
import React, { useState } from 'react';
import { Form, Button, Container, Row, Col } from 'react-bootstrap';
import { useNavigate } from 'react-router-dom';

const Login = ({ setUser, setToken }) => {
  const API = import.meta.env.VITE_BASE_URL
  const navigate = useNavigate();

  const [formData, setFormData] = useState({
    username: '',
    password_hash: '',
  });

  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setFormData((prev) => ({
      ...prev,
      [name]: value,
    }));
  };

  const handleLogin = (e) => {
    e.preventDefault();

    fetch(`${API}/users/login`, {
        method: "POST",
        body: JSON.stringify(formData),
        headers: {
            "Content-Type": "application/json"
        }
    })
        .then(res => res.json())
        .then(res => {
            console.log(res)
            if(res.user.user_id){
                const { user, token } = res
                setUser(user)
                setToken(token)
                setFormData(() => ({
                    username: '',
                    password_hash: ''
                }))
                navigate('/');
            } else {
                console.log(res)
            }
        })
        .catch(err => console.log(err))

    
  };

  return (
    <Container style={{ marginTop: "50px" }}>
      <Row className="justify-content-md-center">
        <Col md={6}>
          <Form onSubmit={handleLogin}>
            <Form.Group className="mb-3" controlId="username">
              <Form.Label>Username</Form.Label>
              <Form.Control
                type="text"
                placeholder="Enter your username"
                name="username"
                value={formData.username}
                onChange={handleInputChange}
              />
            </Form.Group>

            <Form.Group className="mb-3" controlId="password">
              <Form.Label>Password</Form.Label>
              <Form.Control
                type="password"
                placeholder="Enter your password"
                name="password_hash"
                value={formData.password_hash}
                onChange={handleInputChange}
              />
            </Form.Group>

            <Button variant="primary" type="submit">
              Log in
            </Button>
          </Form>
        </Col>
      </Row>
    </Container>
  );
};

export default Login;
```

Back in your `<App/>` component, make sure to import the Login component, make a `<Route>` for it and hand it down the `setUser` and `setToken` props.

This is what your `<App/>` component should look like at this point:

#### App.jsx
```jsx
import { useState } from 'react'
import './App.css'
import NavBar from './Components/NavBar'
import Signup from './Pages/Signup'
import Login from './Pages/Login'
import Home from './Pages/Home'
import { Routes, Route } from 'react-router-dom'

function App() {
  const [user, setUser] = useState(null)
  const [token, setToken] = useState(null)

  return (
    <div>
      <NavBar user={user} setUser={setUser} setToken={setToken} />
      <Routes>
        <Route path='/' element={<Home user={user} token={token}/>} />
        <Route path='/login' element={<Login setUser={setUser} setToken={setToken}/>} />
        <Route path="/signup" element={<Signup setUser={setUser} setToken={setToken} />} />
      </Routes>
      
    </div>
  )
}

export default App
```
