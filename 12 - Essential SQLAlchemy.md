# Chapter 2. Working with Data via SQLAlchemy Core

Now that we have tables in our database, let’s start working with data inside of those tables. We’ll look at how to insert, retrieve, and delete data, and follow that with learning how to sort, group, and use relationships in our data. We’ll be using the SQL Expression Language (SEL) provided by SQLAlchemy Core. We’re going to continue using the tables we created in Chapter 1 for our examples in this chapter. Let’s start by learning how to insert data.

## Inserting Data

First, we’ll build an insert statement to put my favorite kind of cookie (chocolate chip) into the cookies table. To do this, we can call the insert() method on the cookies table, and then use the values() statement with keyword arguments for each column that we are filling with data. Example 2-1 does just that.

```python
ins = cookies.insert().values(
    cookie_name="chocolate chip",
    cookie_recipe_url="http://some.aweso.me/cookie/recipe.html",
    cookie_sku="CC01",
    quantity="12",
    unit_cost="0.50"
)
print(str(ins))
```

In Example 2-1, print(str(ins)) shows us the actual SQL statement that will be executed:

```sql
INSERT INTO cookies
    (cookie_name, cookie_recipe_url, cookie_sku, quantity, unit_cost)
VALUES
    (:cookie_name, :cookie_recipe_url, :cookie_sku, :quantity, :unit_cost
```

Our supplied values have been replaced with :column_name in this SQL statement, which is how SQLAlchemy represents parameters displayed via the str() function. Parameters are used to help ensure that our data has been properly escaped, which mitigates security issues such as SQL injection attacks. It is still possible to view the parameters by looking at the compiled version of our insert statement, because each database backend can handle the parameters in a slightly different manner (this is controlled by the dialect). The compile() method on the ins object returns a SQLCompiler object that gives us access to the actual parameters that will be sent with the query via the params attribute:

```python
ins.compile().params
```

This compiles the statement via our dialect but does not execute it, and we access the params attribute of that statement.

This results in:

```python
{
    'cookie_name': 'chocolate chip',
    'cookie_recipe_url': 'http://some.aweso.me/cookie/recipe.html',
    'cookie_sku': 'CC01',
    'quantity': '12',
    'unit_cost': '0.50'
}
```

Now that we have a complete picture of the insert statement and understand what is going to be inserted into the table, we can use the execute() method on our connection to send the statement to the database, which will insert the record into the table (Example 2-2).


```python
result = connection.execute(ins)
```

We can also get the ID of the record we just inserted by accessing the inserted_primary_key attribute:

```python
result.inserted_primary_key
```

Let’s take a quick detour here to discuss what happens when we call the execute() method. When we are building a SQL Expression Language statement like the insert statement we’ve been using so far, it is actually creating a tree-like structure that can be quickly traversed in a descending manner. When we call the execute method, it uses the statement and any other parameters passed to compile the statement with the proper database dialect’s compiler. That compiler builds a normal parameterized SQL statement by walking down that tree. That statement is returned to the execute method, which sends the SQL statement to the database via the connection on which the method was called. The database server then executes the statement and returns the results of the operation.

In addition to having insert as an instance method off a Table object, it is also available as a top-level function for those times that you want to build a statement “generatively” (a step at a time) or when the table may not be initially known. For example, our company might run two separate divisions, each with its own separate inventory tables. Using the insert function shown in Example 2-3 would allow us to use one statement and just swap the tables.

```python
from sqlalchemy import insert
ins = insert(cookies).values(
    cookie_name="chocolate chip",
    cookie_recipe_url="http://some.aweso.me/cookie/recipe.html",
    cookie_sku="CC01",
    quantity="12",
    unit_cost="0.50"
)
```

Notice the table is now the argument to the insert function.

    NOTE
    While insert works equally well as a method of a Table object and as a more generative standalone function, I prefer the generative approach because it more closely mirrors the SQL statements people are accustomed to seeing.

The execute method of the connection object can take more than just statements. It is also possible to provide the values as keyword arguments to the execute method after our statement. When the statement is compiled, it will add each one of the keyword argument keys to the columns list, and it adds each one of their values to the VALUES part of the SQL statement (Example 2-4).

```python
ins = cookies.insert()
result = connection.execute(
    ins,
    cookie_name='dark chocolate chip', 2
    cookie_recipe_url='http://some.aweso.me/cookie/recipe_dark.html',
    cookie_sku='CC02',
    quantity='1',
    unit_cost='0.75'
)
result.inserted_primary_key
```

