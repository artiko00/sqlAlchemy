# Chapter 4. Testing

Most testing inside of applications consists of both unit and functional tests; however, with SQLAlchemy, it can be a lot of work to correctly mock out a query statement or a model for unit testing. That work often does not truly lead to much gain over testing against a database during the functional test. This leads people to make wrapper functions for their queries that they can easily mock out during unit tests, or to just test against a database in both their unit and function tests. I personally like to use small wrapper functions when possible or—if that doesn’t make sense for some reason or I’m in legacy code—mock it out.

This chapter covers how to perform functional tests against a database, and how to mock out SQLAlchemy queries and connections.

## Testing with a Test Database

For our example application, we are going to have an app.py file that contains our application logic, and a db.py file that contains our database tables and connections. These files can be found in the CH05/ folder of the example code.

How an application is structured is an implementation detail that can have quite an effect on how you need to do your testing. In db.py, you can see that our database is set up via the DataAccessLayer class. We’re using this data access class to enable us to initialize a database schema, and connect it to an engine whenever we like. You’ll see this pattern commonly used in web frameworks when coupled with SQLAlchemy. The DataAccessLayer class is initialized without an engine and a connection in the dal variable. Example 4-1 shows a snippet of our db.py file.

```python
from datetime import datetime
from sqlalchemy import (MetaData, Table, Column, Integer, Numeric, String,
                        DateTime, ForeignKey, Boolean, create_engine)


class DataAccessLayer:
    connection = None
    engine = None
    conn_string = None
    metadata = MetaData()
    cookies = Table('cookies',
                    metadata,
                    Column('cookie_id', Integer(), primary_key=True),
                    Column('cookie_name', String(50), index=True),
                    Column('cookie_recipe_url', String(255)),
                    Column('cookie_sku', String(55)),
                    Column('quantity', Integer()),
                    Column('unit_cost', Numeric(12, 2))
              ) 1

    def db_init(self, conn_string): 2
        self.engine = create_engine(conn_string or self.conn_string)
        self.metadata.create_all(self.engine)
        self.connection = self.engine.connect()

dal = DataAccessLayer() 3
```
1- In the complete file, we create all the tables that we have been using since Chapter 1, not just cookies.
2- This provides a way to initialize a connection with a specific connection string like a factory.
3- This provides an instance of the DataAccessLayer class that can be imported throughout our application.

We are going to write tests for the get_orders_by_customer function we built in Chapter 2, which is found the app.py file, shown in Example 4-2.

```python
from db import dal 1
from sqlalchemy.sql import select




def get_orders_by_customer(cust_name, shipped=None, details=False):
    columns = [dal.orders.c.order_id, dal.users.c.username, dal.users.c.phone]
    joins = dal.users.join(dal.orders) 2

    if details:
        columns.extend([dal.cookies.c.cookie_name,
                        dal.line_items.c.quantity,
                        dal.line_items.c.extended_cost])
        joins = joins.join(dal.line_items).join(dal.cookies)

    cust_orders = select(columns)
    cust_orders = cust_orders.select_from(joins).where(
        dal.users.c.username == cust_name)

    if shipped is not None:
        cust_orders = cust_orders.where(dal.orders.c.shipped == shipped)

    return dal.connection.execute(cust_orders).fetchall()
```

1- This is our DataAccessLayer instance from the db.py file.
2- Because our tables are inside of the dal object, we access them from there.

Let’s look at all the ways the get_orders_by_customer function can be used. We’re going to assume for this exercise that we have already validated that the inputs to the function are of the correct type. However, in your testing, it would be very wise to make sure you test with data that will work correctly and data that could cause errors. Here’s a list of the variables our function can accept and their possible values:

* cust_name can be blank, a string containing the name of a valid customer, or a string that does not contain the name of a valid customer.
* shipped can be None, True, or False.
* details can be True or False.

