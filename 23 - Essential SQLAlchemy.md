# Chapter 8. Understanding the Session and Exceptions

In the previous chapter, we did a lot of work with the session, and we avoided doing anything that could result in an exception. In this chapter, we will learn a bit more about how our objects and the SQLAlchemy session interact. We’ll conclude this chapter by purposely performing some actions incorrectly so that we can see the types of exceptions that occur and how we should respond to them. Let’s start by learning more about the session. First, we’ll set up an in-memory SQLite database using the tables from Chapter 6:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///:memory:')

Session = sessionmaker(bind=engine)

session = Session()

from datetime import datetime

from sqlalchemy import (Table, Column, Integer, Numeric, String, DateTime,
                        ForeignKey, Boolean)
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship, backref


Base = declarative_base()


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer, primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))

    def __init__(self, name, recipe_url=None, sku=None, quantity=0,
                 unit_cost=0.00):
        self.cookie_name = name
        self.cookie_recipe_url = recipe_url
        self.cookie_sku = sku
        self.quantity = quantity
        self.unit_cost = unit_cost

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

    def __init__(self, username, email_address, phone, password):
        self.username = username
        self.email_address = email_address
        self.phone = phone
        self.password = password

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

    user =  relationship("User", backref=backref('orders', order_by=order_id))

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
        return "LineItems(order_id={self.order_id}, " \
                          "cookie_id={self.cookie_id}, " \
                          "quantity={self.quantity}, " \
                          "extended_cost={self.extended_cost})".format(
                    self=self)

Base.metadata.create_all(engine)
```

Now that we have our initial database and objects defined, we’re ready to learn more about how the session works with objects.

## The SQLAlchemy Session

We covered a bit about the SQLAlchemy session in Chapter 7; however, here I want to focus on how our data objects and the session interact.

When we use a query to get an object, we get back an object that is connected to a session. That object could move through several states in relationship to the session.

### Session States

Understanding the session states can be useful for troubleshooting exceptions and handling unexpected behaviors. There are four possible states for data object instances:

```python
Transient
```
The instance is not in session, and is not in the database.

```python
Pending
```
The instance has been added to the session with add(), but hasn’t been flushed or committed.

```python
Persistent
```
The object in session has a corresponding record in the database.

```python
Detached
```
The instance is no longer connected to the session, but has a record in the database.


We can watch an instance move through these states as we work with it. We’ll start by creating an instance of a cookie:
```python
cc_cookie = Cookie('chocolate chip',
                   'http://some.aweso.me/cookie/recipe.html',
                   'CC01', 12, 0.50)
```

To see the instance state, we can use the powerful inspect() method provided by SQLAlchemy. When we use inspect() on an instance, we gain access to several useful pieces of information. Let’s get an inspection of our cc_cookie instance:

```python
from sqlalchemy import inspect
insp = inspect(cc_cookie)
```

In this case, we are interested in the transient, pending, persistent, and detached properties that indicate the current state. Let’s loop through those properties as shown in Example 8-1.

Example 8-1. Getting the session state of an instance
```python
for state in ['transient', 'pending', 'persistent', 'detached']:
    print('{:>10}: {}'.format(state, getattr(insp, state)))
```
Example 8-1 results in a list of states and a Boolean indicating whether that is the current state:

```python
    transient: True
    pending: False
    persistent: False
    detached: False
```

As you can see in the output, the current state of our cookie instance is transient, which is the state newly created objects are in prior to being flushed or commited to the database. If we add cc_cookie to the current session and rerun Example 8-1, we will get the following output:

```python
    transient: False
    pending: True
    persistent: False
    detached: False
```

Now that our cookie instance has been added to the current session, we can see it has been moved to pending. If we commit the session and rerun Example 8-1 yet again, we can see the state is updated to persistent:

```python
    transient: False
    pending: False
    persistent: True
    detached: False
```

Finally, to get cc_cookie into the detached state, we want to call the expunge() method on the session. You might do this if you are moving data from one session to another. One case in which you might want to move data from one session to another is when you are archiving or consolidating data from your primary database to your data warehouse:

```python
session.expunge(cc_cookie)
```

If we rerun Example 8-1 one last time after expunging cc_cookie, we’ll see that it is now in a detached state:

```python
    transient: False
    pending: False
    persistent: False
    detached: True
