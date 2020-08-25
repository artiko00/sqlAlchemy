# Chapter 7. Working with Data via SQLAlchemy ORM

Now that we have defined classes that represent the tables in our database and persisted them, let’s start working with data via those classes. In this chapter, we’ll look at how to insert, retrieve, update, and delete data. Then we’ll learn how to sort and group that data and take a look at how relationships work. We’ll begin by learning about the SQLAlchemy session, one of the most important parts of the SQLAlchemy ORM.

## The Session

The session is the way SQLAlchemy ORM interacts with the database. It wraps a database connection via an engine, and provides an identity map for objects that you load via the session or associate with the session. The identity map is a cache-like data structure that contains a unique list of objects determined by the object’s table and primary key. A session also wraps a transaction, and that transaction will be open until the session is committed or rolled back, very similar to the process described in “Transactions”.

To create a new session, SQLAlchemy provides the sessionmaker class to ensure that sessions can be created with the same parameters throughout an application. It does this by creating a Session class that has been configured according to the arguments passed to the sessionmaker factory. The sessionmaker factory should be used just once in your application global scope, and treated like a configuration setting. Let’s create a new session associated with an in-memory SQLite database:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///:memory:')

Session = sessionmaker(bind=engine)

session = Session()
```

* Imports the sessionmaker class.
* Defines a Session class with the bind configuration supplied by sessionmaker.
* Creates a session for our use from our generated Session class.

Now we have a session that we can use to interact with the database. While session has everything it needs to connect to the database, it won’t connect until we give it some instructions that require it to do so. We’re going to continue to use the classes we created in Chapter 6 for our examples in the chapter. We’re going to add some __repr__ methods to make it easy to see and re-create object instances; however, these methods are not required:

```python
from datetime import datetime

from sqlalchemy import (Table, Column, Integer, Numeric, String, DateTime,
                        ForeignKey)
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship, backref


Base = declarative_base()


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer(), primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))

    def __repr__(self):
        return "Cookie(cookie_name='{self.cookie_name}', " \
                       "cookie_recipe_url='{self.cookie_recipe_url}', " \
                       "cookie_sku='{self.cookie_sku}', " \
                       "quantity={self.quantity}, " \
                       "unit_cost={self.unit_cost})".format(self=self)


class User(Base):
    __tablename__ = 'users'

    user_id = Column(Integer(), primary_key=True)
    username = Column(String(15), nullable=False, unique=True)
    email_address = Column(String(255), nullable=False)
    phone = Column(String(20), nullable=False)
    password = Column(String(25), nullable=False)
    created_on = Column(DateTime(), default=datetime.now)
    updated_on = Column(DateTime(), default=datetime.now,
                        onupdate=datetime.now)

    def __repr__(self):
        return "User(username='{self.username}', " \
                     "email_address='{self.email_address}', " \
                     "phone='{self.phone}', " \
                     "password='{self.password}')".format(self=self)

class Order(Base):
    __tablename__ = 'orders'
    order_id = Column(Integer(), primary_key=True)
    user_id = Column(Integer(), ForeignKey('users.user_id'))

    user =  relationship("User", backref=backref('orders', order_by=order_id))

    def __repr__(self):
        return "Order(user_id={self.user_id}, " \
                      "shipped={self.shipped})".format(self=self)


class LineItems(Base):
    __tablename__ = 'line_items'
    line_item_id = Column(Integer(), primary_key=True)
    order_id = Column(Integer(), ForeignKey('orders.order_id'))
    cookie_id = Column(Integer(), ForeignKey('cookies.cookie_id'))
    quantity = Column(Integer())
    extended_cost = Column(Numeric(12, 2))
    order = relationship("Order", backref=backref('line_items',
                         order_by=line_item_id))
    cookie = relationship("Cookie", uselist=False, order_by=id)

    def __repr__(self):
        return "LineItems(order_id={self.order_id}, " \
                          "cookie_id={self.cookie_id}, " \
                          "quantity={self.quantity}, " \
                          "extended_cost={self.extended_cost})".format(
                    self=self)


Base.metadata.create_all(engine)
```

* A __repr__ method defines how the object should be represented. It typically is the constructor call required to re-create the instance. This will show up later in our print output.
* Creates the tables in the database defined by the engine.

With our classes re-created, we are ready to begin learning how to work with data in our database, and we are going to start with inserting data.

## Inserting Data

To create a new cookie record in our database, we initialize a new instance of the Cookie class that has the desired data in it. We then add that new instance of the Cookie object to the session and commit the session. This is even easier to do because inheriting from the declarative_base provides a default constructor we can use (Example 7-1).

Example 7-1. Inserting a single object
```python
cc_cookie = Cookie(cookie_name='chocolate chip',
                   cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
                   cookie_sku='CC01',
                   quantity=12,
                   unit_cost=0.50)