While this isn’t used often in practice for single inserts, it does provide a good illustration of how a statement is compiled and assembled prior to being sent to the database server. We can insert multiple records at once by using a list of dictionaries with data we are going to submit. Let’s use this knowledge to insert two types of cookies into the cookies table (Example 2-5).

```python
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
result = connection.execute(ins, inventory_list)
```

    NOTE
    The dictionaries in the list must have the exact same keys. SQLAlchemy will compile the statement against the first dictionary in the list, and the statement will fail if subsequent dictionaries are different, as the statement was already built with the prior columns.

Now that we have some data in our cookies table, let’s learn how to query the tables and retrieve that data.

## Querying Data

To begin building a query, we start by using the select function, which is analogous to the standard SQL SELECT statement. Initially, let’s select all the records in our cookies table (Example 2-6).

```python
from sqlalchemy.sql import select
s = select([cookies]) 1
rp = connection.execute(s)
results = rp.fetchall() 2

```

1. Remember we can use str(s) to look at the SQL statement the database will see, which in this case is SELECT cookies.cookie_id, cookies.cookie_name, cookies.cookie_recipe_url, cookies.cookie_sku, cookies.quantity, cookies.unit_cost FROM cookies.

2. This tells rp, the ResultProxy, to return all the rows.

The results variable now contains a list representing all the records in our cookies table:

```python
[(1, u'chocolate chip', u'http://some.aweso.me/cookie/recipe.html', u'CC01',
  12, Decimal('0.50')),
 (2, u'dark chocolate chip', u'http://some.aweso.me/cookie/recipe_dark.html',
  u'CC02', 1, Decimal('0.75')),
 (3, u'peanut butter', u'http://some.aweso.me/cookie/peanut.html', u'PB01',
  24, Decimal('0.25')),
 (4, u'oatmeal raisin', u'http://some.okay.me/cookie/raisin.html', u'EWW01',
  100, Decimal('1.00'))]
```

In the preceding example, I passed a list containing the cookies table. The select method expects a list of columns to select; however, for convenience, it also accepts Table objects and selects all the columns on the table. It is also possible to use the select method on the Table object to do this, as shown in Example 2-7. Again, I prefer seeing it written more like Example 2-6.

```python
from sqlalchemy.sql import select
s = cookies.select()
rp = connection.execute(s)
results = rp.fetchall()
```

Before we go digging any further into queries, we need to know a bit more about ResultProxy objects.

### ResultProxy

A ResultProxy is a wrapper around a DBAPI cursor object, and its main goal is to make it easier to use and manipulate the results of a statement. For example, it makes handling query results easier by allowing access using an index, name, or Column object. Example 2-8 demonstrates all three of these methods. It’s very important to become comfortable using each of these methods to get to the desired column data.

```python
first_row = results[0] 1
first_row[1] 2
first_row.cookie_name 3
first_row[cookies.c.cookie_name] 4
```

1. Get the first row of the ResultProxy from Example 2-7.

2. Access column by index.

3. Access column by name.

4. Access column by Column object.

These all result in u'chocolate chip' and they each reference the exact same data element in the first record of our results variable. This flexibility in access is only part of the power of the ResultProxy. We can also leverage the ResultProxy as an iterable, and perform an action on each record returned without creating another variable to hold the results. For example, we might want to print the name of each cookie in our database (Example 2-9).

```python
rp = connection.execute(s) 1
for record in rp:
    print(record.cookie_name)
```

This returns:

* chocolate chip
* dark chocolate chip
* peanut butter
* oatmeal raisin

In addition to using the ResultProxy as an iterable or calling the fetchall() method, many other ways of accessing data via the ResultProxy are available. In fact, all the result variables in “Inserting Data” were actually ResultProxys. Both the rowcount() and inserted_primary_key() methods we used in that section are just a couple of the other ways to get information from a ResultProxy. You can use the following methods as well to fetch results:

first()
    Returns the first record if there is one and closes the connection.

fetchone()
    Returns one row, and leaves the cursor open for you to make additional fetch calls.

scalar()
    Returns a single value if a query results in a single record with one column.

If you want to see the columns that are available in a result set you can use the keys() method to get a list of the column names. We’ll be using the first, scalar, fetchone, and fetchall methods as well as the ResultProxy as an iterable throughout the remainder of this chapter.

    TIPS FOR GOOD PRODUCTION CODE
    When writing production code, you should follow these guidelines:

    * Use the first method for getting a single record over both the fetchone and scalar methods, because it is clearer to our fellow coders.

    * Use the iterable version of the ResultProxy over the fetchall and fetchone methods. It is more memory efficient and we tend to operate on the data one record at a time.

    * Avoid the fetchone method, as it leaves connections open if you are not careful.

    * Use the scalar method sparingly, as it raises errors if a query ever returns more than one row with one column, which often gets missed during testing.