```

If you are using the inspect() method in normal code, you’ll probably want to use insp.transient, insp.pending, insp.persistent, and insp.detached. We used getattr to access them so we could loop through the states instead of hardcoding each state individually.

Now that we’ve seen how an object moves through the session states, let’s look at how we can use the inspector to see the history of an instance prior to committing it. First, we’ll add our object back to the session and change the cookie_name attribute:

```python
session.add(cc_cookie)
cc_cookie.cookie_name = 'Change chocolate chip'
```

Now let’s use the inspector’s modified property to see if it has changed:

```python
insp.modified
```

This will return True, and we can use the inspector’s attrs collection to find what has changed. Example 8-2 demonstrates one way we can accomplish this.

Example 8-2. Printing the changed attribute history
```python
for attr, attr_state in insp.attrs.items():
    if attr_state.history.has_changes(): 1
        print('{}: {}'.format(attr, attr_state.value))
        print('History: {}\n'.format(attr_state.history)) 2
```

* Checks the attribute state to see if the session can find any changes

* Prints the history object of the changed attribute

When we run Example 8-2, we will get back:

```python
cookie_name: Change chocolate chip
History: History(added=['Change chocolate chip'], unchanged=(), deleted=())
```

The output from Example 8-2 shows us that the cookie_name changed. When we look at the history record for that attribute, it shows us what was added or changed on that attribute. This gives us a good view of how objects interact with the session. Now, let’s investigate how to handle exceptions that arise while using SQLAlchemy ORM.

## Exceptions

There are numerous exceptions that can occur in SQLAlchemy, but we’ll focus on two in particular: MultipleResultsFound and DetachedInstanceError. These exceptions are fairly common, and they also are part of a group of similar exceptions. By learning how to handle these exceptions, you’ll be better prepared to deal with other exceptions you might encounter.

### MultipleResultsFound Exception

This exception occurs when we use the .one() query method, but get more than one result back. Before we can demonstrate this exception, we need to create another cookie and save it to the database:

```python
dcc = Cookie('dark chocolate chip',
             'http://some.aweso.me/cookie/recipe_dark.html',
             'CC02', 1, 0.75)
session.add(dcc)
session.commit()
```

Now that we have more than one cookie in the database, let’s trigger the MultipleResultsFound exception (Example 8-3).

Example 8-3. Causing a MultipleResultsFound exception

```python
results = session.query(Cookie).one()
```
Example 8-3 results in an exception because there are two cookies in our data source that match the query. The exception stops the execution of our program. Example 8-4 shows what this exception looks like.

Example 8-4. Exception output from Example 8-3

```python
MultipleResultsFound                   Traceback (most recent call last) 1
<ipython-input-20-d88068ecde4b> in <module>()
----> 1 results = session.query(Cookie).one() 2

...b/python2.7/site-packages/sqlalchemy/orm/query.pyc in one(self)
   2480         else:
   2481             raise orm_exc.MultipleResultsFound(
-> 2482                 "Multiple rows were found for one()")
   2483
   2484     def scalar(self):

MultipleResultsFound: Multiple rows were found for one() 3
```

* This shows us the type of exception and that a traceback is present.

* This is the actual line where the exception occurred.

* This is the interesting part we need to focus on.

In Example 8-4, we have the typical format for a MultipleResultsFound exception from SQLAlchemy. It starts with a line that indicates the type of exception. Next, there is a traceback showing us where the exception occurred. The final block of lines is where the important details can be found: they specify the type of exception and explain why it occurred. In this case, it is because our query returned two rows and we told it to return one and only one.

NOTE
Another exception related to this is the NoResultFound exception, which would occur if we used the .one() method and the query returned no results.

If we wanted to handle this exception so that our program doesn’t stop executing and prints out a more helpful exception message, we can use the Python try/except block, as shown in Example 8-5.

Example 8-5. Handling a MultipleResultsFound exception
```python
from sqlalchemy.orm.exc import MultipleResultsFound 1
try:
    results = session.query(Cookie).one()
except MultipleResultsFound as error: 2
    print('We found too many cookies... is that even possible?')