session.add(cc_cookie)
session.commit()
```

* Creating an instance of the Cookie class.
* Adding the instance to the session.
* Committing the session.

When commit() is called on the session, the cookie is actually inserted into the database. It also updates cc_cookie with the primary key of the record in the database. We can see that by doing the following:

```python
print(cc_cookie.cookie_id)

>>>1
```

Let’s take a moment to discuss what happens to the database when we run the code in Example 7-1. When we create the instance of the Cookie class and then add it to the session, nothing is sent to the database. It’s not until we call commit() on the session that anything is sent to the database. When commit() is called, the following happens:

```python
INFO:sqlalchemy.engine.base.Engine:BEGIN (implicit)

INFO:sqlalchemy.engine.base.Engine:INSERT INTO cookies (cookie_name,
cookie_recipe_url, cookie_sku, quantity, unit_cost) VALUES (?, ?, ?, ?, ?)

INFO:sqlalchemy.engine.base.Engine:('chocolate chip',
'http://some.aweso.me/cookie/recipe.html', 'CC01', 12, 0.5) 3

INFO:sqlalchemy.engine.base.Engine:COMMIT
```

* Start a transaction.
* Insert the record into the database.
* The values for the insert.
* Commit the transaction.

If you want to see the details of what is happening here, you can add echo=True to your create_engine statement as a keyword argument after the connection string. Make sure to only do this for testing, and don’t use echo=True in production!

First, a fresh transaction is started, and the record is inserted into the database. Next, the engine sends the values of our insert statement. Finally, the transaction is committed to the database, and the transaction is closed. This method of processing is often called the Unit of Work pattern.

Next, let’s look at a couple of ways to insert multiple records. If you are going to do additional work with the objects after inserting them, you’ll want to use the method shown in Example 7-2. We simple create multiple instances, then add them both to the session prior to committing the session.

Example 7-2. Multiple inserts
```python
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
session.add(dcc)
session.add(mol)
session.flush()
print(dcc.cookie_id)
print(mol.cookie_id)
```
* Adds the dark chocolate chip cookie.
* Adds the molasses cookie.
* Flushes the session.

Notice that we used the flush() method on the session instead of commit() in Example 7-2. A flush is like a commit; however, it doesn’t perform a database commit and end the transaction. Because of this, the dcc and mol instances are still connected to the session, and can be used to perform additional database tasks without triggering additional database queries. We also issue the session.flush() statement one time, even though we added multiple records into the database. This actually results in two insert statements being sent to the database inside a single transaction. Example 7-2 will result in the following:

The second method of inserting multiple records into the database is great when you want to insert data into the table and you don’t need to perform additional work on that data. Unlike the method we used in Example 7-2, the method in Example 7-3 does not associate the records with the session.

Example 7-3. Bulk inserting multiple records
```python
c1 = Cookie(cookie_name='peanut butter',
            cookie_recipe_url='http://some.aweso.me/cookie/peanut.html',
            cookie_sku='PB01',
            quantity=24,
            unit_cost=0.25)
c2 = Cookie(cookie_name='oatmeal raisin',
            cookie_recipe_url='http://some.okay.me/cookie/raisin.html',
            cookie_sku='EWW01',
            quantity=100,
            unit_cost=1.00)
session.bulk_save_objects([c1,c2])
session.commit()
print(c1.cookie_id)
```

Adds the cookies to a list and uses the bulk_save_objects method.

Example 7-3 will not result in anything being printed to the screen because the c1 object isn’t associated with the session, and can’t refresh its cookie_id for printing. If we look at what was sent to the database, we can see only a single insert statement was in the transaction:

```python
INFO:sqlalchemy.engine.base.Engine:INSERT INTO cookies (cookie_name,
cookie_recipe_url, cookie_sku, quantity, unit_cost) VALUES (?, ?, ?, ?, ?)