If we want to test all of the possible combinations, we will need 12 (that 3 * 3 * 2) tests to fully test this function.

    NOTE
    It is important not to test things that are just part of the basic functionality of SQLAlchemy, as SQLAlchemy already comes with a large collection of well-written tests. For example, we wouldn’t want to test a simple insert, select, delete, or update statement, as those are tested within the SQLAlchemy project itself. Instead, look to test things that your code manipulates that could affect how the SQLAlchemy statement is run or the results returned by it.

For this testing example, we’re going to use the built-in unittest module. Don’t worry if you’re not familiar with this module; we’ll explain the key points. First, we need to set up the test class, and initialize the dal’s connection, which is shown in Example 4-3 by creating a new file named test_app.py.

```python
import unittest

class TestApp(unittest.TestCase): 1

    @classmethod
    def setUpClass(cls): 2
        dal.db_init('sqlite:///:memory:') 3
```

1- unittest requires test classes inherited from unittest.TestCase.
2- The setUpClass method is run once for the entire test class.
3- This line initializes a connection to an in-memory database for testing.

Now we need to load some data to use during our tests. I’m not going to include the full code here, as it is the same inserts we worked with in Chapter 2, modified to use the DataAccessLayer; it is available in the db.py example file. With our data loaded, we are ready to write some tests. We’re going to add these tests as functions within the TestApp class, as shown in Example 4-4.


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
1- unittest expects each test to begin with the letters test.
2- unittest uses assertEqual to verify that the result matches what you expect. Because a user is not found, you should get an empty list back.

Now save the test file as test_app.py, and run the unit tests with the following command:

```python
# python -m unittest test_app
......

Ran 6 tests in 0.018s
```

    NOTE
    You might get a warning about SQLite and decimal types; just ignore this as it’s normal for our examples. It occurs because SQLite doesn’t have a true decimal type, and SQLAlchemy wants you to know that there could be some oddities due to the conversion from SQLite’s float type. It is always wise to investigate these messages, because in production code they will normally point you to the proper way you should be doing something. We are purposely triggering this warning here so that you will see what it looks like.

Now we need to load up some data, and make sure our tests still work. Again, we’re going to reuse the work we did in Chapter 2, and insert the same users, orders, and line items. However, this time we are going to wrap the data inserts in a function called db_prep. This will allow us to insert this data prior to a test with a simple function call. For simplicity’s sake, I have put this function inside the db.py file (see Example 4-5); however, in real-world situations, it will often be located in a test fixtures or utilities file.

```python
def prep_db():
    ins = dal.cookies.insert()
    dal.connection.execute(ins, cookie_name='dark chocolate chip',
            cookie_recipe_url='http://some.aweso.me/cookie/recipe_dark.html',
            cookie_sku='CC02',
            quantity='1',
            unit_cost='0.75')
    inventory_list = [
        {
            'cookie_name': 'peanut butter',
            'cookie_recipe_url': 'http://some.aweso.me/cookie/peanut.html',
            'cookie_sku': 'PB01',
            'quantity': '24',
            'unit_cost': '0.25'
        },
        {
            'cookie_name': 'oatmeal raisin',
            'cookie_recipe_url': 'http://some.okay.me/cookie/raisin.html',
            'cookie_sku': 'EWW01',
            'quantity': '100',
            'unit_cost': '1.00'
        }
    ]
    dal.connection.execute(ins, inventory_list)

    customer_list = [
        {
            'username': "cookiemon",
            'email_address': "mon@cookie.com",
            'phone': "111-111-1111",
            'password': "password"
        },
        {
            'username': "cakeeater",
            'email_address': "cakeeater@cake.com",
            'phone': "222-222-2222",
            'password': "password"
        },
        {
            'username': "pieguy",
            'email_address': "guy@pie.com",
            'phone': "333-333-3333",
            'password': "password"
        }
    ]
    ins = dal.users.insert()
    dal.connection.execute(ins, customer_list)
    ins = insert(dal.orders).values(user_id=1, order_id='wlk001')
    dal.connection.execute(ins)
    ins = insert(dal.line_items)
    order_items = [
        {
            'order_id': 'wlk001',
            'cookie_id': 1,
            'quantity': 2,
            'extended_cost': 1.00
        },
        {
            'order_id': 'wlk001',
            'cookie_id': 3,
            'quantity': 12,
            'extended_cost': 3.00
        }
    ]
    dal.connection.execute(ins, order_items)
    ins = insert(dal.orders).values(user_id=2, order_id='ol001')
    dal.connection.execute(ins)
    ins = insert(dal.line_items)
    order_items = [
        {
            'order_id': 'ol001',
            'cookie_id': 1,
            'quantity': 24,
            'extended_cost': 12.00
        },
        {
            'order_id': 'ol001',
            'cookie_id': 4,
            'quantity': 6,
            'extended_cost': 6.00
        }
    ]
    dal.connection.execute(ins, order_items)
```

