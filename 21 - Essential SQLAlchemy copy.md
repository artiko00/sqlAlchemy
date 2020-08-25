# Chapter 6. Defining Schema with SQLAlchemy ORM

You define schema slightly different when using the SQLAlchemy ORM because it is focused around user-defined data objects instead of the schema of the underlying database. In SQLAlchemy Core, we created a metadata container and then declared a Table object associated with that metadata. In SQLAlchemy ORM, we are going to define a class that inherits from a special base class called the declarative_base. The declarative_base combines a metadata container and a mapper that maps our class to a database table. It also maps instances of the class to records in that table if they have been saved. Let’s dig into defining tables in this manner.

## Defining Tables via ORM Classes

A proper class for use with the ORM must do four things:

* Inherit from the declarative_base object.

* Contain __tablename__, which is the table name to be used in the database.

* Contain one or more attributes that are Column objects.

* Ensure one or more attributes make up a primary key.

We need to examine the last two attribute-related requirements a bit closer. First, defining columns in an ORM class is very similar to defining columns in a Table object, which we discussed in Chapter 2; however, there is one very important difference. When defining columns in an ORM class, we don’t have to supply the column name as the first argument to the Column constructor. Instead, the column name will be set to the name of the class attribute to which it is assigned. Everything else we covered in “Types” and “Columns” applies here too, and works as expected. Second, the requirement for a primary key might seem odd at first; however, the ORM has to have a way to uniquely identify and associate an instance of the class with a specific record in the underlying database table. Let’s look at our cookies table defined as an ORM class (Example 6-1).

Example 6-1. Cookies table defined as an ORM class
```python
from sqlalchemy import Table, Column, Integer, Numeric, String
from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer(), primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))
```
* Create an instance of the declarative_base
* Inherit from the Base.
* Define the table name.
* Define an attribute and set it to be a primary key

This will result in the same table as shown in Example 2-1, and that can be verified by looking at Cookie._table_ property:

```python
>>> Cookie.__table__
Table('cookies', MetaData(bind=None),
    Column('cookie_id', Integer(), table=<cookies>, primary_key=True,
            nullable=False),
    Column('cookie_name', String(length=50), table=<cookies>),
    Column('cookie_recipe_url', String(length=255), table=<cookies>),
    Column('cookie_sku', String(length=15), table=<cookies>),
    Column('quantity', Integer(), table=<cookies>),
    Column('unit_cost', Numeric(precision=12, scale=2),
    table=<cookies>), schema=None)
```

Let’s look at another example. This time I want to re-create our users table from Example 2-2. Example 6-2 demonstrates how additional keywords work the same in both the ORM and Core schemas.

Example 6-2. Another table with more column options
```python
from datetime import datetime
from sqlalchemy import DateTime


class User(Base):
    __tablename__ = 'users'
    user_id = Column(Integer(), primary_key=True)
    username = Column(String(15), nullable=False, unique=True)
    email_address = Column(String(255), nullable=False)
    phone = Column(String(20), nullable=False)
    password = Column(String(25), nullable=False)
    created_on = Column(DateTime(), default=datetime.now)
    updated_on = Column(DateTime(), default=datetime.now, onupdate=datetime.now)
```
* Here we are making this column required (nullable=False) and requiring the values to be unique.
* The default sets this column to the current time if a date isn’t specified.
* Using onupdate here will reset this column to the current time every time any part of the record is updated.

### Keys, Constraints, and Indexes

In “Keys and Constraints”, we discussed that it is possible to define keys and constraints in the table constructor in addition to being able to do it in the columns themselves, as shown earlier. However, when using the ORM, we are building classes and not using the table constructor. In the ORM, these can be added by using the __table_args__ attribute on our class. __table_args__ expects to get a tuple of additional table arguments, as shown here:

```python
class SomeDataClass(Base):
    __tablename__ = 'somedatatable'
    __table_args__ = (ForeignKeyConstraint(['id'], ['other_table.id']),
                      CheckConstraint(unit_cost >= 0.00',
                                      name='unit_cost_positive'))
```

The syntax is the same as when we use them in a Table() constructor. This also applies to what you learned in “Indexes”.

In this section, we covered how to define a couple of tables and their schemas with the ORM. Another important part of defining data models is establishing relationships between multiple tables and objects.

## Relationships

Related tables and objects are another place where there are differences between SQLAlchemy Core and ORM. The ORM uses a similar ForeignKey column to constrain and link the objects; however, it also uses a relationship directive to provide a property that can be used to access the related object. This does add some extra database usage and overhead when using the ORM; however, the pluses of having this capability far outweigh the drawbacks. I’d love to give you a rough estimate of the additional overhead, but it varies based on your data models and how you use them. In most cases, it’s not even worth considering. Example 6-3 shows how to define a relationship using the relationship and backref methods.

Example 6-3. Table with a relationship
```python
from sqlalchemy import ForeignKey, Boolean
from sqlalchemy.orm import relationship, backref


class Order(Base):
    __tablename__ = 'orders'

    order_id = Column(Integer(), primary_key=True)
    user_id = Column(Integer(), ForeignKey('users.user_id'))
    shipped = Column(Boolean(), default=False)

    user =  relationship("User", backref=backref('orders', order_by=order_id))
```

* Notice how we import the relationship and backref methods from sqlalchemy.orm.
* We are defining a ForeignKey just as we did with SQLAlchemy Core.
* This establishes a one-to-many relationship.

Looking at the user relationship defined in the Order class, it establishes a one-to-many relationship with the User class. We can get the User related to this Order by accessing the user property. This relationship also establishes an orders property on the User class via the backref keyword argument, which is ordered by the order_id. The relationship directive needs a target class for the relationship, and can optionally include a back reference to be added to target class. SQLAlchemy knows to use the ForeignKey we defined that matches the class we defined in the relationship. In the preceding example, the ForeignKey(users.user_id), which has the users table’s user_id column, maps to the User class via the __tablename__ attribute of users and forms the relationship.

It is also possible to establish a one-to-one relationship: in Example 6-4, the LineItem class has a one-to-one relationship with the Cookie class. The uselist=False keyword argument defines it as a one-to-one relationship. We also use a simpler back reference, as we do not care to control the order.

Example 6-4. More tables with relationships
```python
class LineItem(Base):
    __tablename__ = 'line_items'

    line_item_id = Column(Integer(), primary_key=True)
    order_id = Column(Integer(), ForeignKey('orders.order_id'))
    cookie_id = Column(Integer(), ForeignKey('cookies.cookie_id'))
    quantity = Column(Integer())
    extended_cost = Column(Numeric(12, 2))

    order = relationship("Order", backref=backref('line_items',
                                                  order_by=line_item_id))

    cookie = relationship("Cookie", uselist=False) 1
```
* This establishes a one-to-one relationship.

These two examples are good places to start with relationships; however, relationships are an extremely powerful part of the ORM. We’ve barely scrathed the surface on what can be done with relationships, and we will explore them more in the Cookbook in Chapter 14. With our classes all defined and our relationships established, we are ready to create our tables in the database.

## Persisting the Schema

To create our database tables, we are going to use the create_all method on the metadata within our Base instance. It requires an instance of an engine, just as it did in SQLAlchemy Core:

```python
from sqlalchemy import create_engine
engine = create_engine('sqlite:///:memory:')

Base.metadata.create_all(engine)
```

Before we discuss how to work with our classes and tables via the ORM, we need to learn about how sessions are used by the ORM. That’s what the next chapter is all about.



