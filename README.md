# Django-React-Todo-App-with-Authentication

Here's a detailed guide on creating a Todo application with user authentication using Django (with function-based views) and React. This guide will walk you through setting up both applications from scratch, including user authentication and CRUD operations for the Todo items. Each step will be explained with reasons for why we use certain technologies and practices.

# Django + React Todo App with Authentication

## Prerequisites

Before we begin, make sure you have the following installed:

- Python 3.x: The programming language we will use for the backend.
- Node.js and npm: Required for running and managing the React frontend.
- Django: A high-level Python web framework for the backend.
- Create React App (CRA): A tool to set up a modern web app by running one command.
- Virtualenv: To create isolated Python environments.

## Backend: Django Setup

### Step 1: Setting Up the Django Project

1. **Install Django**: Django is a powerful web framework for Python that makes it easier to build web applications quickly.
   ```bash
   pip install django
   ```

2. **Create a Django Project**: This initializes a new Django project named `backend`.
   ```bash
   django-admin startproject backend
   cd backend
   ```

3. **Create a Django App**: Apps are a way to organize your project into functional modules. Here, we create an app named `todos`.
   ```bash
   python manage.py startapp todos
   ```

4. **Install Django Rest Framework**: This library makes it easy to build web APIs with Django.
   ```bash
   pip install djangorestframework
   ```

5. **Install Django REST framework simple JWT**: For handling JSON Web Tokens (JWT) authentication, which is stateless and ideal for modern web applications.
   ```bash
   pip install djangorestframework-simplejwt
   ```

6. **Add `rest_framework` and `todos` to your `INSTALLED_APPS` in `backend/settings.py`**:
   ```python
   INSTALLED_APPS = [
       ...
       'rest_framework',
       'todos',
       'rest_framework_simplejwt',
   ]
   ```

7. **Configure Django Rest Framework and Simple JWT in `backend/settings.py`**:
   ```python
   REST_FRAMEWORK = {
       'DEFAULT_AUTHENTICATION_CLASSES': (
           'rest_framework_simplejwt.authentication.JWTAuthentication',
       ),
   }
   ```
   This configuration ensures that our API uses JWT for authentication.

8. **Run Migrations**: This sets up the database tables for our project.
   ```bash
   python manage.py migrate
   ```

### Step 2: Creating Models, Serializers, and Views

1. **Update the `todos/models.py` with the Todo model**:
   ```python
   from django.db import models
   from django.contrib.auth.models import User

   class Todo(models.Model):
       user = models.ForeignKey(User, on_delete=models.CASCADE)
       title = models.CharField(max_length=200)
       description = models.TextField(blank=True)
       completed = models.BooleanField(default=False)
       created_at = models.DateTimeField(auto_now_add=True)

       def __str__(self):
           return self.title
   ```
   The Todo model represents the structure of a todo item, linking each item to a user via a foreign key relationship.

2. **Create serializers in `todos/serializers.py`**:
   ```python
   from rest_framework import serializers
   from .models import Todo

   class TodoSerializer(serializers.ModelSerializer):
       class Meta:
           model = Todo
           fields = '__all__'
   ```
   Serializers convert complex data types like querysets and model instances to native Python data types for rendering into JSON and vice versa.

3. **Create views in `todos/views.py`**:
   ```python
   from rest_framework.decorators import api_view, permission_classes
   from rest_framework.response import Response
   from rest_framework import status
   from rest_framework.permissions import IsAuthenticated
   from .models import Todo
   from .serializers import TodoSerializer

   @api_view(['GET'])
   @permission_classes([IsAuthenticated])
   def get_todos(request):
       todos = request.user.todo_set.all()
       serializer = TodoSerializer(todos, many=True)
       return Response(serializer.data)

   @api_view(['POST'])
   @permission_classes([IsAuthenticated])
   def create_todo(request):
       serializer = TodoSerializer(data=request.data)
       if serializer.is_valid():
           serializer.save(user=request.user)
           return Response(serializer.data, status=status.HTTP_201_CREATED)
       return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

   @api_view(['GET', 'PUT', 'DELETE'])
   @permission_classes([IsAuthenticated])
   def manage_todo(request, pk):
       try:
           todo = Todo.objects.get(pk=pk, user=request.user)
       except Todo.DoesNotExist:
           return Response(status=status.HTTP_404_NOT_FOUND)

       if request.method == 'GET':
           serializer = TodoSerializer(todo)
           return Response(serializer.data)

       elif request.method == 'PUT':
           serializer = TodoSerializer(todo, data=request.data)
           if serializer.is_valid():
               serializer.save()
               return Response(serializer.data)
           return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

       elif request.method == 'DELETE']:
           todo.delete()
           return Response(status=status.HTTP_204_NO_CONTENT)
   ```
   These views handle the CRUD operations (Create, Read, Update, Delete) for the Todo items.

