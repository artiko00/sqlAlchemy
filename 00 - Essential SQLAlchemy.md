# Introduction to SQLAlchemy

SQLAlchemy is a library used to interact with a wide variety of databases. It enables you to create data models and queries in a manner that feels like normal Python classes and statements. Created by Mike Bayer in 2005, SQLAlchemy is used by many companies great and small, and is considered by many to be the de facto way of working with relational databases in Python.

It can be used to connect to most common databases such as Postgres, MySQL, SQLite, Oracle, and many others. It also provides a way to add support for other relational databases as well. Amazon Redshift, which uses a custom dialect of PostgreSQL, is a great example of database support added by the community.

In this chapter, we’ll explore why we need SQLAlchemy, learn about its two major modes, and get connected to a database

## Why Use SQLAlchemy?

The top reason to use SQLAlchemy is to abstract your code away from the underlying database and its associated SQL peculiarities. SQLAlchemy leverages powerful common statements and types to ensure its SQL statements are crafted efficiently and properly for each database type and vendor without you having to think about it. This makes it easy to migrate logic from Oracle to PostgreSQL or from an application database to a data warehouse. It also helps ensure that database input is sanitized and properly escaped prior to being submitted to the database. This prevents common issues like SQL injection attacks.

SQLAlchemy also provides a lot of flexibility by supplying two major modes of usage: SQL Expression Language (commonly referred to as Core) and ORM. These modes can be used separately or together depending on your preference and the needs of your application.

### SQLAlchemy Core and the SQL Expression Language

The SQL Expression Language is a Pythonic way of representing common SQL statements and expressions, and is only a mild abstraction from the typical SQL language. It is focused on the actual database schema; however, it is standardized in such a way that it provides a consistent language across a large number of backend databases. The SQL Expression Language also acts as the foundation for the SQLAlchemy ORM.

### ORM

The SQLAlchemy ORM is similar to many other object relational mappers (ORMs) you may have encountered in other languages. It is focused around the domain model of the application and leverages the Unit of Work pattern to maintain object state. It also provides a high-level abstraction on top of the SQL Expression Language that enables the user to work in a more idiomatic way. You can mix and match use of the ORM with the SQL Expression Language to create very powerful applications. The ORM leverages a declarative system that is similar to the active-record systems used by many other ORMs such as the one found in Ruby on Rails.

While the ORM is extremely useful, you must keep in mind that there is a difference between the way classes can be related, and how the underlying database relationships work. We’ll more fully explore the ways in which this can affect your implementation in Chapter 6.

## Choosing Between SQLAlchemy Core and ORM

Before you begin building applications with SQLAlchemy, you will need to decide if you are going to primarly use the ORM or Core. The choice of using SQLAlchemy Core or ORM as the dominant data access layer for an application often comes down to a few factors and personal preference.

The two modes use slightly different syntax, but the biggest difference between Core and ORM is the view of data as schema or business objects. SQLAlchemy Core has a schema-centric view, which like traditional SQL is focused around tables, keys, and index structures. SQLAlchemy Core really shines in data warehouse, reporting, analysis, and other scenarios where being able to tightly control the query or operating on unmodeled data is useful. The strong database connection pool and result-set optimizations are perfectly suited to dealing with large amounts of data, even in multiple databases.

However, if you intend to focus more on a domain-driven design, the ORM will encapsulate much of the underlying schema and structure in metadata and business objects. This encapsulation can make it easy to make database interactions feel more like normal Python code. Most common applications lend themselves to being modeled in this way. It can also be a highly effective way to inject domain-driven design into a legacy application or one with raw SQL statements sprinkled throughout. Microservices also benefit from the abstraction of the underlying database, allowing the developer to focus on just the process being implemented.

However, because the ORM is built on top of SQLAlchemy Core, you can use its ability to work with services like Oracle Data Warehousing and Amazon Redshift in the same manner that it interoperates with MySQL. This makes it a wonderful compliment to the ORM when you need to combine business objects and warehoused data.

Here’s a quick checklist to help you decide which option is best for you:

- If you are working with a framework that already has an ORM built in, but want to add more powerful reporting, use Core.

- If you want to view your data in a more schema-centric view (as used in SQL), use Core.

- If you have data for which business objects are not needed, use Core.

- If you view your data as business objects, use ORM.

- If you are building a quick prototype, use ORM.

- If you have a combination of needs that really could leverage both business objects and other data unrelated to the problem domain, use both!

Now that you know how SQLAlchemy is structured and the difference between Core and ORM, we are ready to install and start using SQLAlchemy to connect to a database.

## Installing SQLAlchemy and Connecting to a Database

SQLAlchemy can be used with Python 2.6, Python 3.3, and Pypy 2.1 or greater. I recommend using pip to perform the install with the command pip install sqlalchemy. It’s worth noting that it can also be installed with easy_install and distutils; however, pip is the more straightforward method. During the install, SQLAlchemy will attempt to build some C extensions, which are leveraged to make working with result sets fast and more memory efficient. If you need to disable these extensions due to the lack of a compiler on the system you are installing on, you can use --global-option=--without-cextensions. Note that using SQLAlchemy without C extensions will adversely affect performance, and you should test your code on a system with the C extensions prior to optimizing it.