```

* All of the SQLAlchemy ORM exceptions are available in the sqlalchemy.orm.exc module.
* Catching the MultipleResultsFound exception as error.

Executing Example 8-5 will result in “We found too many cookies… is that even possible?” being printed and our application will keep on running normally. (I don’t think is it possible to find too many cookies.) Now you know what the MultipleResultsFound exception is telling you, and you know at least one method for dealing with it in a way that allows your program to continue executing.

### DetachedInstanceError

This exception occurs when we attempt to access an attribute on an instance that needs to be loaded from the database, but the instance we are using is not currently attached to the database. Before we can explore this exception, we need to set up the records we are going to operate on. Example 8-6 shows how to do that.

Example 8-6. Creating a user and an order for that user
```python
cookiemon = User('cookiemon', 'mon@cookie.com', '111-111-1111', 'password')
session.add(cookiemon)
o1 = Order()
o1.user = cookiemon
session.add(o1)

cc = session.query(Cookie).filter(Cookie.cookie_name ==
                                  "Change chocolate chip").one()
line1 = LineItem(order=o1, cookie=cc, quantity=2, extended_cost=1.00)

session.add(line1)
session.commit()
```

Now that we’ve created a user with an order and some line items, we have what we need to cause this exception. Example 8-7 will trigger the exception for us.

Example 8-7. Causing a DetachedInstanceError
```python
order = session.query(Order).first()
session.expunge(order)
order.line_items
```

In Example 8-7, we are querying to get an instance of an order. Once we have our instance, we detach it from the session with expunge. Then we attempt to load the line_items attribute. Because line_items is a relationship, by default it doesn’t load all that data until you ask for it. In our case, we detached the instance from the session and the relationship doesn’t have a session to execute a query to load the line_items attribute, and it raises the DetachedInstanceError:

```python
DetachedInstanceError                     Traceback (most recent call last)
<ipython-input-35-233bbca5c715> in <module>()
      1 order = session.query(Order).first()
      2 session.expunge(order)
----> 3 order.line_items 1

site-packages/sqlalchemy/orm/attributes.pyc in __get__(self, instance, owner)
    235             return dict_[self.key]
    236         else:
--> 237             return self.impl.get(instance_state(instance), dict_)
    238
    239

site-packages/sqlalchemy/orm/attributes.pyc in get(self, state, dict_, passive)
    576                     value = callable_(state, passive)
    577                 elif self.callable_:
--> 578                     value = self.callable_(state, passive)
    579                 else:
    580                     value = ATTR_EMPTY

site-packages/sqlalchemy/orm/strategies.pyc in _load_for_state(self, state,
passive)
    499                 "Parent instance %s is not bound to a Session; "
    500                 "lazy load operation of attribute '%s' cannot proceed" %
--> 501                 (orm_util.state_str(state), self.key)
    502             )
    503