Every time we queried the database in the preceding examples, all the columns were returned for every record. Often we only need a portion of those columns to perform our work. If the data in these extra columns is large, it can cause our applications to slow down and consume far more memory than it should. SQLAlchemy does not add a bunch of overhead to the queries or ResultProxys; however, accounting for the data you get back from a query is often the first place to look if a query is consuming too much memory. Let’s look at how to limit the columns returned in a query.

### Controlling the Columns in the Query

To limit the fields that are returned from a query, we need to pass the columns we want into the select() method constructor as a list. For example, you might want to run a query, that returns only the name and quantity of cookies, as shown in Example 2-10.

```python
s = select([cookies.c.cookie_name, cookies.c.quantity])
rp = connection.execute(s)
print(rp.keys()) 1
result = rp.first() 2
```

1. Returns the list of columns, which is [u'cookie_name', u'quantity'] in this example (used only for demonstration, not needed for results).

2. Notice this only returns the first result.

(u'chocolate chip', 12),

Now that we can build a simple select statement, let’s look at other things we can do to alter how the results are returned in a select statement. We’ll start with changing the order in which the results are returned.

### Ordering

If you were to look at all the results from Example 2-10 instead of just the first record, you would see that the data is not really in any particular order. However, if we want the list to be returned in a particular order, we can chain an order_by() statement to our select, as shown in Example 2-11.In this case, we want the results to be ordered by the quantity of cookies we have on hand.

```python

s = select([cookies.c.cookie_name, cookies.c.quantity])
s = s.order_by(cookies.c.quantity)
rp = connection.execute(s)
for cookie in rp:
    print('{} - {}'.format(cookie.quantity, cookie.cookie_name))
```

This results in:

```
1 - dark chocolate chip
12 - chocolate chip
24 - peanut butter
100 - oatmeal raisin
```

We saved the select statement into the s variable, used that s variable and added the order_by statement to it, and then reassigned that to the s variable. This is an example of how to compose statements in a generative or step-by-step fashion. This is the same as combining the select and the order_by statements into one line as shown here:

```python
s = select([...]).order_by(...)
```

However, when we have the full list of columns in the select and the order columns in the order_by statement, it exceeds Python’s 79 character per line limit (which was established in PEP8). By using the generative type statement, we can remain under that limit. Throughout the book, we’ll see a few examples where this generative style can introduce additional benefits, such as conditionally adding things to the statement. For now, try to break your statements along those 79 character limits, and it will help make the code more readable.

If you want to sort in reverse or descending order, use the desc() statement. The desc() function wraps the specific column you want to sort in a descending manner, as shown in Example 2-12.


```python
from sqlalchemy import desc
s = select([cookies.c.cookie_name, cookies.c.quantity])
s = s.order_by(desc(cookies.c.quantity)) 1
```

    NOTE
    The desc() function can also be used as a method on a Column object, such as cookies.c.quantity.desc(). However, that can be a bit more confusing to read in long statements, so I always use desc() as a function.

It’s also possible to limit the number of results returned if we only need a certain number of them for our application.

### Limiting

In prior examples, we used the first() or fetchone() methods to get just a single row back. While our ResultProxy gave us the one row we asked for, the actual query ran over and accessed all the results, not just the single record. If we want to limit the query, we can use the limit() function to actually issue a limit statement as part of our query. For example, suppose you only have time to make two batches of cookies, and you want to know which two cookie types you should make. You can use our ordered query from earlier and add a limit statement to return the two cookie types that are most in need of being replenished (Example 2-13).


```python
s = select([cookies.c.cookie_name, cookies.c.quantity])
s = s.order_by(cookies.c.quantity)
s = s.limit(2)
rp = connection.execute(s)
print([result.cookie_name for result in rp]) 1
```

1. Here we are using the iterable capabilities of the ResultsProxy in a list comprehension.

This results in:
```
[u'dark chocolate chip', u'chocolate chip']
```

Now that you know what kind of cookies you need to bake, you’re probably starting to get curious about how many cookies are now left in your inventory. Many databases include SQL functions designed to make certain operations available directly on the database server, such as SUM; let’s explore how to use these functions next.

### Built-In SQL Functions and Labels

SQLAlchemy can also leverage SQL functions found in the backend database. Two very commonly used database functions are SUM() and COUNT(). To use these functions, we need to import the sqlalchemy.sql.func module where they are found. These functions are wrapped around the column(s) on which they are operating. Thus, to get a total count of cookies, you would use something like Example 2-14.