INFO:sqlalchemy.engine.base.Engine:(
('peanut butter', 'http://some.aweso.me/cookie/peanut.html', 'PB01', 24, 0.25),
('oatmeal raisin', 'http://some.okay.me/cookie/raisin.html', 'EWW01', 100, 1.0))
INFO:sqlalchemy.engine.base.Engine:COMMIT
```

A single insert

The method demonstrated in Example 7-3 is substantially faster than performing multiple individual adds and inserts as we did in Example 7-2. This speed does come at the expense of some features we get in the normal add and commit, such as:

* Relationship settings and actions are not respected or triggered.
* The objects are not connected to the session.
* Fetching primary keys is not done by default.
* No events will be triggered.

In addition to bulk_save_objects, there are additional methods to create and update objects via a dictionary, and you can learn more about bulk operations and their performance in the SQLAlchemy documentation.

If you are inserting multiple records and don’t need access to relationships or the inserted primary key, use bulk_save_objects or its related methods. This is especially true if you are ingesting data from an external data source such as a CSV or a large JSON document with nested arrays.

Now that we have some data in our cookies table, let’s learn how to query the tables and retrieve that data.

## Querying Data

To begin building a query, we start by using the query() method on the session instance. Initially, let’s select all the records in our cookies table by passing the Cookie class to the query() method, as shown in Example 7-4.

Example 7-4. Get all the cookies
```python
cookies = session.query(Cookie).all()
print(cookies)
```

Returns a list of Cookie instances that represent all the records in the cookies table.

Example 7-4 will output the following:

```python
[
    Cookie(cookie_name='chocolate chip',
           cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
           cookie_sku='CC01', quantity=12, unit_cost=0.50),
    Cookie(cookie_name='dark chocolate chip',
           cookie_recipe_url='http://some.aweso.me/cookie/recipe_dark.html',
           cookie_sku='CC02', quantity=1, unit_cost=0.75),
    Cookie(cookie_name='molasses',
           cookie_recipe_url='http://some.aweso.me/cookie/recipe_molasses.html',
           cookie_sku='MOL01', quantity=1, unit_cost=0.80),
    Cookie(cookie_name='peanut butter',
           cookie_recipe_url='http://some.aweso.me/cookie/peanut.html',
           cookie_sku='PB01', quantity=24, unit_cost=0.25),
    Cookie(cookie_name='oatmeal raisin',
           cookie_recipe_url='http://some.okay.me/cookie/raisin.html',
           cookie_sku='EWW01', quantity=100, unit_cost=1.00)
]
```
Because the returned value is a list of objects, we can use those objects as we would normally. These objects are connected to the session, which means we can change them or delete them and persist that change to the database as shown later in this chapter.

In addition to being able to get back a list of objects all at once, we can use the query as an iterable, as shown in Example 7-5.

Example 7-5. Using the query as an iterable
```python
for cookie in session.query(Cookie):
    print(cookie)
```
We don’t append an all() when using an iterable.

Using the iterable approach allows us to interact with each record object individually, release it, and get the next object.

Example 7-5 would display the following output:

```python
Cookie(cookie_name='chocolate chip',
       cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
       cookie_sku='CC01', quantity=12, unit_cost=0.50),
Cookie(cookie_name='dark chocolate chip',
       cookie_recipe_url='http://some.aweso.me/cookie/recipe_dark.html',
       cookie_sku='CC02', quantity=1, unit_cost=0.75),
Cookie(cookie_name='molasses',
       cookie_recipe_url='http://some.aweso.me/cookie/recipe_molasses.html',
       cookie_sku='MOL01', quantity=1, unit_cost=0.80),
Cookie(cookie_name='peanut butter',
       cookie_recipe_url='http://some.aweso.me/cookie/peanut.html',
       cookie_sku='PB01', quantity=24, unit_cost=0.25),
Cookie(cookie_name='oatmeal raisin',
       cookie_recipe_url='http://some.okay.me/cookie/raisin.html',
       cookie_sku='EWW01', quantity=100, unit_cost=1.00)
```

In addition to using the query as an iterable or calling the all() method, there are many other ways of accessing the data. You can use the following methods to fetch results:

```python
first()
```
Returns the first record object if there is one.

```python
one()
```
Queries all the rows, and raises an exception if anything other than a single result is returned.
```python
scalar()
```
Returns the first element of the first result, None if there is no result, or an error if there is more than one result.

### TIPS FOR GOOD PRODUCTION CODE
When writing production code, you should follow these guidelines:

* Use the iterable version of the query over the all() method. It is more memory efficient than handling a full list of objects and we tend to operate on the data one record at a time anyway.
* To get a single record, use the first() method (rather than one() or scalar()) because it is clearer to our fellow coders. The only exception to this is when you must ensure that there is one and only one result from a query; in that case, use one().
* Use the scalar() method sparingly, as it raises errors if a query ever returns more than one row with one column. In a query that selects entire records, it will return the entire record object, which can be confusing and cause errors.

Every time we queried the database in the preceding examples, all the columns were returned for every record. Often we only need a portion of those columns to perform our work. If the data in these extra columns is large, it can cause our applications to slow down and consume far more memory than it should. SQLAlchemy does not add a bunch of overhead to the queries or objects; however, accounting for the data you get back from a query is often the first place to look if a query is consuming too much memory. Let’s look at how to limit the columns returned in a query.

### Controlling the Columns in the Query
To limit the fields that are returned from a query, we need to pass in the columns we want in the query() method constructor separated by columns. For example, you might want to run a query that returns only the name and quantity of cookies, as shown in Example 7-6.

Example 7-6. Select only cookie_name and quantity
```python
print(session.query(Cookie.cookie_name, Cookie.quantity).first()) 1
```

Selects the cookie_name and quantity from the cookies table and returns the first result.

When we run Example 7-6, it outputs the following:

```python
(u'chocolate chip', 12),
```

The output from a query where we supply the column names is a tuple of those column values.

Now that we can build a simple select statement, let’s look at other things we can do to alter how the results are returned in a query. We’ll start with changing the order in which the results are returned.

### Ordering

If you were to look at all the results from Example 7-6 instead of just the first record, you would see that the data is not really in any particular order. However, if we want the list to be returned in a particular order, we can chain an order_by() statement to our select, as shown in Example 7-7. In this case, we want the results to be ordered by the quantity of cookies we have on hand.

Example 7-7. Order by quantity ascending
```python
for cookie in session.query(Cookie).order_by(Cookie.quantity):
    print('{:3} - {}'.format(cookie.quantity, cookie.cookie_name))