4. **Add URLs in `todos/urls.py`**:
   ```python
   from django.urls import path
   from . import views

   urlpatterns = [
       path('todos/', views.get_todos),
       path('todos/create/', views.create_todo),
       path('todos/<int:pk>/', views.manage_todo),
   ]
   ```
   This routes the URLs to the appropriate views.

5. **Include `todos` URLs in `backend/urls.py`**:
   ```python
   from django.contrib import admin
   from django.urls import path, include

   urlpatterns = [
       path('admin/', admin.site.urls),
       path('api/', include('todos.urls')),
   ]
   ```

### Step 3: Setting Up JWT Authentication

1. **Add JWT endpoints to `backend/urls.py`**:
   ```python
   from rest_framework_simplejwt.views import (
       TokenObtainPairView,
       TokenRefreshView,
   )

   urlpatterns = [
       ...
       path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
       path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
   ]
   ```
   These endpoints are used to obtain and refresh JWT tokens.

### Step 4: Testing the API

1. **Create a superuser to access the Django admin**:
   ```bash
   python manage.py createsuperuser
   ```

2. **Run the Django server**:
   ```bash
   python manage.py runserver
   ```

3. **Access the Django admin at `http://127.0.0.1:8000/admin` to create Todo items and users.**

## Frontend: React Setup

### Step 1: Setting Up the React Project

1. **Install Create React App globally if you haven't already**: This initializes a new React project named `frontend`.
   ```bash
   npx create-react-app frontend
   cd frontend
   ```

2. **Install Axios and JWT-decode**: Axios is a promise-based HTTP client for the browser and node.js. JWT-decode is used to decode JSON web tokens.
   ```bash
   npm install axios jwt-decode
   ```

### Step 2: Creating React Components

1. **Create a folder structure**:
   ```bash
   mkdir -p src/components src/services
   ```

2. **Create `authService.js` in `src/services` for handling authentication**:
   ```javascript
   import axios from 'axios';
   const API_URL = 'http://127.0.0.1:8000/api/token/';

   const login = (username, password) => {
       return axios.post(API_URL, {
           username,
           password
       });
   };

   const authService = {
       login,
   };

   export default authService;
   ```
   This service handles user login by sending credentials to the Django backend and receiving JWT tokens.

3. **Create `todoService.js` in `src/services` for handling Todo API requests**:
   ```javascript
   import axios from 'axios';

   const API_URL = 'http://127.0.0.1:8000/api/todos/';

   const getTodos = (token) => {
       return axios.get(API_URL, {
           headers: { Authorization: `Bearer ${token}` }
       });
   };

   const createTodo = (token, todo) => {
       return axios.post(`${API_URL}create/`, todo, {
           headers: { Authorization: `Bearer ${token}` }
       });
   };

   const updateTodo = (token, id, todo) => {
       return axios.put(`${API_URL}${id}/`, todo, {
           headers: { Authorization: `Bearer ${token}` }
       });
   };

   const deleteTodo = (token, id) => {
       return axios.delete(`${API_URL}${id}/`, {
           headers: { Authorization: `Bearer ${token}` }
       });
   };

   const todoService = {
       getTodos,
       createTodo,
       updateTodo,
       deleteTodo
   };

   export default todoService;
   ```
   This service manages CRUD operations for Todo items by interacting with the Django backend.

4. **Create `Login.js` in `src/components`**:
   ```

javascript
   import React, { useState } from 'react';
   import authService from '../services/authService';

   function Login({ setToken }) {
       const [username, setUsername] = useState('');
       const [password, setPassword] = useState('');

       const handleSubmit = async (e) => {
           e.preventDefault();
           try {
               const { data } = await authService.login(username, password);
               setToken(data.access);
           } catch (error) {
               console.error("Invalid credentials");
           }
       };

       return (
           <form onSubmit={handleSubmit}>
               <div>
                   <label>Username:</label>
                   <input type="text" value={username} onChange={(e) => setUsername(e.target.value)} />
               </div>
               <div>
                   <label>Password:</label>
                   <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} />
               </div>
               <button type="submit">Login</button>
           </form>
       );
   }

   export default Login;
   ```
   This component handles user login, capturing the username and password, and sending them to the backend to obtain a JWT token.

