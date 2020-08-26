# Chapter 9. Testing with SQLAlchemy ORM

Most testing inside of applications consists of both unit and functional tests; however, with SQLAlchemy, it can be a lot of work to correctly mock out a query statement or a model for unit testing. That work often does not truly lead to much gain over testing against a database during the functional test. This leads people to make wrapper functions for their queries that they can easily mock out during unit tests, or to just test against a database in both their unit and function tests. I personally like to use small wrapper functions when possible or—if that doesn’t make sense for some reason or I’m in legacy code—mock it out.

This chapter covers how to perform functional tests against a database, and how to mock out SQLAlchemy queries and connections.

## Testing with a Test Database

For our example application, we are going to have an app.py file that contains our application logic, and a db.py file that contains our data models and session. These files can be found in the CH10/ folder of this book’s example code.

How an application is structured is an implementation detail that can have quite an effect on how you need to do your testing. In db.py, you can see that our database is set up via the DataAccessLayer class. We’re using this data access class to enable us to initialize a database engine and session whenever we like. You’ll see this pattern commonly used in web frameworks when coupled with SQLAlchemy. The DataAccessLayer class is initialized without an engine and a session in the dal variable.

TIP
While we will be testing with a SQLite database in our examples, it is highly recommended that you test against the same database engine you use in your production environment. If you use a different database in testing, some of your constraints and statements that may work with SQLite might not work with, say, PostgreSQL.

Example 9-1 shows a snippet of our db.py file that contains the DataAccessLayer.

Example 9-1. DataAccessLayer class
```python
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class DataAccessLayer:

    def __init__(self):
        self.engine = None
        self.conn_string = 'some conn string'

    def connect(self):
        self.engine = create_engine(self.conn_string)
        Base.metadata.create_all(self.engine)
        self.Session = sessionmaker(bind=self.engine)


dal = DataAccessLayer()
```

* __init__ provides a way to initialize a connection with a specific connection string like a factory.
* The connect method creates all the tables in our Base class, and uses sessionmaker to create a easy way to make sessions for use in our application.
* The connect method provides an instance of the DataAccessLayer class that can be imported throughout our application.

In addition to our DataAccessLayer, we also have all our data models defined in the db.py file. These models are the same models we used in Chapter 8, all gathered together for us to use in our application. Here are all the models in the db.py file:

```python
from datetime import datetime

from sqlalchemy import (Column, Integer, Numeric, String, DateTime, ForeignKey,
                        Boolean, create_engine)
from sqlalchemy.orm import relationship, backref, sessionmaker


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer, primary_key=True)
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
    updated_on = Column(DateTime(), default=datetime.now, onupdate=datetime.now)

    def __repr__(self):
        return "User(username='{self.username}', " \
            "email_address='{self.email_address}', " \
            "phone='{self.phone}', " \
            "password='{self.password}')".format(self=self)


class Order(Base):
    __tablename__ = 'orders'
    order_id = Column(Integer(), primary_key=True)
    user_id = Column(Integer(), ForeignKey('users.user_id'))
    shipped = Column(Boolean(), default=False)

    user = relationship("User", backref=backref('orders', order_by=order_id))

    def __repr__(self):
        return "Order(user_id={self.user_id}, " \
            "shipped={self.shipped})".format(self=self)


class LineItem(Base):
    __tablename__ = 'line_items'
    line_item_id = Column(Integer(), primary_key=True)
    order_id = Column(Integer(), ForeignKey('orders.order_id'))
    cookie_id = Column(Integer(), ForeignKey('cookies.cookie_id'))
    quantity = Column(Integer())
    extended_cost = Column(Numeric(12, 2))

    order = relationship("Order", backref=backref('line_items',
                                                  order_by=line_item_id))
    cookie = relationship("Cookie", uselist=False)

    def __repr__(self):
        return "LineItem(order_id={self.order_id}, " \
            "cookie_id={self.cookie_id}, " \
            "quantity={self.quantity}, " \
            "extended_cost={self.extended_cost})".format(
                self=self)
```

With our data models and access class in place, we are ready to look at the code we are going to test. We are going to write tests for the get_orders_by_customer function we built in Chapter 7, which is found the app.py file, shown in Example 9-2.