```

Example 7-7 will print the following output:

```python

  1 - dark chocolate chip
  1 - molasses
 12 - chocolate chip
 24 - peanut butter
100 - oatmeal raisin
```

If you want to sort in reverse or descending order, use the desc() statement. The desc() function wraps the specific column you want to sort in a descending manner, as shown in Example 7-8.

Example 7-8. Order by quantity descending
```python
from sqlalchemy import desc
for cookie in session.query(Cookie).order_by(desc(Cookie.quantity)):
    print('{:3} - {}'.format(cookie.quantity, cookie.cookie_name))
```

We wrap the column we want to sort descending in the desc() function.

NOTE
The desc() function can also be used as a method on a column object, such as Cookie.quantity.desc(). However, that can be a bit more confusing to read in long statements, and so I always use desc() as a function.

It’s also possible to limit the number of results returned if we only need a certain number of them for our application. The following section explains how.

### Limiting

In prior examples, we used the first() method to get just a single row back. While our query() gave us the one row we asked for, the actual query ran over and accessed all the results, not just the single record. If we want to limit the query, we can use array slice notation to actually issue a limit statement as part of our query. For example, suppose you only have time to bake two batches of cookies, and you want to know which two cookie types you should make. You can use our ordered query from earlier and add a limit statement to return the two cookie types that are most in need of being replenished. Example 7-9 shows how this can be done.

Example 7-9. Two fewest cookie inventories
```python
query = session.query(Cookie).order_by(Cookie.quantity)[:2]
print([result.cookie_name for result in query])
```
* This runs the query and slices the returned list. This can be very ineffecient with a large result set.

The output from Example 7-9 looks like this:

```python
[u'dark chocolate chip', u'molasses']
```

In addition to using the array slice notation, it is also possible to use the limit() statement.

Example 7-10. Two fewest cookie inventories with limit
```python
query = session.query(Cookie).order_by(Cookie.quantity).limit(2)
print([result.cookie_name for result in query])
```

Now that you know what kind of cookies you need to bake, you’re probably starting to get curious about how many cookies are now left in your inventory. Many databases include SQL functions designed to make certain operations available directly on the database server such as SUM; let’s explore how to use these functions.

### Built-In SQL Functions and Labels

SQLAlchemy can also leverage SQL functions found in the backend database. Two very commonly used database functions are SUM() and COUNT(). To use these functions, we need to import the sqlalchemy.func module generator that makes them available. These functions are wrapped around the column(s) on which they are operating. Thus, to get a total count of cookies, you would use something like Example 7-11.

Example 7-11. Summing our cookies
```python
from sqlalchemy import func
inv_count = session.query(func.sum(Cookie.quantity)).scalar()
print(inv_count)
```
* Notice the use of scalar, which will return only the leftmost column in the first record.

NOTE
I tend to always import the func module generator, as importing sum directly can cause problems and confusion with Python’s built-in sum function

When we run Example 7-11, it results in:

```python
138
```

Now let’s use the count function to see how many cookie inventory records we have in our cookies table (Example 7-12).

Example 7-12. Counting our inventory records
```python
rec_count = session.query(func.count(Cookie.cookie_name)).first()
print(rec_count)
```

Unlike Example 7-11 where we used scalar and got a single value, Example 7-12 results in a tuple for us to use, as we used the first method instead of scalar:

```python
(5,)
```

Using functions such as count() and sum() will end up returning tuples or results with column names like count_1. These types of returns are often not what we want. Also, if we have several counts in a query we’d have to know the occurrence number in the statement, and incorporate that into the column name, so the fourth count() function would be count_4. This simply is not as explicit and clear as we should be in our naming, especially when surrounded with other Python code.

Thankfully, SQLAlchemy provides a way to fix this via the label() function. Example 7-13 performs the same query as Example 7-12; however, it uses label() to give us a more useful name to access that column.

Example 7-13. Renaming our count column
```python
rec_count = session.query(func.count(Cookie.cookie_name) \
                          .label('inventory_count')).first()
