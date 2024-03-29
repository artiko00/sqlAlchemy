# Chapter 12. Building Migrations

In Chapter 11, we initialized and configured the Alembic migration environment to prepare for adding data classes to our application and to create migrations to add them to our database. We’re going to explore how to use autogenerate for adding tables, and how to handcraft migrations to accomplish things that autogenerate cannot do. It’s always a good idea to start with an empty migration, so let’s begin there as it gives us a clean starting point for our migrations.

## Generating a Base Empty Migration

To create the base empty migration, make sure you are in the CH12/ folder of the example code for this book. We’ll create an empty migration using this command:

```python
# alembic revision -m "Empty Init" 1
   Generating ch12/alembic/versions/8a8a9d067_empty_init.py ... done
```

Run the alembic revision command and add the message (-m) "Empty Init" to the migration

This will create a migration file in the alembic/versions/ subfolder. The filenames will always begin with a hash that represents the revision ID and then whatever message you supply. Let’s look inside this file:

```python
"""Empty Init 1

Revision ID: 8a8a9d067
Revises:
Create Date: 2015-09-13 20:10:05.486995

"""

# revision identifiers, used by Alembic.
revision = '8a8a9d067' 2
down_revision = None 3
branch_labels = None 4
depends_on = None 5

from alembic import op
import sqlalchemy as sa


def upgrade():
    pass


def downgrade():
    pass
```

* The migration message we specified

* The Alembic revision ID

* The previous revision used to determine how to downgrade

* The branch associated with this migration

* Any migrations that this one depends on

The file starts with a header that contains the message we supplied (if any), the revision ID, and the date and time at which it was created. Following that is the identifiers section, which explains what migration this is and any details about what it downgrades to or depends on, or any branch associated with this migration. Typically, if you are making data class changes in a branch of your code, you’ll also want to branch your Alembic migrations as well.

Next, there is an upgrade method that would contain any Python code required to perform the changes to the database when applying this migration. Finally, there is a downgrade method that contains the code required to undo this migration and restore the database to the prior migration step.

Because we don’t have any data classes and have made no changes, both our upgrade and downgrade methods are empty. So running this migration will have no effect, but it will provide a great foundation for our migration chain. To run all the migrations from whatever the current database state is to the highest Alembic migration, we execute the following command:

```python
# alembic upgrade head 1
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 8a8a9d067, Empty Init
```
* Upgrades the database to the head (newest) revision.

In the preceding output, it will list every revision it went through to get to the newest revision. In our case, we only have one, the Empty Init migration, which it ran last.

With our base in place, we can begin to add our user data class to our application. Once we add the data class, we can build an autogenerated migration and use it to create the associated table in the database.

## Autogenerating a Migration

Now let’s add our user data class to app/db.py. It’s going to be the same Cookie class we’ve been using in the ORM section. We’re going to add to the imports from sqlalchemy, including the Column and column types that are used in the Cookie class, and then add the Cookie class itself, as shown in Example 12-1.

Example 12-1. Updated app/db.py
```python
from sqlalchemy import create_engine, Column, Integer, Numeric, String

from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///alembictest.db')

Base = declarative_base()


class Cookie(Base):
    __tablename__ = 'cookies'

    cookie_id = Column(Integer, primary_key=True)
    cookie_name = Column(String(50), index=True)
    cookie_recipe_url = Column(String(255))
    cookie_sku = Column(String(55))
    quantity = Column(Integer())
    unit_cost = Column(Numeric(12, 2))
```

With our class added, we are ready to create a migration to add this table to our database. This is a very straightforward migration, and we can take advantage of Alembic’s ability to autogenerate the migration (Example 12-2).

Example 12-2. Autogenerating the migration
```python
# alembic revision --autogenerate -m "Added Cookie model" 1
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'cookies'
INFO  [alembic.autogenerate.compare] Detected added index 'ix_cookies_cookie_name'
      on '['cookie_name']'
  Generating ch12/alembic/versions/34044511331_added_cookie_model.py ... done
```
* Autogenerates a migration with the message “Added Cookie model.”

So when we run the autogeneration command, Alembic inspects the metadata of our SQLAlchemy Base and then compares that to the current database state. In Example 12-2, it detected that we added a table named cookies (based on the __tablename__) and an index on the cookie_name field of that table. It marks the differences as changes, and adds logic to create them in the migration file. Let’s investigate the migration file it created in Example 12-3.

Example 12-3. The cookies migration file
```python
"""Added Cookie model

Revision ID: 34044511331
Revises: 8a8a9d067
Create Date: 2015-09-14 20:37:25.924031

"""

# revision identifiers, used by Alembic.
revision = '34044511331'
down_revision = '8a8a9d067'
branch_labels = None
depends_on = None

from alembic import op
import sqlalchemy as sa


def upgrade():
    ### commands auto generated by Alembic - please adjust! ###
    op.create_table('cookies', 1
    sa.Column('cookie_id', sa.Integer(), nullable=False),
    sa.Column('cookie_name', sa.String(length=50), nullable=True),
    sa.Column('cookie_recipe_url', sa.String(length=255), nullable=True),
    sa.Column('cookie_sku', sa.String(length=55), nullable=True),
    sa.Column('quantity', sa.Integer(), nullable=True),
    sa.Column('unit_cost', sa.Numeric(precision=12, scale=2), nullable=True),
    sa.PrimaryKeyConstraint('cookie_id') 2
    )
    op.create_index(op.f('ix_cookies_cookie_name'), 'cookies', ['cookie_name'],
                    unique=False) 3
    ### end Alembic commands ###


def downgrade():
    ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f('ix_cookies_cookie_name'), table_name='cookies') 4
    op.drop_table('cookies') 5
    ### end Alembic commands ###
```