Now that we have a prep_db function, we can use that in our test_app.py setUpClass method to load data into the database prior to running our tests. So now our setUpClass method looks like this:

```python
@classmethod
def setUpClass(cls):
    dal.db_init('sqlite:///:memory:')
    prep_db()
```

We can use this test data to ensure that our function does the right thing when given a valid username. These tests go inside of our TestApp class as new functions, as Example 4-6 shows.

```python
def test_orders_by_customer(self):
        expected_results = [(u'wlk001', u'cookiemon', u'111-111-1111')]
        results = get_orders_by_customer('cookiemon')
        self.assertEqual(results, expected_results)

    def test_orders_by_customer_shipped_only(self):
        results = get_orders_by_customer('cookiemon', True)
        self.assertEqual(results, [])

    def test_orders_by_customer_unshipped_only(self):
        expected_results = [(u'wlk001', u'cookiemon', u'111-111-1111')]
        results = get_orders_by_customer('cookiemon', False)
        self.assertEqual(results, expected_results)

    def test_orders_by_customer_with_details(self):
        expected_results = [
            (u'wlk001', u'cookiemon', u'111-111-1111', u'dark chocolate chip',
             2, Decimal('1.00')),
            (u'wlk001', u'cookiemon', u'111-111-1111', u'oatmeal raisin',
             12, Decimal('3.00'))
        ]
        results = get_orders_by_customer('cookiemon', details=True)
        self.assertEqual(results, expected_results)

    def test_orders_by_customer_shipped_only_with_details(self):
        results = get_orders_by_customer('cookiemon', True, True)
        self.assertEqual(results, [])

    def test_orders_by_customer_unshipped_only_details(self):
        expected_results = [
            (u'wlk001', u'cookiemon', u'111-111-1111', u'dark chocolate chip',
             2, Decimal('1.00')),
            (u'wlk001', u'cookiemon', u'111-111-1111', u'oatmeal raisin',
             12, Decimal('3.00'))
        ]
        results = get_orders_by_customer('cookiemon', False, True)
        self.assertEqual(results, expected_results)
```

Using the tests in Example 4-6 as guidance, can you complete the tests for what happens for a different user, such as cakeeater? How about the tests for a username that doesn’t exist in the system yet? Or if we get an integer instead of a string for the username, what will be the result? Compare your tests to those in the supplied example code when you are done to see if your tests are similar to the ones used in this book.

We’ve learned how we can use SQLAlchemy in functional tests to determine whether a function behaves as expected on a given data set. We have also looked at how to set up a unittest file and how to prepare the database for use in our tests. Next, we are ready to look at testing without hitting the database.

## Using Mocks

