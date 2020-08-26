# Chapter 14. Cookbook

This chapter is different than previous ones in that each section focuses on a different aspect of using SQLAlchemy. The coverage here isn’t as detailed as in previous chapters; consider this a quick rundown of useful tools. At the end of each section, there’s information about where you can learn more if you’re interested. These sections are not meant to be full tutorials, but rather short recipes on how to accomplish a particular task. The first part of this chapter focuses on a few advanced usages of SQLAlchemy, the second part on using SQLAlchemy with web frameworks like Flask, and the final section on additional libraries that can be used with SQLAlchemy.

## Hybrid Attributes

Hybrid attributes are those that exhibit one behavior when accessed as a class method, and another behavior when accessed on an instance. Another way to think of this is that the attribute will generate valid SQL when it is used in a SQLAlchemy statement, and when accessed on an instance the hybrid attribute will execute the Python code directly against the instance. I find it easiest to understand this when looking at code. We’re going to use our Cookie declarative class to illustrate this in Example 14-1.


Example 14-1. Our Cookie user data model
```python
from datetime import datetime

from sqlalchemy import Column, Integer, Numeric, String, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.hybrid import hybrid_property, hybrid_method
from sqlalchemy.orm import sessionmaker


engine = create_engine('sqlite:///:memory:')

Base = declarative_base()


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer, primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))

    @hybrid_property 1
    def inventory_value(self):
        return self.unit_cost * self.quantity

    @hybrid_method 2
    def bake_more(self, min_quantity):
        return self.quantity < min_quantity

    def __repr__(self):
        return "Cookie(cookie_name='{self.cookie_name}', " \
            "cookie_recipe_url='{self.cookie_recipe_url}', " \
            "cookie_sku='{self.cookie_sku}', " \
            "quantity={self.quantity}, " \
            "unit_cost={self.unit_cost})".format(self=self)


Base.metadata.create_all(engine)
Session = sessionmaker(bind=engine)
```

* Creates a hybrid property.
* Creates a hybrid method because we need an additional input to perform the comparison.

In Example 14-1, inventory_value is a hybrid property that performs a calculation with two attributes of our Cookie data class, and bake_more is a hybrid method that takes an additional input to determine if we should bake more cookies. (Should this ever return false? There’s always room for more cookies!) With our data class defined, we can begin to examine what hybrid really means. Let’s look at how the inventory_value works when used in a query (Example 14-2).

Example 14-2. Hybrid property: Class
```python
print(Cookie.inventory_value < 10.00)
```

Example 14-2 will output the following:

```python
cookies.unit_cost * cookies.quantity < :param_1
```

In the output from Example 14-2, we can see that the hybrid property was expanded into a valid SQL clause. This means that we can filter, order by, group by, and use database functions on these properties. Let’s do the same thing with the bake_more hybrid method:

```python
print(Cookie.bake_more(12))
```

This will output:
```python
cookies.quantity < :quantity_1
```

Again, it builds a valid SQL clause out of the Python code in our hybrid method. In order to see what happens when we access it as an instance method, we’ll need to add some data to database, which we’ll do in Example 14-3.

Example 14-3. Adding some records to the database
```python
session = Session()
cc_cookie = Cookie(cookie_name='chocolate chip',
                   cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
                   cookie_sku='CC01',
                   quantity=12,
                   unit_cost=0.50)
dcc = Cookie(cookie_name='dark chocolate chip',
             cookie_recipe_url='http://some.aweso.me/cookie/recipe_dark.html',
             cookie_sku='CC02',
             quantity=1,
             unit_cost=0.75)
mol = Cookie(cookie_name='molasses',
             cookie_recipe_url='http://some.aweso.me/cookie/recipe_molasses.html',
             cookie_sku='MOL01',
             quantity=1,
             unit_cost=0.80)
session.add(cc_cookie)
session.add(dcc)
session.add(mol)
session.flush()
```

With this data added, we’re ready to look at what happens when we use our hybrid property and method on an instance. We can access the inventory_value property of the dcc instance as shown here:

```python
dcc.inventory_value
0.75
```
You can see that when used on an instance, inventory_value executes the Python code specified in the property. This is exactly the magic of a hybrid property or method. We can also run the bake_more hybrid method on the instance, as shown here:

```python
dcc.bake_more(12)
True
```