Example 9-2. app.py to test
```python
from db import dal, Cookie, LineItem, Order, User, DataAccessLayer 1


def get_orders_by_customer(cust_name, shipped=None, details=False):
    query = dal.session.query(Order.order_id, User.username, User.phone) 2
    query = query.join(User)
    if details:
        query = query.add_columns(Cookie.cookie_name, LineItem.quantity,
                                  LineItem.extended_cost)
        query = query.join(LineItem).join(Cookie)
    if shipped is not None:
        query = query.filter(Order.shipped == shipped)
    results = query.filter(User.username == cust_name).all()
    return results
```
* This is our DataAccessLayer instance from the db.py file, and the data models we are using in our application code.
* Because our session is an attribute of the dal object, we need to access it from there.

Let’s look at all the ways the get_orders_by_customer function can be used. We’re going to assume for this exercise that we have already validated that the inputs to the function are of the correct type. However, it would be very wise to make sure you test with data that will work correctly and data that could cause errors. Here’s a list of the variables our function can accept and their possible values:

* cust_name can be blank, a string containing the name of a valid customer, or a string that does not contain the name of a valid customer.

* shipped can be None, True, or False.

* details can be True or False.

If we want to test all of the possible combinations, we will need 12 (that’s 3 * 3 * 2) tests to fully test this function.

NOTE
It is important not to test things that are just part of the basic functionality of SQLAlchemy, as SQLAlchemy already comes with a large collection of well-written tests. For example, we wouldn’t want to test a simple insert, select, delete, or update statement, as those are tested within the SQLAlchemy project itself. Instead, look to test things that your code manipulates that could affect how the SQLAlchemy statement is run or the results returned by it.

For this testing example, we’re going to use the built-in unittest module. Don’t worry if you’re not familiar with this module; I’ll explain the key points. First, we need to set up the test class and initialize the dal’s connection by creating a new file named test_app.py, as shown in Example 9-3.

Example 9-3. Setting up the tests
```python
import unittest

from db import dal


class TestApp(unittest.TestCase): 1

    @classmethod
    def setUpClass(cls): 2
        dal.conn_string = 'sqlite:///:memory:' 3
        dal.connect()
```
* unittest requires test classes inherited from unittest.TestCase.
* The setUpClass method is run once prior to all the tests.
* Sets the connection string to our in-memory database for testing.

With our database connected, we are ready to write some tests. We’re going to add these tests as functions within the TestApp class as shown in Example 9-4.

Example 9-4. The first six tests for blank usernames
```python
def test_orders_by_customer_blank(self): 1
        results = get_orders_by_customer('')
        self.assertEqual(results, []) 2

    def test_orders_by_customer_blank_shipped(self):
        results = get_orders_by_customer('', True)
        self.assertEqual(results, [])

    def test_orders_by_customer_blank_notshipped(self):
        results = get_orders_by_customer('', False)
        self.assertEqual(results, [])

    def test_orders_by_customer_blank_details(self):
        results = get_orders_by_customer('', details=True)
        self.assertEqual(results, [])

    def test_orders_by_customer_blank_shipped_details(self):
        results = get_orders_by_customer('', True, True)
        self.assertEqual(results, [])

    def test_orders_by_customer_blank_notshipped_details(self):
        results = get_orders_by_customer('', False, True)
        self.assertEqual(results, [])
```

* unittest expects each test to begin with the letters test.
* unittest uses the assertEqual method to verify that the result matches what you expect. Because a user is not found, you should get an empty list back.

Now save the test file as test_app.py, and run the unit tests with the following command:

```python
# python -m unittest test_app
......

Ran 6 tests in 0.018s
```

NOTE
You might get a warning about SQLite and decimal types when running the unit tests; just ignore this as it’s normal for our examples. It occurs because SQLite doesn’t have a true decimal type, and SQLAlchemy wants you to know that there could be some oddities due to the conversion from SQLite’s float type. It is always wise to investigate these messages, because in production code they will normally point you to the proper way you should be doing something. We are purposely triggering this warning here so that you can see what it looks like.