DetachedInstanceError: Parent instance <Order at 0x10dc31350> is not bound to
a Session; lazy load operation of attribute 'line_items' cannot proceed 2
```

* This is the actual line where the error occurred.
* This is the interesting part we need to focus on.

We read this exception output just like we did in Example 8-5; the interesting part of the message indicates that we tried to load the line_items attribute on an Order instance, and it couldn’t do that because that instance wasn’t bound to a session.

While this is a forced example of the DetachedInstanceError, it acts very similarly to other exceptions such as ObjectDeletedError, StaleDataError, and ConcurrentModificationError. All of these are related to information differing between the instance, the session, and the database. The same method used in Example 8-6 with the try/except block will also work for this exception and allow the code to continue running. You could check for a detacted instance, and in the except block add it back to the session; however, the DetachedInstanceError is normally an indicator of an exception occuring prior to this point in the code.

TIP
To get more details on additional exceptions that can occur with the ORM, check out the SQLAlchemy documentation at http://docs.sqlalchemy.org/en/latest/orm/exceptions.html.

All the exceptions mentioned so far, such as MultipleResultsFound and DetachedInstanceError, are related to a single statement failing. If we have multiple statements, such as multiple adds or deletes that fail during a commit or a flush, we need to handle those differently. Specifically, we handle them by manually controlling the transaction. The following section covers transactions in detail, and teaches you how to use them to help handle exceptions and recover a session.

## Transactions

Transactions are a group of statements that we need to succeed or fail as a group. When we first create a session, it is not connected to the database. When we undertake our first action with the session such as a query, it starts a connection and a transaction. This means that by default, we don’t need to manually create transactions. However, if we need to handle any exceptions where part of the transaction succeeds and another part fails or where the result of a transaction creates an exception, then we must know how to control the transaction manually.

There is already a good example of when we might want to do this in our existing database. After a customer has ordered cookies from us, we need to ship those cookies to the customer and remove them from our inventory. However, what if we do not have enough of the right cookies to fulfill an order? We will need to detect that and not ship that order. We can solve this with transactions.

We’ll need a fresh Python shell with the tables from Chapter 6; however, we need to add a CheckConstraint to the quantity column to ensure it cannot go below 0, because we can’t have negative cookies in inventory. Next, we need to re-create the cookiemon user as well as the chocolate chip and dark chocolate chip cookie records and set the quantity of chocolate chip cookies to 12 and the dark chocolate chip cookies to 1. Example 8-8 shows how to perform all those steps.

Example 8-8. Setting up the transactions environment
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///:memory:')

Session = sessionmaker(bind=engine)

session = Session()

from datetime import datetime

from sqlalchemy import (Table, Column, Integer, Numeric, String,
                        DateTime, ForeignKey, Boolean, CheckConstraint)
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship, backref


Base = declarative_base()


class Cookie(Base):
    __tablename__ = 'cookies'
    __table_args__ = (CheckConstraint('quantity >= 0', name='quantity_positive'),) 1

    cookie_id = Column(Integer, primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))

    def __init__(self, name, recipe_url=None, sku=None, quantity=0, unit_cost=0.00):
        self.cookie_name = name
        self.cookie_recipe_url = recipe_url
        self.cookie_sku = sku
        self.quantity = quantity
        self.unit_cost = unit_cost

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

    def __init__(self, username, email_address, phone, password):
        self.username = username
        self.email_address = email_address
        self.phone = phone
        self.password = password

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

    user =  relationship("User", backref=backref('orders', order_by=order_id))

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

Base.metadata.create_all(engine)

cookiemon = User('cookiemon', 'mon@cookie.com', '111-111-1111', 'password') 2
cc = Cookie('chocolate chip', 'http://some.aweso.me/cookie/recipe.html', 3
            'CC01', 12, 0.50)


dcc = Cookie('dark chocolate chip', 4
             'http://some.aweso.me/cookie/recipe_dark.html',
             'CC02', 1, 0.75)

session.add(cookiemon)
session.add(cc)
session.add(dcc)
```

* Adds the positive quantity CheckConstraint.
* Adds the cookiemon user.
* Adds the chocolate chip cookie.
* Adds the dark chocolate chip cookie.

We’re now going to define two orders for the cookiemon user. The first order will be for two chocolate chip cookies and nine chocolate chip cookies, and the second order will be for nine dark chocolate chip cookies. Example 8-9 shows the details.

Example 8-9. Adding the orders
```python
o1 = Order()
o1.user = cookiemon
session.add(o1)

line1 = LineItem(order=o1, cookie=cc, quantity=9, extended_cost=4.50)


session.add(line1)
session.commit() 1
o2 = Order()
o2.user = cookiemon
session.add(o2)

line1 = LineItem(order=o2, cookie=cc, quantity=2, extended_cost=1.50)
line2 = LineItem(order=o2, cookie=dcc, quantity=9, extended_cost=6.75)


session.add(line1)
session.add(line2)
session.commit() 2
```

* Adding the first order.
* Adding the second order.


That will give us all the order data we need to explore how transactions work. Now we need to define a function called ship_it. Our ship_it function will accept an order_id, remove the cookies from inventory, and mark the order as shipped. Example 8-10 shows how this works.

Example 8-10. Defining the ship_it function
```python
def ship_it(order_id):
    order = session.query(Order).get(order_id)
    for li in order.line_items:
        li.cookie.quantity = li.cookie.quantity - li.quantity 1
        session.add(li.cookie)
    order.shipped = True 2
    session.add(order)
    session.commit()
    print("shipped order ID: {}".format(order_id))
```

* For each line item we find on the order, this removes the quantity on that line item from the cookie quantity so we know how many cookies we have left.
* We update the order to mark it as shipped.

The ship_it function will perform all the actions required when we ship an order. Let’s run it on our first order and then query the cookies table to make sure it reduced the cookie count correctly. Example 8-11 shows how to do that.

Example 8-11. Running ship_it on the first order
```python
ship_it(1) 1
print(session.query(Cookie.cookie_name, Cookie.quantity).all()) 2
```

* Runs ship_it on the first order_id.

* Prints our cookie inventory.

Running the code in Example 8-11 results in:

