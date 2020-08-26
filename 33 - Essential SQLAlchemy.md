# Chapter 13. Controlling Alembic

In the previous chapter, we learned how to create and apply migrations, and in this chapter we’re going to discuss how to further control Alembic. We’ll explore how to learn the current migration level of the database, how to downgrade from a migration, and how to mark the database at a certain migration level.

## Determining a Database’s Migration Level

Before performing migrations, you should double-check to make sure what migrations have been applied to the database. You can determine what the last migration applied to the database is by using the alembic current command. It will return the revision ID of the current migration, and tell you whether it is the latest migration (also known as the head). Let’s run the alembic current command in the CH12/ folder of the sample code for this book:

```python
# alembic current
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
2e6a6cc63e9 (head) 1
```

* The last migration applied to the database

While this output shows that we are on migration 2e6a6cc63e9, it also tells us we are on the head or latest migration. This was the migration that changed the cookies table to new_cookies. We can confirm that with the alembic history command, which will show us a list of our migrations. Example 13-1 shows what the Alembic history looks like.

Example 13-1. Our migration history
```python
# alembic history
34044511331 -> 2e6a6cc63e9 (head), Renaming cookies to new_cookies
8a8a9d067 -> 34044511331, Added Cookie model
<base> -> 8a8a9d067, Empty Init
```

The output from Example 13-1 shows us every step from our initial empty migration to our current migration that named the cookies table. I think we can all agree, new_cookies was a terrible name for a table, especially when we change cookies again and have new_new_cookies. To fix that and revert to the previous name, let’s learn how to downgrade a migration.

## Downgrading Migrations

To downgrade migrations, we need to choose the revision ID for the migration that we want to go back to. For example, if you wanted to return all the way back to the initial empty migration, you would choose revision ID 8a8a9d067. We mostly want to undo the table rename, so for Example 13-2, let’s revert to revision ID 34044511331 using the alembic downgrade command.

Example 13-2. Downgrading the rename cookies migration
```python
# alembic downgrade 34044511331
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running downgrade 2e6a6cc63e9 -> 34044511331,
      Renaming cookies to new_cookies 1
```

* Downgrade line

As we can see from the downgrade line in the output of Example 13-2, it has reverted the rename. We can check the tables with the same SQLite .tables command we used in Chapter 12. We can see the output of that command here:

```python
# sqlite3 alembictest.db
SQLite version 3.8.5 2014-08-15 22:37:57
Enter ".help" for usage hints.
sqlite> .tables
alembic_version  cookies 1
```

* Our table is back to being named cookies.

We can see the table has been returned to it’s original name of cookies. Now let’s use alembic current to see what migration level the database is at:

```python
± alembic current
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
34044511331 1
```

* The revision ID for the add cookies model migration

We can see that we’re back to the migration we specified in the downgrade command. We also need to update our application code to use the cookies table name as well, just as we did in Chapter 12, if we want to restore the application to a working state. You can also see that we are not at the latest migration, as head is not on that last line. This creates an interesting issue that we need to handle. If we no longer want to use the rename cookies to new_cookies migration again, we can just delete it from our alembic/versions/ directory. If you leave it behind it will get run the next time alembic upgrade head is run, and this can cause disastrous results. We’re going to leave it in place so that we can explore how to mark the database as being at a certain migration level.

## Marking the Database Migration Level

When we want to do something like skip a migration or restore a database, it is possible that the database believes we are at a different migration than where we really are. We will want to explicitly mark the database as being a specific migration level to correct the issue. In Example 13-3, we are going to mark the database as being at revision ID 2e6a6cc63e9 with the alembic stamp command.

Example 13-3. Marking the database migration level
```python
# alembic stamp 2e6a6cc63e9
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running stamp_revision 34044511331
       -> 2e6a6cc63e9
```

We can see from Example 13-3 that it stamped revision 34044511331 as the current database migration level. However, if we look at the database tables, we’ll still see the cookies table is present. Stamping the database migration level does not actually run the migrations, it merely updates the Alembic table to reflect the migration level we supplied in the command. This effectively skips applying the 34044511331 migration.

WARNING
If you skip a migration like this, it will only apply to the current database. If you point your Alembic migration environment to a different database by changing the sqlalchemy.url, and that new database is below the skipped migration’s level or is blank, running alembic upgrade head will apply this migration.

## Generating SQL

If you would like to change your production database’s schema with SQL, Alembic supports that as well. This is common in environments with tighter change management controls, or for those who have massively distributed environments and need to run many different database servers. The process is the same as performing an “online” Alembic upgrade like we did in Chapter 12. We can specify both the starting and ending versions of the SQL generated. If you don’t supply a starting migration, Alembic will build the SQL scripts for upgrading from an empty database. Example 13-4 shows how to generate the SQL for our rename migration.

Example 13-4. Generating rename migration SQL
```python
# alembic upgrade 34044511331:2e6a6cc63e9 --sql 1
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Generating static SQL
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 34044511331 -> 2e6a6cc63e9,
      Renaming cookies to new_cookies
-- Running upgrade 34044511331 -> 2e6a6cc63e9

ALTER TABLE cookies RENAME TO new_cookies;

UPDATE alembic_version SET version_num='2e6a6cc63e9'
WHERE alembic_version.version_num = '34044511331';
```

* Upgrading from 34044511331 to 2e6a6cc63e9.

The output from Example 13-4 shows the two SQL statements required to rename the cookies table, and update the Alembic migration level to the new level designated by revision ID 2e6a6cc63e9. We can write this to a file by simple redirecting the standard output to the filename of our choice, as shown here:

```python
# alembic upgrade 34044511331:2e6a6cc63e9 --sql > migration.sql
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Generating static SQL
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 34044511331 -> 2e6a6cc63e9,
      Renaming cookies to new_cookies
```

After the command executes, we can cat the migration.sql file to see our SQL statements like so:

```python
# cat migration.sql
-- Running upgrade 34044511331 -> 2e6a6cc63e9

ALTER TABLE cookies RENAME TO new_cookies;

UPDATE alembic_version SET version_num='2e6a6cc63e9'
WHERE alembic_version.version_num = '34044511331';
```

With our SQL statements prepared, we can now take this file and run it via whatever tool we like on our database servers.

WARNING
If you develop on one database backend (such as SQLite) and deploy to a different database (such as PostgreSQL), make sure to change the sqlalchemy.url configuration setting that we set in Chapter 11 to use the connection string for a PostgreSQL database. If not, you could get SQL output that is not correct for your production database! To avoid issues such as this, I always develop on the database I use in production.

This wraps ups the basics of using Alembic. In this chapter, we learned how to see the current migration level, downgrade a migration, and create SQL scripts that we can apply without using Alembic directly on our production databases. If you want to learn more about Alembic, such as how to handle code branches, you can get additional details in the documentation.

The next chapter covers a collection of common things people need to do with SQLAlchemy, including how to use it with the Flask web framework.