Now we need to load up some data, and make sure our tests still work. We’re going to reuse the work we did in Chapter 7, and insert the same users, orders, and line items. However, this time we are going to wrap the data inserts in a function called db_prep. This will allow us to insert this data prior to a test with a simple function call. For simplicity’s sake, I have put this function inside the db.py file (see Example 9-5); however, in real-world situations, it will often be located in a test fixtures or utilities file instead.

Example 9-5. Inserting some test data
```python
def prep_db(session):
    c1 = Cookie(cookie_name='dark chocolate chip',
                cookie_recipe_url='http://some.aweso.me/cookie/dark_cc.html',
                cookie_sku='CC02',
                quantity=1,
                unit_cost=0.75)
    c2 = Cookie(cookie_name='peanut butter',
                cookie_recipe_url='http://some.aweso.me/cookie/peanut.html',
                cookie_sku='PB01',
                quantity=24,
                unit_cost=0.25)
    c3 = Cookie(cookie_name='oatmeal raisin',
                cookie_recipe_url='http://some.okay.me/cookie/raisin.html',
                cookie_sku='EWW01',
                quantity=100,
                unit_cost=1.00)
    session.bulk_save_objects([c1, c2, c3])
    session.commit()

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

    o1 = Order()
    o1.user = cookiemon
    session.add(o1)

    line1 = LineItem(cookie=c1, quantity=2, extended_cost=1.00)

    line2 = LineItem(cookie=c3, quantity=12, extended_cost=3.00)

    o1.line_items.append(line1)
    o1.line_items.append(line2)
    session.commit()

    o2 = Order()
    o2.user = cakeeater

    line1 = LineItem(cookie=c1, quantity=24, extended_cost=12.00)
    line2 = LineItem(cookie=c3, quantity=6, extended_cost=6.00)

    o2.line_items.append(line1)
    o2.line_items.append(line2)

    session.add(o2)
    session.commit()
```

Now that we have a prep_db function, we can use it in our test_app.py setUpClass method to load data into the database prior to running our tests. So now our setUpClass method looks like this:

```python
@classmethod
def setUpClass(cls):
    dal.conn_string = 'sqlite:///:memory:'
    dal.connect()
    dal.session = dal.Session()
    prep_db(dal.session)
    dal.session.close()
```

We also need to create a new session before each test, and roll back any changes made during that session after each test. We can do that by adding a setUp method that is automatically run prior to each individual test, and a tearDown method that is autmatically run after each test, as follows:

```python
def setUp(self):
    dal.session = dal.Session()

def tearDown(self):
    dal.session.rollback()
    dal.session.close()
```
* Establishes a new session before each test.
* Rolls back any changes during the test and cleans up the session after each test.

We’re also going to add our expected results as properties of the TestApp class as shown here:

```python
cookie_orders = [(1, u'cookiemon', u'111-111-1111')]
    cookie_details = [
        (1, u'cookiemon', u'111-111-1111',
            u'dark chocolate chip', 2, Decimal('1.00')),
        (1, u'cookiemon', u'111-111-1111',
            u'oatmeal raisin', 12, Decimal('3.00'))]
```
We can use the test data to ensure that our function does the right thing when given a valid username. These tests go inside of our TestApp class as new functions, as Example 9-6 shows.

Example 9-6. Tests for a valid user
```python
def test_orders_by_customer(self):
        results = get_orders_by_customer('cookiemon')
        self.assertEqual(results, self.cookie_orders)

    def test_orders_by_customer_shipped_only(self):
        results = get_orders_by_customer('cookiemon', True)
        self.assertEqual(results, [])

    def test_orders_by_customer_unshipped_only(self):
        results = get_orders_by_customer('cookiemon', False)
        self.assertEqual(results, self.cookie_orders)

    def test_orders_by_customer_with_details(self):
        results = get_orders_by_customer('cookiemon', details=True)
        self.assertEqual(results, self.cookie_details)

    def test_orders_by_customer_shipped_only_with_details(self):
        results = get_orders_by_customer('cookiemon', True, True)
        self.assertEqual(results, [])

    def test_orders_by_customer_unshipped_only_details(self):
        results = get_orders_by_customer('cookiemon', False, True)
        self.assertEqual(results, self.cookie_details)
```

