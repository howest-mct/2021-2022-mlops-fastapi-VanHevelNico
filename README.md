# 01_FastAPI

## Watch the introduction lesson on Leho
The theoretical session is recorded and put on Leho for you to find.

## Installation
- Clone this repository from GitHub Classrooms.
- Create a new folder called `api`.
- Create a `.env` file in the root of your directory, containing real values based on the `.env.example`.

- Create a new virtual environment on your PC / VM, where you will run your Python code.
`conda create -n 01_fastapi` if you use Conda.
- Activate the conda environment in a terminal.
`conda activate 01_fastapi`.
- `pip install -r api/requirements.dev.txt`

**You do not need to install it in a seperate virtual environment, but take the consequences then!**

**I have tested all of this in a fresh VM, where I only installed pip: `sudo apt-get install python3-pip`.**

## Setup the project
- Add a `main.py` script in your `api > app` folder. Place this inside.
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

Make sure you are inside your `api > app` folder, then execute `python3 -m uvicorn main:app --reload`. This will launch uvicorn, and makes sure it auto-reloads on any change you make. If you are working with Visual Studio Code, you will get a message that there is a port running, and you can open this in the browser.
VSCode also port-forwards any ports running in a VM.

Visit your application on `127.0.0.1:8000`. You will get a default `{"message":"Hello World"}` if everything goes well.

The cool thing about FastAPI is that it automatically enables and runs a Swagger API as well. You can visit it at the `/docs` page. Check it out!

## Adding our first routes

This API is all about Bird spotting, a small part of the project will be re-used by other students working on a similar project in other courses. 

We will add a little bit of structure to our project to get started.

### Schemas

- Create a `schemas` directory under `app`. Add two empty Python files: `bird.py` and `user.py`. Here we will create `pydantic` models.

```python
# bird.py
from typing import Optional
from pydantic import BaseModel

class Bird(BaseModel):
    uuid: Optional[str]
    id: str
    name: str
    short: str
    image: str
    recon: list
    food: dict
    see: str
```

- Create the `user.py` schema yourself. A User must have a `uuid`, name, locationOfResidence, age, gender and a registrationDate.

- Also make sure there is a method called `sayHello()` on the class which will say something base on the name of the bird or user.

---
### Adding routes
- In `main.py` add a few birds and a user. You can use the JSON file of the birds to read all of them in. *We can later use this to seed the database.*

    - Just read in the JSON file and load it in a variable `birds`

- Add a route to GET the list of birds
- Add a route to GET the list of users.

**Think whether or not you need to have an async method or not...**

**Use the FastAPI documentation to add the following routes**

- Add a route to POST a user to the list.
- Add a route to GET one bird based on it's ID.

> TIP: To convert a Python dictionary to a  Pydantic model, you can use `Bird(**birdDict)`.
> This will allow you to access their properties as well.
> 
> Try it out:
> ```python
> from schemas.bird import Bird
> vink = Bird(**birds[0])
> vink.sayHello()
> ```

---
### Adding a little bit more structure

- Create a new folder called `routers` in the `app` folder.
- Inside, we will create a `bird_router.py` and `user_router.py` file.

In these files, we will create a new `APIRouter`, like this.

```python
router = APIRouter(
    prefix="/birds",
    tags=["Bird"],
    responses={404: {"Bird": "Not found"}},
)
```

Now we can use this `router` instead of the `app` from the `main` app.

- Copy all the content related to the Birds and Users into their own files, and convert `@app.get()` to `@router.get()`.

Note as well that we have included a `prefix="/birds"` which means that all our API routes inside this router will be prefixed with `/birds`. Convert the routes to reflect your changes.

Now that you have moved them in their own files, make sure to import them from the `main` app.

```python
from routers import (
    bird_router as bird, # Just to make an alias, because it looks nicer.
    user_router as user
)

app.include_router(bird.router)
app.include_router(user.router)
```
---
---
## Connect a database

Currently, our application already has a nice, expendable structure. Which is great for future additions!