```python
shipped order ID: 1
[(u'chocolate chip', 3), (u'dark chocolate chip', 1)]
```
Excellent! It worked. We can see that we don’t have enough cookies in our inventory to fulfill the second order; however, in our fast-paced warehouse, these orders might be processed at the same time. Now try shipping our second order with the ship_it function by running the following line of code:

```python
ship_it(2)
```

That command gives us this result:

```python
IntegrityError                            Traceback (most recent call last)
<ipython-input-7-8a7f7805a7f6> in <module>()
 ---> 1 ship_it(2)
<ipython-input-5-c442ae46326c> in ship_it(order_id)
      6     order.shipped = True
      7     session.add(order)
 ---> 8     session.commit()
      9     print("shipped order ID: {}".format(order_id))
...

IntegrityError: (sqlite3.IntegrityError) CHECK constraint failed:
quantity_positive [SQL: u'UPDATE cookies SET quantity=? WHERE
cookies.cookie_id = ?'] [parameters: (-8, 2)]
```

We got an IntegrityError because we didn’t have enough dark chocolate chip cookies to ship the order. This actually breaks our current session. If we attempt to issue any more statements via the session such as a query to get the list of cookies, we’ll get the output shown in Example 8-12.

Example 8-12. Query on an errored session

```python
print(session.query(Cookie.cookie_name, Cookie.quantity).all())

InvalidRequestError                       Traceback (most recent call last)
<ipython-input-8-90b93364fb2d> in <module>()
 ---> 1 print(session.query(Cookie.cookie_name, Cookie.quantity).all())

...

InvalidRequestError: This Session's transaction has been rolled back due to a
previous exception during flush. To begin a new transaction with this Session,
first issue Session.rollback(). Original exception was: (sqlite3.IntegrityError)
CHECK constraint failed: quantity_positive [SQL: u'UPDATE cookies SET
quantity=? WHERE cookies.cookie_id = ?'] [parameters: ((1, 1), (-8, 2))]
```

Trimming out all the traceback details, an InvalidRequestError exception is raised due to the previous IntegrityError. To recover from this session state, we need to manually roll back the transaction. The message shown in Example 8-12 can be a bit confusing because it says “This Session’s transaction has been rolled back,” but it does point you toward performing the roll back manually. The rollback() method on the session will restore our session to a working station. Whenever we encounter an exception, we want to issue a rollback():

```python
session.rollback()
print(session.query(Cookie.cookie_name, Cookie.quantity).all())
```

This code will output normally with the data we expect:

```python
[(u'chocolate chip', 3), (u'dark chocolate chip', 1)]
```

With our session working normally, now we can use a try/except block to ensure that if the IntegrityError exception occurs a message is printed and transaction is rolled back so our application will continue to function normally, as shown in Example 8-13.

Example 8-13. Transactional ship_it
```python
from sqlalchemy.exc import IntegrityError 1
def ship_it(order_id):
    order = session.query(Order).get(order_id)
    for li in order.line_items:
        li.cookie.quantity = li.cookie.quantity - li.quantity
        session.add(li.cookie)
    order.shipped = True
    session.add(order)
    try:
        session.commit()
        print("shipped order ID: {}".format(order_id))
    except IntegrityError as error: 2
        print('ERROR: {!s}'.format(error.orig))
        session.rollback() 3
```

* Importing the IntegrityError so we can handle its exception.

* Committing the transaction if no exceptions occur.

* Rolling back the transaction if an exception occurs.

Let’s rerun our transaction-based ship_it on the second order as shown here:

```python
ship_it(2)

ERROR: CHECK constraint failed: quantity_positive
```

The program doesn’t get stopped by the exception, and prints us the exception message without the traceback. Let’s check the inventory like we did in Example 8-12 to make sure that it didn’t mess up our inventory with a partial shipment:

```python
[(u'chocolate chip', 3), (u'dark chocolate chip', 1)]
```

Excellent! Our transactional function didn’t mess up our inventory or crash our application. We also didn’t have to do a lot of coding to manually roll back the statements that did succeed. As you can see, transactions can be really useful in situations like this, and can save you a lot of manual coding.

In this chapter, we saw how to handle exceptions in both single statements and groups of statements. By using a normal try/except block on a single statements, we can prevent our application from crashing in case a database statement error occurs. We also looked at the session transaction to avoid inconsistent databases, and application crashes in groups of statements. In the next chapter, we’ll learn how to test our code to ensure it behaves the way we expect.