* Uses Alembic’s create_table method to add our cookies table.
* Adds the primary key on the cookie_id column.
* Uses Alembic’s create_index method to add an index on the cookie_name column.
* Drops the cookie_name index with Alembic’s drop_index method.
* Drops the cookies table.

In the upgrade method of Example 12-3, the syntax of the create_table method is the same as the Table constructor we used with SQLAlchemy Core in Chapter 1: it’s the table name followed by all the table’s columns, and any constraints. The create_index method is the same as the Index constructor from Chapter 1 as well.

Finally, the downgrade method of Example 12-3 uses Alembic’s drop_index and drop_table methods in the proper order to ensure that both the index and the table are removed.

Let’s run this migration using the same command we did for running the empty migration:

```python
# alembic upgrade head
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 8a8a9d067 -> 34044511331,
Added Cookie model
```

After running this migration, we can take a peek in the database to make sure the changes happened:

```python
# sqlite3 alembictest.db 1
SQLite version 3.8.5 2014-08-15 22:37:57
Enter ".help" for usage hints.
sqlite> .tables 2
alembic_version  cookies
sqlite> .schema cookies 3
CREATE TABLE cookies (
	cookie_id INTEGER NOT NULL,
	cookie_name VARCHAR(50),
	cookie_recipe_url VARCHAR(255),
	cookie_sku VARCHAR(55),
	quantity INTEGER,
	unit_cost NUMERIC(12, 2),
	PRIMARY KEY (cookie_id)
);
CREATE INDEX ix_cookies_cookie_name ON cookies (cookie_name);
```

* Accessing the database via the sqlite3 command.
* Listing the tables in the database.
* Printing the schema for the cookies table.

Excellent! Our migration created our table and our index perfectly. However, there are some limitations to what Alembic autogenerate can do. Table 12-1 shows a list of common schema changes that autogenerate can detect, and after that Table 12-2 shows some schema changes that autogenerate cannot detect or detects improperly.


|Table 12-1. Schema changes that autogenerate can detect|
|Schema element|Changes|
|---|---|
|Table|Additions and removals|
|Column|Additions, removals, change of nullable status on columns|
|Index|Basic changes in indexes and explicitly named unique constraints, support for autogenerate of indexes and unique constraints|
|Keys|Basic renames|




|Table 12-2. Schema changes that autogenerate cannot detect|
|Schema element	|Actions|
|---|---|
|Tables|Name changes|
|Column|Name changes|
|Constraints|Constraints without an explicit name|
|Types|Types like ENUM that aren’t directly supported on a database backend|

In addition to the actions that autogenerate can and cannot support, there are some optionally supported capabilities that require special configuration or custom code to implement. Some examples of these are changes to a column type or changes in a server default. If you want to know more about autogeneration’s capabilities and limitations, you can read more at the Alembic autogenerate documentation.

## Building a Migration Manually

Alembic can’t detect a table name change, so let’s learn how to do that without autogeneration and change the cookies table to be the new_cookies table. We need to start by changing the __tablename__ in our Cookie class in app/db.py. We’ll change it to look like the following:

```python
class Cookie(Base):
    __tablename__ = 'new_cookies'
```

Now we need to create a new migration that we can edit with the message “Renaming cookies to new_cookies”:

```python
# alembic revision -m "Renaming cookies to new_cookies"
  Generating ch12/alembic/versions/2e6a6cc63e9_renaming_cookies_to_new_cookies.py
... done
```
With our migration created, let’s edit the migration file and add the rename operation to the upgrade and downgrade methods as shown here:

```python
def upgrade():
    op.rename_table('cookies', 'new_cookies')


def downgrade():
    op.rename_table('new_cookies', 'cookies')
```

Now, we’re ready to run our migration with the alembic upgrade command:

```python
# alembic upgrade head
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 34044511331 -> 2e6a6cc63e9,
      Renaming cookies to new_cookies
```

This will rename our table in the database to new_cookies. Let’s confirm that has happened with the sqlite3 command:

```python
± sqlite3 alembictest.db
SQLite version 3.8.5 2014-08-15 22:37:57
Enter ".help" for usage hints.
sqlite> .tables
alembic_version  new_cookies
```

As we expected, there is no longer a cookies table, as it has been replaced by new_cookies. In addition to rename_table, Alembic has many operations that you can use to help with your database changes, as shown in Table 12-3.

|Table 12-3. Alembic operations|
|---|---|
|Operation|Used for|
|add_column|Adds a new column|
|alter_column|Changes a column type, server default, or name|
|create_check_constraint|Adds a new CheckConstraint|
|create_foreign_key|Adds a new ForeignKey|
|create_index|Adds a new Index|
|create_primary_key|Adds a new PrimaryKey|
|create_table|Adds a new table|
|create_unique_constraint|Adds a new UniqueConstraint|
|drop_column|Removes a column|
|drop_constraint|Removes a constraint|
|drop_index|Drops an index|
|drop_table|Drops a table|
|execute|Run a raw SQL statement|
|rename_table|Renames a table|

NOTE
While Alembic supports all the operations in Table 12-3, they are not supported on every backend database. For example, alter_column does not work on SQLite databases because SQLite doesn’t support altering a column in any way. SQLite also doesn’t support dropping a column.

In this chapter, we’ve seen how to autogenerate and handcraft migrations to make changes to our database in a repeatable manner. We also learned how to apply migrations with the alembic upgrade command. Remember, in addition to Alembic operations, we can use any valid Python code in the migration to help us accomplish our goals. In the next chapter, we’ll explore how to perform downgrades and further control Alembic.