Again, because of the hybrid method’s behavior when accessed on an instance, bake_more executes the Python code as expected. Now that we understand how the hybrid properties and methods work, let’s use them in some queries. We’ll start by using inventory_value in Example 14-4.

Example 14-4. Using a hybrid property in a query
```python
from sqlalchemy import desc

for cookie in session.query(Cookie).order_by(desc(Cookie.inventory_value)):
    print('{:>20} - {:.2f}'.format(cookie.cookie_name, cookie.inventory_value))
```

When we run Example 14-4, we get the following output:

```python
    chocolate chip - 6.00
    molasses - 0.80
    dark chocolate chip - 0.75
```
This behaves just as we would expect any ORM class attribute to work. In Example 14-5, we will use the bake_more method to decide what cookies we need to bake.

Example 14-5. Using a hybrid method in a query
```python
for cookie in session.query(Cookie).filter(Cookie.bake_more(12)):
    print('{:>20} - {}'.format(cookie.cookie_name, cookie.quantity))
```
Example 14-5 outputs:

```python
dark chocolate chip - 1
           molasses - 1
```
Again, this behaves just like we hoped it would. While this is an amazing set of behaviors from our code, hybrid attributes can do so much more, and you can read about it in the SQLAlchemy documentation on hybrid attributes.

## Association Proxy

An association proxy is a pointer across a relationship to a specific attribute; it can be used to make it easier to access an attribute across a relationship in code. For example, this would come in handy if we wanted a list of ingredient names that are used to make our cookies. Let’s walk through how that works with a typical relationship. The relationship between cookies and ingredients will be a many-to-many relationship. Example 14-6 sets up our data models and their relationships.

Example 14-6. Setting up our models and relationships
```python
from datetime import datetime
from sqlalchemy import (Column, Integer, Numeric, String, Table,
                        ForeignKey, create_engine)
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///:memory:')

Base = declarative_base()
Session = sessionmaker(bind=engine)



cookieingredients_table = Table('cookieingredients', Base.metadata, 1
    Column('cookie_id', Integer, ForeignKey("cookies.cookie_id"),
           primary_key=True),
    Column('ingredient_id', Integer, ForeignKey("ingredients.ingredient_id"),
           primary_key=True)
)


class Ingredient(Base):
    __tablename__ = 'ingredients'

    ingredient_id = Column(Integer, primary_key=True)
    name = Column(String(255), index=True)

    def __repr__(self):
        return "Ingredient(name='{self.name}')".format(self=self)


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer, primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))

    ingredients = relationship("Ingredient",
                                secondary=cookieingredients_table) 2

    def __repr__(self):
        return "Cookie(cookie_name='{self.cookie_name}', " \
                       "cookie_recipe_url='{self.cookie_recipe_url}', " \
                       "cookie_sku='{self.cookie_sku}', " \
                       "quantity={self.quantity}, " \
                       "unit_cost={self.unit_cost})".format(self=self)


Base.metadata.create_all(engine)
```

* Creating the through table used for our many-to-many relationship.
* Creating the relationship to ingredients via the through table.

Because this is a many-to-many relationship, we need a through table to map the multiple relationships. We use the cookieingredients_table to do that. Next, we need to add a cookie and some ingredients, as shown here:

```python
session = Session()
cc_cookie = Cookie(cookie_name='chocolate chip', 1
                   cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
                   cookie_sku='CC01',
                   quantity=12,
                   unit_cost=0.50)

flour = Ingredient(name='Flour') 2
sugar = Ingredient(name='Sugar')
egg = Ingredient(name='Egg')
cc = Ingredient(name='Chocolate Chips')
cc_cookie.ingredients.extend([flour, sugar, egg, cc]) 3
session.add(cc_cookie)
session.flush()
```
* Creating the cookie.
* Creating the ingredients.
* Adding the ingredients to the relationship with the cookie.

So to add the ingredients to the cookie, we have to create them and then add them to the relationship on cookies, ingredients. Now, if we want to list the names of all the ingredients, which was our original goal, we have to iterate through all the ingredients and get the name attribute. We can accomplish this as follows:


```python
[ingredient.name for ingredient in cc_cookie.ingredients]
```

This will return:

```python
['Flour', 'Sugar', 'Egg', 'Chocolate Chips']
```