```python
from sqlalchemy.sql import func
s = select([func.sum(cookies.c.quantity)])
rp = connection.execute(s)
print(rp.scalar()) 1
```

Notice the use of scalar, which will return only the leftmost column in the first record.

This results in:
```python

137
```
    NOTE
    I tend to always import the func module, as importing sum directly can cause problems and confusion with Python’s built-in sum function.

Now let’s use the count function to see how many cookie inventory records we have in our cookies table (Example 2-15).

Example 2-15. Counting our inventory records

```
s = select([func.count(cookies.c.cookie_name)])
rp = connection.execute(s)
record = rp.first()
print(record.keys()) 1
print(record.count_1) 2
```
1. This will show us the columns in the ResultProxy.

2. The column name is autogenerated and is commonly <func_name>_<position>.

This results in:
```
[u'count_1']
```
4. This column name is annoying and cumbersome. Also, if we have several counts in a query, we’d have to know the occurrence number in the statement, and incorporate that into the column name, so the fourth count() function would be count_4. This simply is not as explicit and clear as we should be in our naming, especially when surrounded with other Python code. Thankfully, SQLAlchemy provides a way to fix this via the label() function. Example 2-16 performs the same query as Example 2-15; however, it uses label to give us a more useful name to access that column.

Example 2-16. Renaming our count column
```
s = select([func.count(cookies.c.cookie_name).label('inventory_count')]) 1
rp = connection.execute(s)
record = rp.first()
print(record.keys())
print(record.inventory_count)
```
1. Notice that we just use the label() function on the column object we want to change.

This results in:
```
[u'inventory_count']
```

We’ve seen examples of how to restrict the columns or the number of rows returned from the database, so now it’s time to learn about queries that filter data based on criteria we specify.

### Filtering
Filtering queries is done by adding where() statements just like in SQL. A typical where() clause has a column, an operator, and a value or column. It is possible to chain multiple where() clauses together, and they will act like ANDs in traditional SQL statements. In Example 2-17, we’ll find a cookie named “chocolate chip”.

Example 2-17. Filtering by cookie name
```
s = select([cookies]).where(cookies.c.cookie_name == 'chocolate chip')
rp = connection.execute(s)
record = rp.first()
print(record.items()) 1
```
1. Here I’m calling the items() method on the row object, which will give me a list of columns and values.

This results in:
```
[
    (u'cookie_id', 1),
    (u'cookie_name', u'chocolate chip'),
    (u'cookie_recipe_url', u'http://some.aweso.me/cookie/recipe.html'),
    (u'cookie_sku', u'CC01'),
    (u'quantity', 12),
    (u'unit_cost', Decimal('0.50'))
]
```
We can also use a where() statement to find all the cookie names that contain the word “chocolate” (Example 2-18).

Example 2-18. Finding names with chocolate in them
```
s = select([cookies]).where(cookies.c.cookie_name.like('%chocolate%'))
rp = connection.execute(s)
for record in rp.fetchall():
    print(record.cookie_name)
```
This results in:
```
chocolate chip
dark chocolate chip
```
In the where() statement of Example 2-18, we are using the cookies.c.cookie_name column inside of a where() statement as a type of ClauseElement to filter our results. We should take a brief moment and talk more about ClauseElements and the additional capabilities they provide.

### ClauseElements
ClauseElements are just an entity we use in a clause, and they are typically columns in a table; however, unlike columns, ClauseElements come with many additional capabilities. In Example 2-18, we are taking advantage of the like() method that is available on ClauseElements. There are many other methods available, which are listed in Table 2-1. Each of these methods are analogous to a standard SQL statement construct. You’ll find various examples of these used throughout the book.

Table 2-1. ClauseElement methods
|Method|Purpose|
|---|---|
|between(cleft, cright)|Find where the column is between cleft and cright|
|concat(column_two)|Concatenate column with column_two|
|distinct()|Find only unique values for the column|
|in_([list])|Find where the column is in the list|
|is_(None)|Find where the column is None (commonly used for Null checks with None)|
|contains(string)|Find where the column has string in it (case-sensitive)|
|endswith(string)|Find where the column ends with string (case-sensitive)|
|like(string)|Find where the column is like string (case-sensitive)|
|startswith(string)|Find where the column begins with string (case-sensitive)|
|ilike(string)|Find where the column is like string (this is not case-sensitive)|

    NOTE
    There are also negative versions of these methods, such as notlike and notin_(). The only exception to the not<method> naming convention is the isnot() method, which drops the underscore.