This technique can be a powerful tool when you have a test environment where creating a test database doesn’t make sense or simply isn’t feasible. If you have a large amount of logic that operates on the result of the query, it can be useful to mock out the SQLAlchemy code to return the values you want so you can test only the surrounding logic. Normally when I am going to mock out some part of the query, I still create the in-memory database, but I don’t load any data into it, and I mock out the database connection itself. This allows me to control what is returned by the execute and fetch methods. We are going to explore how to do that in this section.

To learn how to use mocks in our tests, we are going to make a single test for a valid user. This time we will use the powerful Python mock library to control what is returned by the connection. Mock is part of the unittest module in Python 3. However, if you are using Python 2, you will need to install the mock library using pip to get the latest mock features. To do that, run this command:

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

With mock imported, we are going to use the patch method as a decorator to replace the connection part of the dal object. A decorator is a function that wraps another function, and alters the wrapped function’s behavior. Because the dal object is imported by name into the app.py file, we will need to patch it inside the app module. This will get passed into the test function as an argument. Now that we have a mock object, we can set a return value for the execute method, which in this case should be nothing but a chained fetchall method whose return value is the data we want to test with. Example 4-7 shows the code needed to use the mock in place of the dal object.

```python
import unittest
from decimal import Decimal

import mock

from db import dal, prep_db
from app import get_orders_by_customer


class TestApp(unittest.TestCase):
    cookie_orders = [(u'wlk001', u'cookiemon', u'111-111-1111')]
    cookie_details = [
        (u'wlk001', u'cookiemon', u'111-111-1111',
         u'dark chocolate chip', 2, Decimal('1.00')),
        (u'wlk001', u'cookiemon', u'111-111-1111',
         u'oatmeal raisin', 12, Decimal('3.00'))
    ]

    @mock.patch('app.dal.connection') 1
    def test_orders_by_customer(self, mock_conn): 2
        mock_conn.execute.return_value.fetchall.return_value = self.cookie_orders 3
        results = get_orders_by_customer('cookiemon') 4
        self.assertEqual(results, self.cookie_orders)

```

1- Patching dal.connection in the app module with a mock.
2- That mock is passed into the test function as mock_conn.
3- We set the return value of the execute method to the chained returned value of the fetchall method, which we set to self.cookie_order.
4- Now we call the test function where the dal.connection will be mocked and return the value we set in the prior step.

You can see that a complicated query or ResultProxy like the one in Example 4-7 might get tedious quickly when trying to mock out the full query or connection. Don’t shy away from the work though; it can be very useful for finding obscure bugs.

If you did want to mock out the query, you would follow the same pattern of using the mock.patch decorator and mocking out the select object in the app module. Let’s try that with one of the empty test queries. We have to mock out all the chained query element return values, which in this query are the select, select_from, and where clauses. Example 4-8 demonstrates how to do this.

```python
@mock.patch('app.select') 1
@mock.patch('app.dal.connection')
def test_orders_by_customer_blank(self, mock_conn, mock_select): 2
    mock_select.return_value.select_from.return_value.\
        where.return_value = '' 3
    mock_conn.execute.return_value.fetchall.return_value = [] 4
    results = get_orders_by_customer('')
    self.assertEqual(results, [])
```

1- Mocking out the select method, as it starts the query chain.
2- The decorators are passed into the function in order. As we progress up the decorators stack from the function, the arguments get added to the left.
3- We have to mock the return value for all parts of the chained query.
4- We still need to mock the connection or the app module SQLAlchemy code would try to make the query.

As an exercise, you should work on building the remainder of the tests that we built with the in-memory database with the mocked test types. I would encourage you to mock both the query and the connection to get familiar with the mocking process.

You should now feel comfortable with how to test a function that contains SQLAlchemy functions within it. You should also understand how to prepopulate data into the test database for use in your test. Finally, you should understand how to mock both the query and connection objects. While this chapter used a simple example, we will dive further into testing in Chapter 14, which looks at Flask, Pyramid, and pytest.

Up next, we are going to look at how to handle an existing database with SQLAlchemy without the need to re-create all the schema in Python via reflection.