This can be cumbersome if all we really want from ingredients is the name attribute. Using a traditional relationship also requires us to manually create each ingredient and add it to the cookie. It becomes more work if we need to determine if the ingredient already exists. This is where the association proxy can be useful in simplifying this type of usage.

To establish an association proxy that we can use for attribute access and ingredient creation, we need to do three things:

* Import the association proxy.
* Add an __init__ method to the targeted object that makes it easy to create new instances with just the required values.
* Create an association proxy that targets the table name and column name you want to proxy.

We’re going to start a fresh Python shell, and set up our classes again. However, this time we are going to add an association proxy to give us easier access to the ingredient names in Example 14-7.

Example 14-7. Association proxy setup

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///:memory:')

Session = sessionmaker(bind=engine)

from datetime import datetime

from sqlalchemy import Column, Integer, Numeric, String, Table, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.associationproxy import association_proxy 1


Base = declarative_base()


cookieingredients_table = Table('cookieingredients', Base.metadata,
    Column('cookie_id', Integer, ForeignKey("cookies.cookie_id"),
           primary_key=True),
    Column('ingredient_id', Integer, ForeignKey("ingredients.ingredient_id"),
           primary_key=True)
)


class Ingredient(Base):
    __tablename__ = 'ingredients'

    ingredient_id = Column(Integer, primary_key=True)
    name = Column(String(255), index=True)

    def __init__(self, name): 2
        self.name = name


    def __repr__(self):
        return "Ingredient(name='{self.name}')".format(self=self)


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer, primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))

    ingredients = relationship("Ingredient",
                               secondary=cookieingredients_table)

    ingredient_names = association_proxy('ingredients', 'name') 3


    def __repr__(self):
        return "Cookie(cookie_name='{self.cookie_name}', " \
                       "cookie_recipe_url='{self.cookie_recipe_url}', " \
                       "cookie_sku='{self.cookie_sku}', " \
                       "quantity={self.quantity}, " \
                       "unit_cost={self.unit_cost})".format(self=self)


Base.metadata.create_all(engine)
```

* Importing the association proxy.
* Defining an __init__ method that only requires a name.
* Establishing an association proxy to the ingredients’ name attribute that we can reference as ingredient_names.

With these three things done, we can now use our classes as we did previously:

```python
session = Session()
cc_cookie = Cookie(cookie_name='chocolate chip',
                   cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
                   cookie_sku='CC01',
                   quantity=12,
                   unit_cost=0.50)
dcc = Cookie(cookie_name='dark chocolate chip',
             cookie_recipe_url='http://some.aweso.me/cookie/recipe_dark.html',
             cookie_sku='CC02',
             quantity=1,
             unit_cost=0.75)
flour = Ingredient(name='Flour')
sugar = Ingredient(name='Sugar')
egg = Ingredient(name='Egg')
cc = Ingredient(name='Chocolate Chips')
cc_cookie.ingredients.extend([flour, sugar, egg, cc])
session.add(cc_cookie)
session.add(dcc)
session.flush()
```

So the relationship still works like we expect; however, to get the ingredient names we can use the association proxy to avoid the list comprehension we used before. Here is an example of using ingrendient_names:

```python
cc_cookie.ingredient_names
```

This will output:

```python
['Flour', 'Sugar', 'Egg', 'Chocolate Chips']
```

This is much easier than looping though all the ingredients to create the list of ingredient names. However, we forgot to add one key ingredient, “Oil”. We can add this new ingredient using the association proxy as shown here:

```python
cc_cookie.ingredient_names.append('Oil')
session.flush()
```

When we do this, the association proxy creates a new ingredient using the Ingredient.__init__ method for us automatically. It’s important to note that if we already had an ingredient named Oil, the association proxy would still attempt to create it for us. That could lead to duplicate records or cause an exception if the addition violates a constraint.

To work around this, we can query for existing ingredients prior to using the association proxy. Example 14-8 shows how we can accomplish that.

Example 14-8. Filtering ingredients
```python
dcc_ingredient_list = ['Flour', 'Sugar', 'Egg', 'Dark Chocolate Chips',
                       'Oil'] 1
existing_ingredients = session.query(Ingredient).filter(
    Ingredient.name.in_(dcc_ingredient_list)).all() 2
missing = set(dcc_ingredient_list) - set([x.name for x in
                                          existing_ingredients]) 3