If we don’t use one of the methods listed in Table 2-1, then we will have an operator in our where clauses. Most of the operators work as you might expect; however, we should discuss operators in a bit more detail, as there are a few differences.

### Operators
So far, we have only explored where a column was equal to a value or used one of the ClauseElement methods such as like(); however, we can also use many other common operators to filter data. SQLAlchemy provides overloading for most of the standard Python operators. This includes all the standard comparison operators (==, !=, <, >, <=, >=), which act exactly like you would expect in a Python statement. The == operator also gets an additional overload when compared to None, which converts it to an IS NULL statement. Arithmetic operators (\+, -, *, /, and %) are also supported with additional capabilities for database-independent string concatenation, as shown in Example 2-19.

Example 2-19. String concatenation with \+
```s = select([cookies.c.cookie_name, 'SKU-' + cookies.c.cookie_sku])
for row in connection.execute(s):
    print(row)
```
This results in:
```
(u'chocolate chip', u'SKU-CC01')
(u'dark chocolate chip', u'SKU-CC02')
(u'peanut butter', u'SKU-PB01')
(u'oatmeal raisin', u'SKU-EWW01')
```
Another common usage of operators is to compute values from multiple columns. You’ll often do this in applications and reports dealing with financial data or statistics. Example 2-20 shows a common inventory value calculation.

Example 2-20. Inventory value by cookie
```
from sqlalchemy import cast 1
s = select([cookies.c.cookie_name,
          cast((cookies.c.quantity * cookies.c.unit_cost),
               Numeric(12,2)).label('inv_cost')]) 2
for row in connection.execute(s):
    print('{} - {}'.format(row.cookie_name, row.inv_cost))
```
1. Cast() is another function that allows us to convert types. In this case, we will be getting back results such as 6.0000000000, so by casting it, we can make it look like currency. It is also possible to accomplish the same task in Python with print('{} - {:.2f}'.format(row.cookie_name, row.inv_cost)).

2. Notice we are again using the label() function to rename the column. Without this renaming, the column would be named anon_1, as the operation doesn’t result in a name.

This results in:
```
chocolate chip - 6.00
dark chocolate chip - 0.75
peanut butter - 6.00
oatmeal raisin - 100.00
Boolean Operators
```
SQLAlchemy also allows for the SQL Boolean operators AND, OR, and NOT via the bitwise logical operators (&, |, and ~). Special care must be taken when using the AND, OR, and NOT overloads because of the Python operator precedence rules. For instance, & binds more closely than <, so when you write A < B & C < D, what you are actually writing is A < (B&C) < D, when you probably intended to get (A < B) & (C < D). Please use conjunctions instead of these overloads, as they will make your code more expressive.

Often we want to chain multiple where() clauses together in inclusionary and exclusionary manners; this should be done via conjunctions.

### Conjunctions
While it is possible to chain multiple where() clauses together, it’s often more readable and functional to use conjunctions to accomplish the desired effect. The conjunctions in SQLAlchemy are and_(), or_(), and not_(). So if we wanted to get a list of cookies with a cost of less than an amount and above a certain quantity we could use the code shown in Example 2-21.

Example 2-21. Using the and() conjunction

```
from sqlalchemy import and_, or_, not_
s = select([cookies]).where(
    and_(
        cookies.c.quantity > 23,
        cookies.c.unit_cost < 0.40
   )
)
for row in connection.execute(s):
    print(row.cookie_name)
```
The or_() function works as the opposite of and_() and includes results that match either one of the supplied clauses. If we wanted to search our inventory for cookie types that we have between 10 and 50 of in stock or where the name contains chip, we could use the code shown in Example 2-22.

Example 2-22. Using the or() conjunction
```
from sqlalchemy import and_, or_, not_
s = select([cookies]).where(
    or_(
        cookies.c.quantity.between(10, 50),
        cookies.c.cookie_name.contains('chip')
    )
)
for row in connection.execute(s):
    print(row.cookie_name)
```
This results in:
```
chocolate chip
dark chocolate chip
peanut butter
```
The not_() function works in a similar fashion to other conjunctions, and it simply is used to select records where a record does not match the supplied clause. Now that we can comfortably query data, we are ready to move on to updating existing data.

### Updating Data

Much like the insert method we used earlier, there is also an update method with syntax almost identical to inserts, except that it can specify a where clause that indicates which rows to update. Like insert statements, update statements can be created by either the update() function or the update() method on the table being updated. You can update all rows in a table by leaving off the where clause.