5. **Create `TodoList.js` in `src/components`**:
   ```javascript
   import React, { useEffect, useState } from 'react';
   import todoService from '../services/todoService';

   function TodoList({ token }) {
       const [todos, setTodos] = useState([]);
       const [newTodo, setNewTodo] = useState({ title: '', description: '' });

       useEffect(() => {
           const fetchTodos = async () => {
               try {
                   const { data } = await todoService.getTodos(token);
                   setTodos(data);
               } catch (error) {
                   console.error("Error fetching todos");
               }
           };

           fetchTodos();
       }, [token]);

       const handleCreateTodo = async (e) => {
           e.preventDefault();
           try {
               const { data } = await todoService.createTodo(token, newTodo);
               setTodos([...todos, data]);
               setNewTodo({ title: '', description: '' });
           } catch (error) {
               console.error("Error creating todo");
           }
       };

       const handleUpdateTodo = async (id, updatedTodo) => {
           try {
               const { data } = await todoService.updateTodo(token, id, updatedTodo);
               setTodos(todos.map(todo => (todo.id === id ? data : todo)));
           } catch (error) {
               console.error("Error updating todo");
           }
       };

       const handleDeleteTodo = async (id) => {
           try {
               await todoService.deleteTodo(token, id);
               setTodos(todos.filter(todo => todo.id !== id));
           } catch (error) {
               console.error("Error deleting todo");
           }
       };

       return (
           <div>
               <h2>Todo List</h2>
               <form onSubmit={handleCreateTodo}>
                   <input
                       type="text"
                       placeholder="Title"
                       value={newTodo.title}
                       onChange={(e) => setNewTodo({ ...newTodo, title: e.target.value })}
                   />
                   <input
                       type="text"
                       placeholder="Description"
                       value={newTodo.description}
                       onChange={(e) => setNewTodo({ ...newTodo, description: e.target.value })}
                   />
                   <button type="submit">Add Todo</button>
               </form>
               <ul>
                   {todos.map(todo => (
                       <li key={todo.id}>
                           <input
                               type="text"
                               value={todo.title}
                               onChange={(e) => handleUpdateTodo(todo.id, { ...todo, title: e.target.value })}
                           />
                           <input
                               type="text"
                               value={todo.description}
                               onChange={(e) => handleUpdateTodo(todo.id, { ...todo, description: e.target.value })}
                           />
                           <button onClick={() => handleDeleteTodo(todo.id)}>Delete</button>
                       </li>
                   ))}
               </ul>
           </div>
       );
   }

   export default TodoList;
   ```
   This component displays the list of todos, allowing users to create, update, and delete todo items.

6. **Update `App.js` to include the Login and TodoList components**:
   ```javascript
   import React, { useState } from 'react';
   import Login from './components/Login';
   import TodoList from './components/TodoList';

   function App() {
       const [token, setToken] = useState('');

       return (
           <div className="App">
               {!token ? <Login setToken={setToken} /> : <TodoList token={token} />}
           </div>
       );
   }

   export default App;
   ```
   The main App component conditionally renders the Login or TodoList component based on whether the user is authenticated.

### Step 3: Running the React App

1. **Start the React application**:
   ```bash
   npm start
   ```

## Final Notes

1. **Ensure Django server is running on `http://127.0.0.1:8000` and React app on `http://localhost:3000`.**
2. **Create users via Django admin and use them to log in from the React app.**

This guide covers the essential steps to create a simple Todo application with user authentication using Django and React. You can further enhance this app by adding more features, improving the UI, and handling edge cases.

### README.md

Here's a `README.md` file that explains the setup process:

```markdown
# Django + React Todo App with Authentication

This is a Todo application built with Django (backend) and React (frontend). The app includes user authentication and CRUD operations for Todo items.

## Prerequisites

- Python 3.x
- Node.js and npm
- Django
- Create React App (CRA)
- Virtualenv

## Setup Instructions

### Backend (Django)

1. **Install Django and create a Django project:**
   ```bash
   pip install django
   django-admin startproject backend
   cd backend
   ```

2. **Create a Django app and install dependencies:**
   ```bash
   python manage.py startapp todos
   pip install djangorestframework djangorestframework-simplejwt
   ```

3. **Update `backend/settings.py`:**
   - Add `rest_framework` and `todos` to `INSTALLED_APPS`.
   - Configure Django Rest Framework and Simple JWT.

4. **Create models, serializers, and views in `todos` app.**

5. **Add URLs in `todos/urls.py` and include them in `backend/urls.py`.**

6. **Run migrations and create a superuser:**
   ```bash
   python manage.py migrate
   python manage.py createsuperuser
   ```

7. **Run the Django server:**
   ```bash
   python manage.py runserver
   ```

### Frontend (React)

1. **Create a React app and install dependencies:**
   ```bash
   npx create-react-app frontend
   cd frontend
   npm install axios jwt-decode
   ```

2. **Create services for authentication and Todo API requests.**

3. **Create React components for login and Todo list management.**

4. **Update `App.js` to include the Login and TodoList components.**

5. **Start the React application:**
   ```bash
   npm start
   ```

## Running the Application

- Ensure Django server is running on `http://127.0.0.1:8000`.
- Ensure React app is running on `http://localhost:3000`.
- Create users via Django admin and use them to log in from the React app.

## License

This project is licensed under the MIT License.
```

This guide should help you set up a Django backend with a React frontend, including user authentication and CRUD operations for a Todo application. Each step includes explanations of why we use certain tools and practices to ensure a comprehensive understanding of the setup process.
