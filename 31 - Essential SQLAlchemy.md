# Chapter 11. Getting Started with Alembic

Alembic provides a way for us to programmically create and perform migrations to handle changes to the database that we’ll need to make as our application evolves. For example, we might add columns to our tables or remove attributes from our models. We might also add entirely new models, or split an existing model into multiple models. Alembic provides a way for us to preform these types of changes by leveraging the power of SQLAlchemy.

To get started, we need to install Alembic, which we can do with the following:

```python
pip install alembic
```

Once we have Alembic installed, we need to create the migration environment.

## Creating the Migration Environment

To create the migration environment, we are going to create a folder labeled CH12, and change into that directory. Next, run the alembic init alembic command to create our migration environment in the alembic/ directory. People commonly create the migration environment in a migrations/ directory, which you can do with alembic init migrations. You can choose whatever directory name you like, but I encourage you to name it something distinctive that won’t be used as a module name in your code anywhere. This initialization process creates the migration environment and also creates an alembic.ini file with the configuration options. If we look at our directory now, we’ll see the following structure:

```python
.
├── alembic
│   ├── README
│   ├── env.py
│   ├── script.py.mako
│   └── versions
└── alembic.ini
```

Inside our newly created migration environment, we’ll find env.py and the script.py.mako template file along with a versions/ directory. The versions/ directory will hold our migration scripts. The env.py file is used by Alembic to define and instantiate a SQLAlchemy engine, connect to that engine, and start a transaction, and calls the migration engine properly when you run an Alembic command. The script.py.mako template is used when creating a migration, and it defines the basic structure of a migration.

With the environment created, it’s time to configure it to work with our application.

## Configuring the Migration Environment

The settings in the alembic.ini and env.py files need to be tweaked so that Alembic can work with our database and application. Let’s start with the alembic.ini file where we need to change the sqlalchemy.url option to match our database connection string. We want to set it to connect to a SQLite file named alembictest.db in the current directory; to do that, edit the sqlalchemy.url line to look like the following:

```python
sqlalchemy.url = sqlite:///alembictest.db
```

In all our Alembic examples, we will be creating all our code in an app/db.py file using the declarative style of the ORM. First, create an app/ directory and an empty __init__.py in it to make app a module.

Then we’ll add the following code into app/db.py to set up SQLAlchemy to use the same database that Alembic is configured to use in the alembic.ini file and define a declarative base:

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///alembictest.db')

Base = declarative_base()
```

Now we need to change the env.py file to point to our metadata, which is an attribute of the Base instance we created in app/db.py. It uses the metadata to compare what it finds in the database to the models defined in SQLAlchemy. We’ll start by adding the current directory to the path that Python uses for locating modules so that it can see our app module:

```python
import os
import sys

sys.path.append(os.getcwd())
```

* Adds the current working directory (CH12) to the sys.path that Python uses when searching for modules

Finally, we’ll change the target metadata line in env.py to match our metadata object in the app/db.py file:

```python
from app.db import Base
target_metadata = Base.metadata
```

* Import our instance of Base.
* Instruct Alembic to use the Base.metadata as its target.

With this completed, we have our Alembic environment set up properly to use the application’s database and metadata, and we have built a skeleton of an application where we will define our data models in Chapter 12.

In this chapter, we learned how to create a migration environment and configure it to share the same database and metadata as our application. In the next chapter, we will begin building migrations both by hand and using the autogeneration capabilities.