For example, suppose you’ve finished baking those chocolate chip cookies that we needed for our inventory. In Example 2-23, we’ll add them to our existing inventory with an update query, and then check to see how many we currently have in stock.

Example 2-23. Updating data
```
from sqlalchemy import update
u = update(cookies).where(cookies.c.cookie_name == "chocolate chip")
u = u.values(quantity=(cookies.c.quantity + 120)) 1
result = connection.execute(u)
print(result.rowcount) 2
s = select([cookies]).where(cookies.c.cookie_name == "chocolate chip")
result = connection.execute(s).first()
for key in result.keys():
    print('{:>20}: {}'.format(key, result[key]))
```
1. Using the generative method of building our statement.

2. Printing how many rows were updated.

This returns:
```
1
           cookie_id: 1
         cookie_name: chocolate chip
   cookie_recipe_url: http://some.aweso.me/cookie/recipe.html
          cookie_sku: CC01
            quantity: 132
           unit_cost: 0.50
```

In addition to updating data, at some point we will want to remove data from our tables. The following section explains how to do that.

### Deleting Data
To create a delete statement, you can use either the delete() function or the delete() method on the table from which you are deleting data. Unlike insert() and update(), delete() takes no values parameter, only an optional where clause (omitting the where clause will delete all rows from the table). See Example 2-24.

Example 2-24. Deleting data
```
from sqlalchemy import delete
u = delete(cookies).where(cookies.c.cookie_name == "dark chocolate chip")
result = connection.execute(u)
print(result.rowcount)

s = select([cookies]).where(cookies.c.cookie_name == "dark chocolate chip")
result = connection.execute(s).fetchall()
print(len(result))
```
This returns:
```
1
0
```

OK, at this point, let’s load up some data using what we already learned for the users, orders, and line_items tables. You can copy the code shown here, but you should also take a moment to play with different ways of inserting the data:
```
customer_list = [
    {
        'username': 'cookiemon',
        'email_address': 'mon@cookie.com',
        'phone': '111-111-1111',
        'password': 'password'
    },
    {
        'username': 'cakeeater',
        'email_address': 'cakeeater@cake.com',
        'phone': '222-222-2222',
        'password': 'password'
    },
    {
        'username': 'pieguy',
        'email_address': 'guy@pie.com',
        'phone': '333-333-3333',
        'password': 'password'
    }
]
ins = users.insert()
result = connection.execute(ins, customer_list)
Now that we have customers, we can start to enter their orders and line items into the system as well:

ins = insert(orders).values(user_id=1, order_id=1)
result = connection.execute(ins)
ins = insert(line_items)
order_items = [
    {
        'order_id': 1,
        'cookie_id': 1,
        'quantity': 2,
        'extended_cost': 1.00
    },
    {
        'order_id': 1,
        'cookie_id': 3,
        'quantity': 12,
        'extended_cost': 3.00
    }
]
result = connection.execute(ins, order_items)
ins = insert(orders).values(user_id=2, order_id=2)
result = connection.execute(ins)
ins = insert(line_items)
order_items = [
    {
        'order_id': 2,
        'cookie_id': 1,
        'quantity': 24,
        'extended_cost': 12.00
    },
    {
        'order_id': 2,
        'cookie_id': 4,
        'quantity': 6,
        'extended_cost': 6.00
    }
]
result = connection.execute(ins, order_items)
```
In Chapter 1, you learned how to define ForeignKeys and relationships, but we haven’t used them to perform any queries up to this point. Let’s take a look at relationships next.

### Joins
Now let’s use the join() and outerjoin() methods to take a look at how to query related data. For example, to fulfill the order placed by the cookiemon user, we need to determine how many of each cookie type were ordered. This requires you to use a total of three joins to get all the way down to the name of the cookies. It’s also worth noting that depending on how the joins are used in a relationship, you might want to rearrange the from part of a statement; one way to accomplish that in SQLAlchemy is via the select_from() clause. With select_from(), we can replace the entire from clause that SQLAlchemy would generate with one we specify (Example 2-25).

Example 2-25. Using join to select from multiple tables
```
columns = [orders.c.order_id, users.c.username, users.c.phone,
           cookies.c.cookie_name, line_items.c.quantity,
           line_items.c.extended_cost]
cookiemon_orders = select(columns)
cookiemon_orders = cookiemon_orders.select_from(orders.join(users).join( 1
                        line_items).join(cookies)).where(users.c.username ==
                            'cookiemon')
result = connection.execute(cookiemon_orders).fetchall()
for row in result:
    print(row)
```
1. Notice we are telling SQLAlchemy to use the relationship joins as the from clause.