print(rec_count.keys())
print(rec_count.inventory_count)
```

I used the label() function on the column object I want to change.

Example 7-13 results in:

```python
[u'inventory_count']
5
```

We’ve seen examples of how to restrict the columns or the number of rows returned from the database, so now it’s time to learn about queries that filter data based on criteria we specify.

### Filtering

Filtering queries is done by appending filter() statements to our query. A typical filter() clause has a column, an operator, and a value or column. It is possible to chain multiple filters() clauses together or comma separate multiple ClauseElement expressions in a single filter, and they will act like ANDs in traditional SQL statements. In Example 7-14, we’ll find a cookie named “chocolate chip.”

Example 7-14. Filtering by cookie name with filter
```python
record = session.query(Cookie).filter(Cookie.cookie_name == 'chocolate chip').first()
print(record)
```

Example 7-14 prints out the chocolate chip cookie record:
```python
Cookie(cookie_name='chocolate chip',
       cookie_recipe_url='http://some.aweso.me/cookie/recipe.html',
       cookie_sku='CC01', quantity=12, unit_cost=0.50)
```

There is also a filter_by() method that works similarly to the filter() method except instead of explicity providing the class as part of the filter expression it uses attribute keyword expressions from the primary entity of the query or the last entity that was joined to the statement. It also uses a keyword assignment instead of a Boolean. Example 7-15 performs the exact same query as Example 7-14.

Example 7-15. Filtering by cookie name with filter_by
```python
record = session.query(Cookie).filter_by(cookie_name='chocolate chip').first()
print(record)
```

We can also use a where statement to find all the cookie names that contain the word “chocolate,” as shown in Example 7-16.

Example 7-16. Finding names with “chocolate” in them
```python
query = session.query(Cookie).filter(Cookie.cookie_name.like('%chocolate%'))
for record in query:
    print(record.cookie_name)
```

The code in Example 7-16 will return:

```python
chocolate chip
dark chocolate chip
```

In Example 7-16, we are using the Cookie.cookie_name column inside of a filter statement as a type of ClauseElement to filter our results, and we are taking advantage of the like() method that is available on ClauseElements. There are many other methods available, which are listed in Table 2-1.

If we don’t use one of the ClauseElement methods, then we will have an operator in our filter clauses. Most of the operators work as you might expect, but as the following section explains, there are a few differences.

### Operators

So far, we have only explored situations where a column was equal to a value or used one of the ClauseElement methods such as like(); however, we can also use many other common operators to filter data. SQLAlchemy provides overloading for most of the standard Python operators. This includes all the standard comparison operators (==, !=, <, >, <=, >=), which act exactly like you would expect in a Python statement. The == operator also gets an additional overload when compared to None, which converts it to an IS NULL statement. Arithmetic operators (\+, -, *, /, and %) are also supported with additional capabilities for database-independent string concatenation, as shown in Example 7-17.

Example 7-17. String concatenation with +
```python
results = session.query(Cookie.cookie_name, 'SKU-' + Cookie.cookie_sku).all()
for row in results:
    print(row)
```

Example 7-17 results in:
```python
('chocolate chip', 'SKU-CC01')
('dark chocolate chip', 'SKU-CC02')
('molasses', 'SKU-MOL01')
('peanut butter', 'SKU-PB01')
('oatmeal raisin', 'SKU-EWW01')
```

Another common usage of operators is to compute values from multiple columns. You’ll often do this in applications and reports dealing with data or statistics. Example 7-18 shows a common inventory value calculation.

Example 7-18. Inventory value by cookie
```python
from sqlalchemy import cast
query = session.query(Cookie.cookie_name,
                      cast((Cookie.quantity * Cookie.unit_cost),
                           Numeric(12,2)).label('inv_cost'))
for result in query:
    print('{} - {}'.format(result.cookie_name, result.inv_cost))
```

* cast is a function that allows us to convert types. In this case, we will be getting back results such as 6.0000000000, so by casting it, we can make it look like currency. It is also possible to accomplish the same task in Python with print('{} - {:.2f}'.format(row.cookie_name, row.inv_cost)).

* We are using the label() function to rename the column. Without this renaming, the column would not be listed in the keys of the result object, as the operation doesn’t have a name.

Example 7-18 results in:

```python
chocolate chip - 6.00
dark chocolate chip - 0.75
molasses - 0.80
peanut butter - 6.00
oatmeal raisin - 100.00
```

If we need to combine where statements, we can use a couple of different methods. One of those methods is known as Boolean operators.

### Boolean Operators

SQLAlchemy also allows for the SQL Boolean operators AND, OR, and NOT via the bitwise logical operators (&, |, and ~). Special care must be taken when using the AND, OR, and NOT overloads because of the Python operator precedence rules. For instance, & binds more closely than <, so when you write A < B & C < D, what you are actually writing is A < (B&C) < D, when you probably intended to get (A < B) & (C < D).

Often we want to chain multiple where clauses together in inclusive and exclusionary manners; this should be done via conjunctions.

### Conjunctions

While it is possible to chain multiple filter() clauses together, it’s often more readable and functional to use conjunctions to accomplish the desired effect. I also prefer to use conjunctions instead of Boolean operators, as conjunctions will make your code more expressive. The conjunctions in SQLAlchemy are and_(), or_(), and not_(). They have underscores to separate them from the built-in keywords. So if we wanted to get a list of cookies with a cost of less than an amount and above a certain quantity we could use the code shown in Example 7-19.

Example 7-19. Using filter with multiple ClauseElement expressions to perform an AND
```python
query = session.query(Cookie).filter(
    Cookie.quantity > 23,
    Cookie.unit_cost < 0.40
)
for result in query:
    print(result.cookie_name)
