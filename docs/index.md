# Home

## Getting Started

This documents will have information about working with smart-digi-build application. and it's deployment on ubuntu server.


## User Roles

The application have four user roles, and multiple user groups which user can create based on it's requirement.

* Super admin  - sa
* Customer admin - admin
* Facility admin - facility
* User - user

## Structure

The application developed with API First approach, and it's have two parts.

* Backend - API
* Frontend - Web

## Backend

The backend is developed with FastAPI Rest Framework.

```
PYTHON 3.11.3
```

## Frontend

The frontend is developed with ReactJS.

```
TBD
```

## Database

The database is developed using MongoDB.

```
MongoDB version 6.0.8
```

## Login 

The below login screen is common for all the users, Redirect based on user role handled on frontend.

<figure markdown>
  ![Login](./images/login.png){ width="800" }
  <figcaption>Login</figcaption>
</figure>