```

* Defining the list of ingredients for our dark chocolate chip cookies.
* Querying to find the ingredients that already exist in our database.
* Finding the missing packages by using the difference of the two sets of ingredients.

After finding the ingredients we already had, we then compared it with our required ingredient list to determine what was missing, which is “Dark Chocolate Chips.” Now we can add all the existing ingredients via the relationship, and then the new ingredients via the association proxy as shown in Example 14-9.

Example 14-9. Adding ingredients to dark chocolate chip cookie
```python
dcc.ingredients.extend(existing_ingredients) 1
dcc.ingredient_names.extend(missing) 2
```
* Adding the existing ingredients via the relationship.
* Adding the new ingredients via the association proxy.

Now we can print a list of the ingredient names, as shown here:

```python
dcc.ingredient_names
```

And we will get the following output:

```python
['Egg', 'Flour', 'Oil', 'Sugar', 'Dark Chocolate Chips']
```

This enabled us to quickly handle existing and new ingredients when we added them to our cookie, and the resulting output was just what we desired. Association proxies have lots of other uses, too; you can learn more in the association proxy documentation.

## Integrating SQLAlchemy with Flask

It’s common to see SQLAlchemy used with a Flask web application. The creator of Flask has also created a Flask-SQLalchemy package to make this integration easy. Using Flask-SQLalchemy will provide preconfigured scoped sessions that are tied to the page life cycle of your Flask application. You can install Flask-SQLalchemy with pip as shown here:

```python
# pip install flask-sqlalchemy
```

When using Flask-SQLalchemy, I highly recommend you use the app factory pattern, which is not what is shown in the quick start section of the Flask-SQLalchemy documentation. The app factory pattern uses a function that assembles an application with all the appropriate add-ons and configuration. This is typically placed in your application’s app/__init__.py file. Example 14-10 contains an example of how to structure your create_app method.

Example 14-10. App factory function
```python
from flask import Flask
from flask.ext.sqlalchemy import SQLAlchemy

from config import config

db = SQLAlchemy() 1


def create_app(config_name): 2
    app = Flask(__name__)
    app.config.from_object(config[config_name])

    db.init_app(app) 3
    return app
```
* Creates an unconfigured Flask-SQLalchemy instance.
* Defines the create_app app factory.
* Initializes the instance with the app context.

The app factory needs a configuration that defines the Flask settings and the SQLAlchemy connection string. In Example 14-11, we define a configuration file, config.py, in the root of the application.

Example 14-11. Flask app configuration
```python
import os


basedir = os.path.abspath(os.path.dirname(__file__))


class Config: 1
    SECRET_KEY = 'development key'
    ADMINS = frozenset(['jason@jasonamyers.com', ])


class DevelopmentConfig(Config): 2
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = "sqlite:////tmp/dev.db"


class ProductionConfig(Config): 3
    SECRET_KEY = 'Prod key'
    SQLALCHEMY_DATABASE_URI = "sqlite:////tmp/prod.db"


config = { 4
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'default': DevelopmentConfig
}
```
* Defines the base Flask config.
* Defines the configuration used when developing.
* Defines the configuration used when in production.
* Creates a dictionary that is used to map from a simple name to the configuration class.

With the app factory and configuration ready, we have a functioning Flask application. We’re now ready to define our data classes. We’ll define our Cookie model in apps/models.py, as demonstrated in Example 14-12.

Example 14-12. Defining the Cookie model
```python
from app import db


class Cookie(db.Model):
    __tablename__ = 'cookies'

    cookie_id = db.Column(db.Integer(), primary_key=True)
    cookie_name = db.Column(db.String(50), index=True)
    cookie_recipe_url = db.Column(db.String(255))
    quantity = db.Column(db.Integer())