```

The or_() function works as the opposite of and_() and includes results that match either one of the supplied clauses. If we wanted to search our inventory for cookie types that we have between 10 and 50 of in stock or where the name contains chip, we could use the code shown in Example 7-20.

Example 7-20. Using the or() conjunction
```python
from sqlalchemy import and_, or_, not_
query = session.query(Cookie).filter(
    or_(
        Cookie.quantity.between(10, 50),
        Cookie.cookie_name.contains('chip')
    )
)
for result in query:
    print(result.cookie_name)
```

Example 7-20 results in:

```python
chocolate chip
dark chocolate chip
peanut butter
```

The not_() function works in a similar fashion to other conjunctions, and it is used to select records where a record does not match the supplied clause.

Now that we can comfortably query data, we are ready to move on to updating existing data.

## Updating Data

Much like the insert method we used earlier, there is also an update method with syntax almost identical to inserts, except that they can specify a where clause that indicates which rows to update. Like insert statements, update statements can be created by using either the update() function or the update() method on the table being updated. You can update all rows in a table by leaving off the where clause.

For example, suppose you’ve finished baking those chocolate chip cookies that we needed for our inventory. In Example 7-21, we’ll add them to our existing inventory with an update query, and then check to see how many we have currently have in stock.

Example 7-21. Updating data via object
```python
query = session.query(Cookie)
cc_cookie = query.filter(Cookie.cookie_name == "chocolate chip").first()
cc_cookie.quantity = cc_cookie.quantity + 120
session.commit()
print(cc_cookie.quantity)
```

* Lines 1 and 2 are using the generative method to build our statement.
* We’re querying to get the object here; however, if you already have it, you can directly edit it without querying it again.

Example 7-21 returns:

```python
132
```
It is also possible to update data in place without having the object originally (Example 7-22).

Example 7-22. Updating data in place
```python
query = session.query(Cookie)
query = query.filter(Cookie.cookie_name == "chocolate chip")
query.update({Cookie.quantity: Cookie.quantity - 20})

cc_cookie = query.first()
print(cc_cookie.quantity)
```

* The update() method causes the record to be updated outside of the session, and returns the number of rows updated.

* We are reusing query here because it has the same selection criteria we need.

Example 7-22 returns:

```python
112
```

In addition to updating data, at some point we will want to remove data from our tables. The following section explains how to do that.

## Deleting Data

To create a delete statement, you can use either the delete() function or the delete() method on the table from which you are deleting data. Unlike insert() and update(), delete() takes no values parameter, only an optional where clause (omitting the where clause will delete all rows from the table). See Example 7-23.

Example 7-23. Deleting data
```python
query = session.query(Cookie)
query = query.filter(Cookie.cookie_name == "dark chocolate chip")
dcc_cookie = query.one()
session.delete(dcc_cookie)
session.commit()
dcc_cookie = query.first()
print(dcc_cookie)
```

Example 7-23 returns:

```python
None
```

It is also possible to delete data in place without having the object (Example 7-24).

Example 7-24. Deleting data
```python
query = session.query(Cookie)
query = query.filter(Cookie.cookie_name == "molasses")
query.delete()
mol_cookie = query.first()
print(mol_cookie)
```

Example 7-24 returns:
```python
None
```

OK, at this point, let’s load up some data using what we already learned for the users, orders, and line_items tables. You can copy the code shown here, but you should also take a moment to play with different ways of inserting the data:

```python
cookiemon = User(username='cookiemon',
                 email_address='mon@cookie.com',
                 phone='111-111-1111',
                 password='password')
cakeeater = User(username='cakeeater',
                 email_address='cakeeater@cake.com',
                 phone='222-222-2222',
                 password='password')
pieperson = User(username='pieperson',
                 email_address='person@pie.com',
                 phone='333-333-3333',
                 password='password')