Using the tests in Example 9-6 as guidance, can you complete the tests for what happens for a different user, such as cakeeater? How about the tests for a username that doesn’t exist in the system yet? Or if we get an integer instead of a string for the username, what will be the result? When you are done, compare your tests to those in the supplied example code for this chapter to see if your tests are similar to the ones used in this book.

We’ve now learned how we can use SQLAlchemy in functional tests to determine whether a function behaves as expected on a given data set. We have also looked at how to set up a unittest file and how to prepare the database for use in our tests. Next, we are ready to look at testing without hitting the database.

## Using Mocks

This technique can be a powerful tool when you have a test environment where creating a test database doesn’t make sense or simply isn’t feasible. If you have a large amount of logic that operates on the result of the query, it can be useful to mock out the SQLAlchemy code to return the values you want so you can test only the surrounding logic. Normally when I am going to mock out some part of the query, I still create the in-memory database, but I don’t load any data into it, and I mock out the database connection itself. This allows me to control what is returned by the execute and fetch methods. We are going to explore how to do that in this section.

To learn how to use mocks in our tests, we are going to make a single test for a valid user. We will use the powerful Python mock library to control what is returned by the connection. Mock is part of the unittest module in Python 3. However, if you are using Python 2, you will need to install the mock library using pip to get the latest mock features. To do that, run this command:

```python
pip install mock
```

Now that we have mock installed, we can use it in our tests. Mock has a patch function that will let us replace a given object in a Python file with a MagicMock that we can control from our test. A MagicMock is a special type of Python object that tracks how it is used and allows us to define how it behaves based on how it is being used.

First, we need to import the mock library. In Python 2, we need to do the following:

```python
import mock
```

In Python 3, we need to do the following:

```python
from unittest import mock
```

With mock imported, we are going to use the patch method as a decorator to replace the session part of the dal object. A decorator is a function that wraps another function, and alters the wrapped function’s behavior. Because the dal object is imported by name into the app.py file, we will need to patch it inside the app module. This will get passed into the test function as an argument. Now that we have a mock object, we can set a return value for the execute method, which in this case should be nothing but a chained fetchall method whose return value is the data we want to test with. Example 9-7 shows the code needed to use the mock in place of the dal object.

Example 9-7. Mocked connection test
```python
import unittest
from decimal import Decimal

import mock

from app import get_orders_by_customer


class TestApp(unittest.TestCase):
    cookie_orders = [(1, u'cookiemon', u'111-111-1111')]
    cookie_details = [
        (1, u'cookiemon', u'111-111-1111',
            u'dark chocolate chip', 2, Decimal('1.00')),
        (1, u'cookiemon', u'111-111-1111',
            u'oatmeal raisin', 12, Decimal('3.00'))]

    @mock.patch('app.dal.session') 1
    def test_orders_by_customer(self, mock_dal): 2
        mock_dal.query.return_value.join.return_value.filter.return_value. \
            all.return_value = self.cookie_orders 3
        results = get_orders_by_customer('cookiemon') 4
        self.assertEqual(results, self.cookie_orders)
```
* Patching dal.session in the app module with a mock.
* That mock is passed into the test function as mock_dal.
* We set the return value of the execute method to the chained return value of the all method which we set to self.cookie_order.
* Now we call the test function where the dal.connection will be mocked and return the value we set in the prior step.

You can see that a complicated query like the one in Example 9-7 might get tedious quickly when trying to mock out the full query or connection. Don’t shy away from the work though; it can be very useful for finding obscure bugs

As an exercise, you should work on building the remainder of the tests that we built with the in-memory database with the mocked test types. I would encourage you to mock both the query and the connection to get familiar with the mocking process.

You should now feel comfortable with testing a function that contains SQLAlchemy ORM functions and models within it. You should also understand how to prepopulate data into the test database for use in your test. Finally, you should understand how to mock both the query and connection objects. While this chapter used a simple example, we will dive further into testing in Chapter 14, which looks at Flask and Pyramid.

Up next, we are going to look at how to handle an existing database with SQLAlchemy without the need to re-create all the schema in Python via reflection with Automap.