This results in:
```
(u'1', u'cookiemon', u'111-111-1111', u'chocolate chip', 2, Decimal('1.00'))
(u'1', u'cookiemon', u'111-111-1111', u'peanut butter', 12, Decimal('3.00'))
```
The SQL looks like this:
```
SELECT orders.order_id, users.username, users.phone, cookies.cookie_name,
line_items.quantity, line_items.extended_cost FROM users JOIN orders ON
users.user_id = orders.user_id JOIN line_items ON orders.order_id =
line_items.order_id JOIN cookies ON cookies.cookie_id = line_items.cookie_id
WHERE users.username = :username_1
```
It is also useful to get a count of orders by all users, including those who do not have any present orders. To do this, we have to use the outerjoin() method, and it requires a bit more care in the ordering of the join, as the table we use the outerjoin() method on will be the one from which all results are returned (Example 2-26).

Example 2-26. Using outerjoin to select from multiple tables
```
columns = [users.c.username, func.count(orders.c.order_id)]
all_orders = select(columns)
all_orders = all_orders.select_from(users.outerjoin(orders)) 1
all_orders = all_orders.group_by(users.c.username)
result = connection.execute(all_orders).fetchall()
for row in result:
    print(row)
```
1. SQLAlchemy knows how to join the users and orders tables because of the foreign key defined in the orders table.

This results in:
```
(u'cakeeater', 1)
(u'cookiemon', 1)
(u'pieguy', 0)

```
So far, we have been using and joining different tables in our queries. However, what if we have a self-referential table like a table of employees and their bosses? To make this easy to read and understand, SQLAlchemy uses aliases.

### Aliases
When using joins, it is often necessary to refer to a table more than once. In SQL, this is accomplished by using aliases in the query. For instance, suppose we have the following (partial) schema that tracks the reporting structure within an organization:
```
employee_table = Table(
    'employee', metadata,
    Column('id', Integer, primary_key=True),
    Column('manager', None, ForeignKey('employee.id')),
    Column('name', String(255)))
```
Now suppose we want to select all the employees managed by an employee named Fred. In SQL, we might write the following:
```
SELECT employee.name
FROM employee, employee AS manager
WHERE employee.manager_id = manager.id
    AND manager.name = 'Fred'
```

SQLAlchemy also allows the use of aliasing selectables in this type of situation via the alias() function or method:

```
>>> manager = employee_table.alias('mgr')
>>> stmt = select([employee_table.c.name],
...               and_(employee_table.c.manager_id==manager.c.id,
...                    manager.c.name=='Fred'))
>>> print(stmt)
SELECT employee.name
FROM employee, employee AS mgr
WHERE employee.manager_id = mgr.id AND mgr.name = ?
```
SQLAlchemy can also choose the alias name automatically, which is useful for guaranteeing that there are no name collisions:
```
>>> manager = employee_table.alias()
>>> stmt = select([employee_table.c.name],
...               and_(employee_table.c.manager_id==manager.c.id,
...                    manager.c.name=='Fred'))
>>> print(stmt)
SELECT employee.name
FROM employee, employee AS employee_1
WHERE employee.manager_id = employee_1.id AND employee_1.name = ?
```
It’s also useful to be able to group data when we are looking to report on data, so let’s look into that next.

### Grouping
When using grouping, you need one or more columns to group on and one or more columns that it makes sense to aggregate with counts, sums, etc., as you would in normal SQL. Let’s get an order count by customer (Example 2-27).

Example 2-27. Grouping data
```
columns = [users.c.username, func.count(orders.c.order_id)] 1
all_orders = select(columns)
all_orders = all_orders.select_from(users.outerjoin(orders))
all_orders = all_orders.group_by(users.c.username) 2
result = connection.execute(all_orders).fetchall()
for row in result:
    print(row)
```
1. Aggregation via count

2. Grouping by the nonaggregated included column

This results in:
```
(u'cakeeater', 1)
(u'cookiemon', 1)
(u'pieguy', 0)
```
We’ve shown the generative building of statements throughout the previous examples, but I want to focus on it specifically for a moment.

### Chaining
We’ve used chaining several times throughout this chapter, and just didn’t acknowledge it directly. Where query chaining is particularly useful is when you are applying logic when building up a query. So if we wanted to have a function that got a list of orders for us it might look like Example 2-28.