session.add(cookiemon)
session.add(cakeeater)
session.add(pieperson)
session.commit()
```

Now that we have customers, we can start to enter their orders and line items into the system as well. We’re going to take advantage of the relationship between these two objects when inserting them into the table. If we have an object that we want to associate to another one, we can do that by assigning it to the relationship property just like we would any other property. Let’s see this in action a few times in Example 7-25.

Example 7-25. Adding related objects
```python
o1 = Order()
o1.user = cookiemon 1
session.add(o1)

cc = session.query(Cookie).filter(Cookie.cookie_name ==
                                  "chocolate chip").one()
line1 = LineItem(cookie=cc, quantity=2, extended_cost=1.00) 2

pb = session.query(Cookie).filter(Cookie.cookie_name ==
                                  "peanut butter").one()
line2 = LineItem(quantity=12, extended_cost=3.00) 3
line2.cookie = pb
line2.order = o1

o1.line_items.append(line1) 4
o1.line_items.append(line2)

session.commit()
```

* Sets cookiemon as the user who placed the order.
* Builds a line item for the order, and associates it with the cookie.
* Builds a line item a piece at a time.
* Adds the line item to the order via the relationship.

In Example 7-25, we create an empty Order instance, and set its user property to the cookiemon instance. Next, we add it to the session. Then we query for the chocolate chip cookie and create a LineItem, and set the cookie to be the chocolate chip one we just queried for. We repeat that process for the second line item on the order; however, we build it up piecemeal. Finally, we add the line items to the order and commit it.

Before we move on, let’s add one more order for another user:

```python
o2 = Order()
o2.user = cakeeater

cc = session.query(Cookie).filter(Cookie.cookie_name ==
                                  "chocolate chip").one()
line1 = LineItem(cookie=cc, quantity=24, extended_cost=12.00)

oat = session.query(Cookie).filter(Cookie.cookie_name ==
                                   "oatmeal raisin").one()
line2 = LineItem(cookie=oat, quantity=6, extended_cost=6.00)

o2.line_items.append(line1)
o2.line_items.append(line2)

session.add(o2) 1
session.commit()
```

I moved this line down with the other adds, and SQLAlchemy determined the proper order in which to create them to ensure it was successful.

In Chapter 6, you learned how to define ForeignKeys and relationships, but we haven’t used them to perform any queries up to this point. Let’s take a look at relationships.

## Joins

Now let’s use the join() and outerjoin() methods to take a look at how to query related data. For example, to fulfill the order placed by the cookiemon user, we need to determine how many of each cookie type were ordered. This requires you to use a total of three joins to get all the way down to the name of the cookies. The ORM makes this query much easier than it normally would be with raw SQL (Example 7-26).

Example 7-26. Using join to select from multiple tables
```python
query = session.query(Order.order_id, User.username, User.phone,
                      Cookie.cookie_name, LineItem.quantity,
                      LineItem.extended_cost)
query = query.join(User).join(LineItem).join(Cookie) 1
results = query.filter(User.username == 'cookiemon').all()
print(results)
```

Tells SQLAlchemy to join the related objects.

Example 7-26 will output the following:

```python
[
    (u'1', u'cookiemon', u'111-111-1111', u'chocolate chip', 2, Decimal('1.00'))
    (u'1', u'cookiemon', u'111-111-1111', u'peanut butter', 12, Decimal('3.00'))
]
```

It is also useful to get a count of orders by all users, including those who do not have any present orders. To do this, we have to use the outerjoin() method, and it requires a bit more care in the ordering of the join, as the table we use the outerjoin() method on will be the one from which all results are returned (Example 7-27).

Example 7-27. Using outerjoin to select from multiple tables
```python
query = session.query(User.username, func.count(Order.order_id))
query = query.outerjoin(Order).group_by(User.username)
for row in query:
    print(row)
```

Example 7-27 results in:
```python
(u'cakeeater', 1)
(u'cookiemon', 1)
(u'pieperson', 0)
```

So far, we have been using and joining different tables in our queries. However, what if we have a self-referential table like a table of managers and their reports? The ORM allows us to establish a relationship that points to the same table; however, we need to specify an option called remote_side to make the relationship a many to one:

```python
class Employee(Base):
    __tablename__ = 'employees'

    id = Column(Integer(), primary_key=True)
    manager_id = Column(Integer(), ForeignKey('employees.id'))
    name = Column(String(255), nullable=False)

    manager = relationship("Employee", backref=backref('reports'),
                           remote_side=[id]) 1

Base.metadata.create_all(engine)
```

Establishes a relationship back to the same table, specifies the remote_side, and makes the relationship a many to one.

Let’s add an employee and another employee that reports to her:

```python
marsha = Employee(name='Marsha')
fred = Employee(name='Fred')

marsha.reports.append(fred)