```

In Example 14-12, we used the db instance we created in app/__init__.py to access most parts of the SQLAlchemy library. Now we can use our cookie models in queries just like we did in the ORM section of the book. The only change is that our sessions are nested in the db.session object.

One other thing that Flask-SQLalchemy adds that is not found in normal SQLAlchemy code is the addition of a query method to every ORM data class. I highly encourage you to not use this syntax, because it can be confusing when mixed and matched with other SQLAlchemy queries. This style of query looks like the following:

```python
Cookie.query.all()
```

This should give you a taste of how to integrate SQLAlchemy with Flask; you can learn more in the Flask-SQLalchemy documentation. Miguel Grinberg also wrote a fantastic book on building out a whole application titled Flask Web Development.

## SQLAcodegen

We learned about reflection with SQLAlchemy Core in Chapter 5 and automap for use with the ORM in Chapter 10. While both of those solutions are great, they require you to perform the reflection every time the application is restarted, or in the case of dynamic modules, every time the module is reloaded. SQLAcodegen uses reflection to build a collection of ORM data classes that you can use in your application code base to avoid reflecting the database multiple times. SQLAcodegen has the ability to detect many-to-one, one-to-one, and many-to-many relationships. It can be installed via pip:

```python
pip install sqlacodegen
```

To run SQLAcodegen, we need to specify a database connection string for it to connect to. We’ll use a copy of the Chinook database that we have worked with previously (available in the CH15/ folder of the example code from this book). While inside the CH15/ folder, Run the SQLAcodegen command with the appropriate connection string, as shown in Example 14-13.

Example 14-13. Running SQLAcodegen on the Chinook database
```python
# sqlacodegen sqlite:///Chinook_Sqlite.sqlite 1

# coding: utf-8
from sqlalchemy import (Table, Column, Integer, Numeric, Unicode, DateTime, 
                        ForeignKey)
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()
metadata = Base.metadata


class Album(Base):
    __tablename__ = 'Album'

    AlbumId = Column(Integer, primary_key=True, unique=True)
    Title = Column(Unicode(160), nullable=False)
    ArtistId = Column(ForeignKey(u'Artist.ArtistId'), nullable=False, index=True)

    Artist = relationship(u'Artist')


class Artist(Base):
    __tablename__ = 'Artist'

    ArtistId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))


class Customer(Base):
    __tablename__ = 'Customer'

    CustomerId = Column(Integer, primary_key=True, unique=True)
    FirstName = Column(Unicode(40), nullable=False)
    LastName = Column(Unicode(20), nullable=False)
    Company = Column(Unicode(80))
    Address = Column(Unicode(70))
    City = Column(Unicode(40))
    State = Column(Unicode(40))
    Country = Column(Unicode(40))
    PostalCode = Column(Unicode(10))
    Phone = Column(Unicode(24))
    Fax = Column(Unicode(24))
    Email = Column(Unicode(60), nullable=False)
    SupportRepId = Column(ForeignKey(u'Employee.EmployeeId'), index=True)

    Employee = relationship(u'Employee')

class Employee(Base):
    __tablename__ = 'Employee'

    EmployeeId = Column(Integer, primary_key=True, unique=True)
    LastName = Column(Unicode(20), nullable=False)
    FirstName = Column(Unicode(20), nullable=False)
    Title = Column(Unicode(30))
    ReportsTo = Column(ForeignKey(u'Employee.EmployeeId'), index=True)
    BirthDate = Column(DateTime)
    HireDate = Column(DateTime)
    Address = Column(Unicode(70))
    City = Column(Unicode(40))
    State = Column(Unicode(40))
    Country = Column(Unicode(40))
    PostalCode = Column(Unicode(10))
    Phone = Column(Unicode(24))
    Fax = Column(Unicode(24))
    Email = Column(Unicode(60))

    parent = relationship(u'Employee', remote_side=[EmployeeId])


class Genre(Base):
    __tablename__ = 'Genre'

    GenreId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))


class Invoice(Base):
    __tablename__ = 'Invoice'

    InvoiceId = Column(Integer, primary_key=True, unique=True)
    CustomerId = Column(ForeignKey(u'Customer.CustomerId'), nullable=False,
                        index=True)
    InvoiceDate = Column(DateTime, nullable=False)
    BillingAddress = Column(Unicode(70))
    BillingCity = Column(Unicode(40))
    BillingState = Column(Unicode(40))
    BillingCountry = Column(Unicode(40))
    BillingPostalCode = Column(Unicode(10))
    Total = Column(Numeric(10, 2), nullable=False)

    Customer = relationship(u'Customer')


class InvoiceLine(Base):
    __tablename__ = 'InvoiceLine'

    InvoiceLineId = Column(Integer, primary_key=True, unique=True)
    InvoiceId = Column(ForeignKey(u'Invoice.InvoiceId'), nullable=False, index=True)
    TrackId = Column(ForeignKey(u'Track.TrackId'), nullable=False, index=True)
    UnitPrice = Column(Numeric(10, 2), nullable=False)
    Quantity = Column(Integer, nullable=False)

    Invoice = relationship(u'Invoice')
    Track = relationship(u'Track')