### Installing Database Drivers

By default, SQLAlchemy will support SQLite3 with no additional drivers; however, an additional database driver that uses the standard Python DBAPI (PEP-249) specification is needed to connect to other databases. These DBAPIs provide the basis for the dialect each database server speaks, and often enable the unique features seen in different database servers and versions. While there are multiple DBAPIs available for many of the databases, the following instructions focus on the most common:

#### PostgreSQL
Psycopg2 provides wide support for PostgreSQL versions and features and can be installed with pip install psycopg2.

#### MySQL
PyMySQL is my preferred Python library for connecting to a MySQL database server. It can be installed with a pip install pymysql. MySQL support in SQLAlchemy requires MySQL version 4.1 and higher due to the way passwords worked prior to that version. Also, if a particular statement type is only available in a certain version of MySQL, SQLAlchemy does not provide a method to use those statements on versions of MySQL where the statement isn’t available. It’s important to review the MySQL documentation if a particular component or function in SQLAlchemy does not seem to work in your environment.

#### Others
SQLAlchemy can also be used in conjunction with Drizzle, Firebird, Oracle, Sybase, and Microsoft SQL Server. The community has also supplied external dialects for many other databases like IBM DB2, Informix, Amazon Redshift, EXASolution, SAP SQL Anywhere, Monet, and many others. Creating an additional dialect is well supported by SQLAlchemy, and Chapter 7 will examine the process of doing just that.

Now that we have SQLAlchemy and a DBAPI installed, let’s actually build an engine to connect to a database.

### Connecting to a Database

To connect to a database, we need to create a SQLAlchemy engine. The SQLAlchemy engine creates a common interface to the database to execute SQL statements. It does this by wrapping a pool of database connections and a dialect in such a way that they can work together to provide uniform access to the backend database. This enables our Python code not to worry about the differences between databases or DBAPIs.

SQLAlchemy provides a function to create an engine for us given a connection string and optionally some additional keyword arguments. A connection string is a specially formatted string that provides:

* Database type (Postgres, MySQL, etc.)

* Dialect unless the default for the database type (Psycopg2, PyMySQL, etc.)
 
* Optional authentication details (username and password)
 
* Location of the database (file or hostname of the database server)
 
* Optional database server port
 
* Optional database name

SQLite database connections strings have us represent a specific file or a storage location. Example P-1 defines a SQLite database file named cookies.db stored in the current directory via a relative path in the second line, an in-memory database on the third line, and a full path to the file on the fourth (Unix) and fifth (Windows) lines. On Windows, the connection string would look like engine4; the \\ are required for proper string escaping unless you use a raw string (r'').


```python
from sqlalchemy import create_engine
engine = create_engine('sqlite:///cookies.db')
engine2 = create_engine('sqlite:///:memory:')
engine3 = create_engine('sqlite:////home/cookiemonster/cookies.db')
engine4 = create_engine('sqlite:///c:\\Users\\cookiemonster\\cookies.db')
```

Let’s create an engine for a local PostgreSQL database named mydb. We’ll start by importing the create_engine function from the base sqlalchemy package. Next, we’ll use that function to construct an engine instance. In Example P-2, you’ll notice that I use postgresql+psycopg2 as the engine and dialect components of the connection string, even though using only postgres will work. This is because I prefer to be explicit instead of implicit, as recommended in the Zen of Python.

```python
rom sqlalchemy import create_engine
engine = create_engine('postgresql+psycopg2://username:password@localhost:' \
                       '5432/mydb')
```

Now let’s look at a MySQL database on a remote server. You’ll notice, in Example P-3, that after the connection string we have a keyword parameter, pool_recycle, to define how often to recycle the connections.

```python
from sqlalchemy import create_engine
engine = create_engine('mysql+pymysql://cookiemonster:chocolatechip'
                       '@mysql01.monster.internal/cookies', pool_recycle=3600)
```

Some optional keywords for the create_engine function are:

echo

    This will log the actions processed by the engine, such as SQL statements and their parameters. It defaults to false.

encoding

    This defines the string encoding used by SQLAlchemy. It defaults to utf-8, and most DBAPIs support this encoding by default. This does not define the encoding type used by the backend database itself.

isolation_level

    This instructs SQLAlchemy to use a specific isolation level. For example, PostgreSQL with Psycopg2 has READ COMMITTED, READ UNCOMMITTED, REPEATABLE READ, SERIALIZABLE, and AUTOCOMMIT available with a default of READ COMMITTED. PyMySQL has the same options with a default of REPEATABLE READ for InnoDB databases.

pool_recycle

    This recycles or times out the database connections at regular intervals. This is important for MySQL due to the connection timeouts we mentioned earlier. It defaults to -1, which means there is no timeout.

Once we have an engine initialized, we are ready to actually open a connection to the database. That is done by calling the connect() method on the engine as shown here:

```python
from sqlalchemy import create_engine
engine = create_engine('mysql+pymysql://cookiemonster:chocolatechip' \
                       '@mysql01.monster.internal/cookies', pool_recycle=3600)
connection = engine.connect()
```

Now that we have a database connection, we can start using either SQLAlchemy Core or the ORM. In Part I, we will begin exploring SQLAlchemy Core and learning how to define and query your database.