Example 2-28. Chaining
```
def get_orders_by_customer(cust_name):
    columns = [orders.c.order_id, users.c.username, users.c.phone,
               cookies.c.cookie_name, line_items.c.quantity,
               line_items.c.extended_cost]
    cust_orders = select(columns)
    cust_orders = cust_orders.select_from(
        users.join(orders).join(line_items).join(cookies))
    cust_orders = cust_orders.where(users.c.username == cust_name)
    result = connection.execute(cust_orders).fetchall()
    return result

get_orders_by_customer('cakeeater')
```
This results in:
```
[(u'2', u'cakeeater', u'222-222-2222', u'chocolate chip', 24, Decimal('12.00')),
 (u'2', u'cakeeater', u'222-222-2222', u'oatmeal raisin', 6, Decimal('6.00'))]
```
However, what if we wanted to get only the orders that have shipped or haven’t shipped yet? We’d have to write additional functions to support those additional desired filter options, or we can use conditionals to build up query chains. Another option we might want is whether or not to include details. This ability to chain queries and clauses together enables quite powerful reporting and complex query building (Example 2-29).

Example 2-29. Conditional chaining
```
def get_orders_by_customer(cust_name, shipped=None, details=False):
    columns = [orders.c.order_id, users.c.username, users.c.phone]
    joins = users.join(orders)
    if details:
        columns.extend([cookies.c.cookie_name, line_items.c.quantity,
                       line_items.c.extended_cost])
        joins = joins.join(line_items).join(cookies)
    cust_orders = select(columns)
    cust_orders = cust_orders.select_from(joins)
    cust_orders = cust_orders.where(users.c.username == cust_name)
    if shipped is not None:
        cust_orders = cust_orders.where(orders.c.shipped == shipped)
    result = connection.execute(cust_orders).fetchall()
    return result

get_orders_by_customer('cakeeater') 1

get_orders_by_customer('cakeeater', details=True) 2

get_orders_by_customer('cakeeater', shipped=True) 3

get_orders_by_customer('cakeeater', shipped=False) 4

get_orders_by_customer('cakeeater', shipped=False, details=True) 5
```
1. Gets all orders.

2. Gets all orders with details.

3. Gets only orders that have shipped.

4. Gets orders that haven’t shipped yet.

5. Gets orders that haven’t shipped yet with details.

This results in:
```
[(u'2', u'cakeeater', u'222-222-2222')]

[(u'2', u'cakeeater', u'222-222-2222', u'chocolate chip', 24, Decimal('12.00')),
 (u'2', u'cakeeater', u'222-222-2222', u'oatmeal raisin', 6, Decimal('6.00'))]

[]

[(u'2', u'cakeeater', u'222-222-2222')]

[(u'2', u'cakeeater', u'222-222-2222', u'chocolate chip', 24, Decimal('12.00')),
 (u'2', u'cakeeater', u'222-222-2222', u'oatmeal raisin', 6, Decimal('6.00'))]
```

So far in this chapter, we’ve used the SQL Expression Language for all the examples; however, you might be wondering if you can execute standard SQL statements as well. Let’s take a look.

### Raw Queries
It is also possible to execute raw SQL statements or use raw SQL in part of a SQLAlchemy Core query. It still returns a ResultProxy, and you can continue to interact with it just as you would a query built using the SQL Expression syntax of SQLAlchemy Core. I encourage you to only use raw queries and text when you must, as it can lead to unforeseen results and security vulnerabilities. First, we’ll want to execute a simple select statement (Example 2-30).

Example 2-30. Full raw queries
```
result = connection.execute("select * from orders").fetchall()
print(result)
```
This results in:
```
[(1, 1, 0), (2, 2, 0)]
```
While I rarely use a full raw SQL statement, I will often use small text snippets to help make a query clearer. Example 2-31 is of a raw SQL where clause using the text() function.

Example 2-31. Partial text query
```
from sqlalchemy import text
stmt = select([users]).where(text("username='cookiemon'"))
print(connection.execute(stmt).fetchall())
```
This results in:
```
[(1, None, u'cookiemon', u'mon@cookie.com', u'111-111-1111', u'password',
  datetime.datetime(2015, 3, 30, 13, 48, 25, 536450),
  datetime.datetime(2015, 3, 30, 13, 48, 25, 536457))
]
```
Now you should have an understanding of how to use the SQL Expression Language to work with data in SQLAlchemy. We explored how to create, read, update, and delete operations. This is a good point to stop and explore a bit on your own. Try to create more cookies, orders, and line items, and use query chains to group them by order and user. Now that you’ve explored a bit more and hopefully broken something, let’s investigate how to react to exceptions raised in SQLAlchemy, and how to use transactions to group statements that must succeed or fail as a group.