session.add(marsha)
session.commit()
```

Now if we want to print the employees that report to Marsha, we would do so by accessing the reports property as follows:

```python
for report in marsha.reports:
    print(report.name)
```
It’s also useful to be able to group data when we are looking to report on data, so let’s look into that next.

## Grouping

When using grouping, you need one or more columns to group on and one or more columns that it makes sense to aggregate with counts, sums, etc., as you would in normal SQL. Let’s get an order count by customer (Example 7-28).

Example 7-28. Grouping data
```python
query = session.query(User.username, func.count(Order.order_id)) 1
query = query.outerjoin(Order).group_by(User.username) 2
for row in query:
    print(row)
```

* Aggregation via count

* Grouping by the nonaggregated included column

Example 7-28 results in:

```python
(u'cakeeater', 1)
(u'cookiemon', 1)
(u'pieguy', 0)
```
We’ve shown the generative building of statements throughout the previous examples, but I want to focus on it specifically for a moment.

## Chaining

We’ve used chaining several times throughout this chapter, and just didn’t acknowledge it directly. Where query chaining is particularly useful is when you are applying logic when building up a query. So if we wanted to have a function that got a list of orders for us it might look like Example 7-29.

Example 7-29. Chaining
```python
def get_orders_by_customer(cust_name):
    query = session.query(Order.order_id, User.username, User.phone,
                      Cookie.cookie_name, LineItem.quantity,
                      LineItem.extended_cost)
    query = query.join(User).join(LineItem).join(Cookie)
    results = query.filter(User.username == cust_name).all()
    return results

get_orders_by_customer('cakeeater')
```

Example 7-29 results in:

```python
[(u'2', u'cakeeater', u'222-222-2222', u'chocolate chip', 24, Decimal('12.00')),
 (u'2', u'cakeeater', u'222-222-2222', u'oatmeal raisin', 6, Decimal('6.00'))]
```

However, what if we wanted to get only the orders that have shipped or haven’t shipped yet? We’d have to write additional functions to support those additional filter options, or we can use conditionals to build up query chains. Another option we might want is whether or not to include details. This ability to chain queries and clauses together enables quite powerful reporting and complex query building (Example 7-30).

Example 7-30. Conditional chaining
```python
def get_orders_by_customer(cust_name, shipped=None, details=False):
    query = session.query(Order.order_id, User.username, User.phone)
    query = query.join(User)
    if details:
        query = query.add_columns(Cookie.cookie_name, LineItem.quantity,
                                  LineItem.extended_cost)
        query = query.join(LineItem).join(Cookie)
    if shipped is not None:
        query = query.where(Order.shipped == shipped)
    results = query.filter(User.username == cust_name).all()
    return results

get_orders_by_customer('cakeeater') 1

get_orders_by_customer('cakeeater', details=True) 2

get_orders_by_customer('cakeeater', shipped=True) 3

get_orders_by_customer('cakeeater', shipped=False) 4

get_orders_by_customer('cakeeater', shipped=False, details=True) 5
```

* Gets all orders.

* Gets all orders with details.

* Gets only orders that have shipped.

* Gets orders that haven’t shipped yet.

* Gets orders that haven’t shipped yet with details.

Example 7-30 results in:

```python
[(u'2', u'cakeeater', u'222-222-2222')]

[(u'2', u'cakeeater', u'222-222-2222', u'chocolate chip', 24, Decimal('12.00')),
 (u'2', u'cakeeater', u'222-222-2222', u'oatmeal raisin', 6, Decimal('6.00'))]

[]

[(u'2', u'cakeeater', u'222-222-2222')]

[(u'2', u'cakeeater', u'222-222-2222', u'chocolate chip', 24, Decimal('12.00')),
 (u'2', u'cakeeater', u'222-222-2222', u'oatmeal raisin', 6, Decimal('6.00'))]
```

So far in this chapter, we’ve used the ORM for all the examples; however, it is also possible to use literal SQL at times as well. Let’s take a look.

## Raw Queries

While I rarely use a full raw SQL statement, I will often use small text snippets to help make a query clearer. Example 7-31 shows a raw SQL where clause using the text() function.

Example 7-31. Partial text query
```python
from sqlalchemy import text
query = session.query(User).filter(text("username='cookiemon'"))
print(query.all())
```
Example 7-31 results in:

```python
[User(username='cookiemon', email_address='mon@cookie.com',
      phone='111-111-1111', password='password')]
```

Now you should have an understanding of how to use the ORM to work with data in SQLAlchemy. We explored how to create, read, update, and delete operations. This is a good point to stop and explore a bit on your own. Try to create more cookies, orders, and line items, and use query chains to group them by order and user.

Now that you’ve explored a bit more and hopefully broken something, let’s investigate how to react to exceptions raised in SQLAlchemy, and how to use transactions to group statements that must succeed or fail as a group.

CopyAdd HighlightAdd Note