However, it is time to add some databases, so that we can start working with relational data and more!

FastAPI has a package called `SQLAlchemy` included into their app. Together with this package and `Pydantic` we can easily create a database connection.

But first, we'll have to get the database set up.

### Remember Docker?
We will quickly spin up a database, just like we did last year with Backend development. You'll have to search for a few things tho. I won't give you everything.

- Add a `docker-compose.yml` file inside the root of your project.
- Add a `mariadb` service, with the `mariadb:10.5.9` image.
    - Make sure you port-forward port `3306` which is the default mysql port.
    - Provide a persistent storage in some way. Find out [from the documentation](https://hub.docker.com/_/mysql) where the mysql container keeps it's data.
        > **ANSWER**  
        > Fill in here: `...` 
    - Add `.env` file with the configuration you need for your database. The `MYSQL_HOST` environment variable will be different for a Docker environment and a localhost environment, remember from last year?
    Use `localhost` when you are running it local.

- You can add an `adminer` service with one of it's image on Docker Hub. This app serves as an in-browser database viewer, so that you do not need any other clients installed. Find out the default port it runs on. You can remap that to another port if you want. I chose 9999.
    > **ANSWER** "How did you set up the docker-compose service for adminer? Show the YAML here
    > ```yaml
    > # Answer here
    > ```

- Start the Docker Compose services and wait for everything to start up. Use the Visual Studio Code 'Docker' plugin to view your running containers.

- Go to your adminer service, and log in to your database with the information you provided in the `.env` file.

- Check to see if the database you chose in the `.env` file is created. We will not see any tables yet. That's the next step

### Let's connect from Python now.

As we are going to be developping our application code-first, we will create the tables from our Python code. You can also choose to work model-first, but then you'll still need to write your Python code anyways.

- Next to the `main.py` add a `database.py` file, which will load in our database connection.

```python
# database.py

import os
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# This will get our environment variables, or some fallback values. But remember that these won't work as the .env file was included for the database
MYSQL_USER = os.getenv('MYSQL_USER', 'admin')
MYSQL_HOST = os.getenv('MYSQL_HOST', 'mariadb')
MYSQL_PORT = os.getenv('MYSQL_PORT', '3306')
MYSQL_DATABASE = os.getenv('MYSQL_DATABASE', 'default_db')
MYSQL_PASSWORD = os.getenv('MYSQL_PASSWORD', 'mypassword')

engine = create_engine(
    f'mysql+pymysql://{MYSQL_USER}:{MYSQL_PASSWORD}@{MYSQL_HOST}:{MYSQL_PORT}/{MYSQL_DATABASE}'
)
session = sessionmaker(autoflush=False, bind=engine) 

db = session()
Base = declarative_base()

def start_db():
    Base.metadata.create_all(engine)
```

Before we add link this code into our `main.py` file, we will have to inject our `.env` values.

- Add these lines before getting the `os.getenv()` values.
```python
from dotenv import load_dotenv

load_dotenv() # Make sure we have our .env values
```

- You can now go and start the database connection on the `main.py`
```python
import database as db
db.start_db()
```
If you do not have any errors, you can go on. Otherwise, you'll have to fix them first.

### Creating tables
As I mentioned before, the tables will be created by our Python code itself. So-called **code-first**.

For this, we will add a folder called `models`. This will contain our database models. Not to be confused with the `schemas` which were Pydantic Schema's.

- Add a `bird_model.py` and `user_model.py` file.

The basics we need to import for **SQLAlchemy** to notice that we want a table, is the `Base` object which was defined on line 23 of `database.py`:  
`Base = declarative_base()`  
We can use this **Class** as the base for a new class. This will register a new table.

```python
from database import Base
class Bird(Base):
    __tablename__ = 'birds'
```

To add the different columns, use the [documentation of SQLAlchemy](https://docs.sqlalchemy.org/en/14/orm/tutorial.html) to get what you want.

> **TIP**  
> If you want to auto-generate a UUID, use the following method
> ```python
> def generate_uuid():
>    return str(uuid.uuid4())
> ```

- In case you want to add a simple **list** or **dict** inside a column, you can do it like this:
```python

import json
import sqlalchemy
from sqlalchemy.types import TypeDecorator

SIZE = 5120

class TextPickleType(TypeDecorator):

    impl = sqlalchemy.Text(SIZE)

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value

recon = Column(TextPickleType()) # This will place a list inside one cell
food = Column(TextPickleType()) # This will place a dict inside one cell
```

- Import your `bird_model` inside your `main.py`, start your application and check if your table has been created.

- Do the same for the User table.

> **Answer** "Paste your Bird and User Classes here"
> ```python
> # Bird.py
> 
> # User.py
> 
> ```

### Updating routes
As we now have a database attached to our application, we can start writing new routes to use this database.

The first one will make sure we can add a new Bird to our database, and later on we will query them.

- Update your Bird model to import the db connector `from database import db`
- Add a property to your Bird model: `self.model = Bird`, so that we can refer to this class in that way.
- Import your Bird Schema into your model file: `from schemas.bird import Bird as BirdSchema`
- Add a property to your Bird model: `self.schema = BirdSchema`

- Register your Bird Schema `schemas > bird.py` as an ORM-ready object:
    ```python
    class Bird(BaseModel):
        uuid: Optional[str]
        id: str
        name: str
        short: str
        image: str
        recon: list
        food: dict
        see: str

        class Config:
            orm_mode = True

        def sayHello(self):
            print(f"{self.name} is flying by.")
    ```

- Add a first query in a method called `get_all()`
```python
def get_all(self):
    try:
        db_objects = db.query(self.model).all() # The actual query
        if db_objects:
            return db_objects
        else:
            print(f"No {self.model} was found!")
            return None
    except Exception as e:
        print(f"Error while getting all {self.model}s.")
        print(e)
        db.rollback()
```

- Add a second query in a method called `create()`
```python
def create(self, obj: BirdSchema):
    try:
        obj_in_db = self.get_by(name=obj.name)
        if obj_in_db is None:
            print(f"No {self.model} was found with name {obj.name}!")

            new_obj = self.model(**obj.dict())
            db.add(new_obj)
            db.commit()

            print(f"{self.model} has been added to the database!")
            obj = self.schema.from_orm(new_obj)
        else:
            obj = None
            print(f"A {self.model} already exists.")

        return obj

    except Exception as e:
        print(f"Error while creating {self.model}.")
        print("Rolling back the database commit.")
        print(e)
        db.rollback()
```

- Copy the previous method to add other interesting queries: **Delete a bird**, **Update a bird** **Get one bird based on ID**, **Get many birds based on a property**.

> **ANSWER** "Paste two of your queries below"
> ```python
> # Query 1: UPDATE | DELETE | GET ONE | GET MANY (Select which you chose)
>
> # Query 2: UPDATE | DELETE | GET ONE | GET MANY (Select which you chose)
>
> ```

- Now that we have to queries, we can start adding them to our routes.
    - Put the previous routes of our Bird Router in a comment, so we still have it, but deactivated.
    - Import our ORM Model `from models.bird_model import Bird as BirdRepo`. Let's call it a Repo for the sake of logic with Entity Framework. Initialize the repo before you can use it (The methods are not static!) `repo = BirdRepo()`.
    - Now go ahead and write all the logic, like this: `objects = repo.get_all()`

- To add proper exceptions, import `from fastapi import HTTPException` and edit your routes to raise the error:
    ```python
    if objects is None:
        raise HTTPException(status_code=400, detail="Something went wrong here!")
    ```
- To allow Swagger to give us the right API hints, please give your Pydantic scheme as a typehint in your router methods
    ```python
    @router.post("")
    def postBird(bird: Bird):
        pass
    ```

- Now test if you can add a bird to your database. Use the birds from your JSON file.

### Optional: Feel free to continue with relations
Our database is far from finished. To be good, we should add some more models to it, in order to use our relational database to its full potential.

**You are not required to do these adaptions for the rest of the course, but it could help you in working with FastAPI in the future**.

You could:
- Add an **Observation** model with an `observationDate`, `observationLocation`, `birdsObserved` (which has a relation with birds), and `user` (which has a relation with users)
- Add the observations as a list to your user model
- One observation could contain multiple bird species. I could see 3 doves and an eagle in one observation, at one specific location, so adapt your model for that.


---
---

# Docker deployment

We are working from a Docker [image that was created by the makers of FastAPI](https://hub.docker.com/r/tiangolo/uvicorn-gunicorn-fastapi/).
Note the details they write there:

> It will expect a file at /app/app/main.py.
>
> Or otherwise a file at /app/main.py.
>
> And will expect it to contain a variable app with your FastAPI application.

### Database checks
> It could be that our database is not fully set-up before our application is started.
> Of course, we could use the `depends_on` option in Docker, to allow our database to start up first, but that doesn't check if the database was really booted already. It only checks whether the container is start up.
> To work around this problem, we can use a `prestart` script, which runs before our main app, and waits until our database is active.

- Add a `backend_pre_start.py` Python script in the root of your `app`. Paste this starter inside

```python
from database import db
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
from tenacity import after_log, before_log, retry, stop_after_attempt, wait_fixed

max_tries = 60 * 5  # 5 minutes
wait_seconds = 1

@retry(
    stop=stop_after_attempt(max_tries),
    wait=wait_fixed(wait_seconds),
    before=before_log(logger, logging.INFO),
    after=after_log(logger, logging.WARN),
)
def init() -> None:
    try:
        # Try to create session to check if DB is awake
        db.execute("SELECT 1")
    except Exception as e:
        logger.error(e)
        raise e

def main() -> None:
    logger.info("Initializing service")
    init()
    logger.info("Service finished initializing")


if __name__ == "__main__":
    main()
```

**NOTE:** If you want to, you could write a seeder script and add it to this file as well.

### Dockerfile

The Dockerfile that I gave you in the initial repo contains everything we need. Just try to understand what goes on, if you want you can adapt it for more performance. 

A few things to note:
- We included an `app/prestart.sh` script to run a few things before launching. This is optional, but we use it to check for our database. We do not need to register it anywhere, the FastAPI docker image knows how to use it. It should contain this:
    ```bash
    #!/bin/bash
    # Let the DB start
    python /app/backend_pre_start.py
    ```
- If you are working on Windows to build your image, we need to use `dos2unix` to convert our `app/prestart.sh` file and make it executable in Linux (/The container). We included it to be safe.
- Why are we working with a multi-stage Docker build?
    > **ANSWER**  
    > ...

### Docker-compose
To make our life easier (because it's already hard enough as it is ...), we will use Docker Compose once again!
- Add a service to the already existing `docker-compose.yml`.
    - Name the service `api`
    - Add the build option
        ```yaml
        build:
            context: api
            dockerfile: Dockerfile
        ```
    - Make it wait for the database
    - Add the `.env` file.
    - Port forward to a port you want

- Make sure to change the `.env` value of the `MYSQL_HOST` to suit the Docker environment.
- Also uncomment the `birds.json` line in the `bird_router.py` if that is still there.

Start your docker-compose services, and test it out.
```python
# Just some fun
if thingsWentWrong: 
    fixProblems()
elif:
    print("Enjoy your day!")
```
# Further adaptions

Think about further adaptions to this FastAPI which could improve your codebase.
Try to make it more generic, so you can easily re-use this boilerplate for other projects.

- Add a Seeder
- Add a generic CRUD Router, which you can easily use to inherit by other router classes.
- Add a frontend to query your API.
- Add some more routes if you want
- Nicer exception handling and so on
- Uploading images
- ...

# What did you learn?

Fill in something that you learned during this lesson

> ...  
> ...

## Give three interesting exam questions

1) ...
2) ...
3) ...