class MediaType(Base):
    __tablename__ = 'MediaType'

    MediaTypeId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))


class Playlist(Base):
    __tablename__ = 'Playlist'

    PlaylistId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))

    Track = relationship(u'Track', secondary='PlaylistTrack')


t_PlaylistTrack = Table(
    'PlaylistTrack', metadata,
    Column('PlaylistId', ForeignKey(u'Playlist.PlaylistId'), primary_key=True,
           nullable=False),
    Column('TrackId', ForeignKey(u'Track.TrackId'), primary_key=True, nullable=False,
           index=True),
    Index('IPK_PlaylistTrack', 'PlaylistId', 'TrackId', unique=True)
)


class Track(Base):
    __tablename__ = 'Track'

    TrackId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(200), nullable=False)
    AlbumId = Column(ForeignKey(u'Album.AlbumId'), index=True)
    MediaTypeId = Column(ForeignKey(u'MediaType.MediaTypeId'), nullable=False,
                         index=True)
    GenreId = Column(ForeignKey(u'Genre.GenreId'), index=True)
    Composer = Column(Unicode(220))
    Milliseconds = Column(Integer, nullable=False)
    Bytes = Column(Integer)
    UnitPrice = Column(Numeric(10, 2), nullable=False)

    Album = relationship(u'Album')
    Genre = relationship(u'Genre')
    MediaType = relationship(u'MediaType')
```

* Runs SQLAcodegen against the local Chinook SQLite database

As you can see in Example 14-13, when we run the command, it builds up a complete file that contains all the ORM data classes of the database along with the proper imports. This file is ready for use in our application. You might need to tweak the settings for the Base object if it was established elsewhere. It is also possible to only generate the code for a few tables by using the --tables argument. In Example 14-14, we are going to specify the Artist and Track tables.

Example 14-14. Running SQLAcodegen on the Artist and Track tables
```python
# sqlacodegen sqlite:///Chinook_Sqlite.sqlite --tables Artist,Track 1

# coding: utf-8
from sqlalchemy import Column, ForeignKey, Integer, Numeric, Unicode
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()
metadata = Base.metadata


class Album(Base):
    __tablename__ = 'Album'

    AlbumId = Column(Integer, primary_key=True, unique=True)
    Title = Column(Unicode(160), nullable=False)
    ArtistId = Column(ForeignKey(u'Artist.ArtistId'), nullable=False, index=True)

    Artist = relationship(u'Artist')


class Artist(Base):
    __tablename__ = 'Artist'

    ArtistId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))


class Genre(Base):
    __tablename__ = 'Genre'

    GenreId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))


class MediaType(Base):
    __tablename__ = 'MediaType'

    MediaTypeId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(120))


class Track(Base):
    __tablename__ = 'Track'

    TrackId = Column(Integer, primary_key=True, unique=True)
    Name = Column(Unicode(200), nullable=False)
    AlbumId = Column(ForeignKey(u'Album.AlbumId'), index=True)
    MediaTypeId = Column(ForeignKey(u'MediaType.MediaTypeId'), nullable=False,
                         index=True)
    GenreId = Column(ForeignKey(u'Genre.GenreId'), index=True)
    Composer = Column(Unicode(220))
    Milliseconds = Column(Integer, nullable=False)
    Bytes = Column(Integer)
    UnitPrice = Column(Numeric(10, 2), nullable=False)

    Album = relationship(u'Album')
    Genre = relationship(u'Genre')
    MediaType = relationship(u'MediaType')
```

Runs SQLAcodegen against the local Chinook SQLite database, but only for the Artist and Track tables.

When we specified the Artist and Track tables in Example 14-14, SQLAcodegen built classes for those tables, and all the tables that had a relationship with those tables. SQLAcodegen does this to ensure that the code it builds is ready to be used. If you want to save the generated classes directly to a file, you can do so with standard redirection, as shown here:

```python
# sqlacodegen sqlite:///Chinook_Sqlite.sqlite --tables Artist,Track > db.py
```

If you want more information on what you can do with SQLAcodegen, you can find it by running sqlacodegen --help to see the additional options. There is not any official documentation at the time of this writing.

That concludes the cookbook section of the book. I hope you enjoyed these few samples of how to do more with SQLAlchemy.
















