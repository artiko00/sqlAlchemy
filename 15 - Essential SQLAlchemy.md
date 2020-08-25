# Chapter 5. Reflection

Reflection is a technique that allows us to populate a SQLAlchemy object from an existing database. You can reflect tables, views, indexes, and foreign keys. This chapter will explore how to use reflection on an example database.

For testing, I recommend using Chinook database. You can learn more about it at http://chinookdatabase.codeplex.com/. We’ll be using the SQLite version, which is available in the CH06/ folder of this book’s sample code. That folder also contains an image of the database schema so you can visualize the schema that we’ll be working with throughout this chapter. We’ll begin by reflecting a single table.

## Reflecting Individual Tables

For our first reflection, we are going to generate the Artist table. We’ll need a meta-data object to hold the reflected table schema information, and an engine attached to the Chinook database. Example 5-1 demonstrates how to set up both of these things; the process should be very familiar to you now.

```python
from sqlalchemy import MetaData, create_engine
metadata = MetaData()
engine = create_engine('sqlite:///Chinook_Sqlite.sqlite') 1
```

1- This connection string assumes you are in the same directory as the example database.

With the metadata and engine set up, we have everything we need to reflect a table. We are going to create a table object using code similar to the table creation code in Chapter 1; however, instead of defining the columns by hand, we are going to use the autoload and autoload_with keyword arguments. This will reflect the schema information into the metadata object and store a reference to the table in the artist variable. Example 5-2 demonstrates how to perform the reflection.

```python
from sqlalchemy import Table
artist = Table('Artist', metadata, autoload=True, autoload_with=engine)
```

That last line really is all we need to reflect the Artist table! We can use this table just as we did in Chapter 2 with our manually defined tables. Example 5-3 shows how to perform a simple query with the table to see what kind of data is in it.

```python
artist.columns.keys() 1
from sqlalchemy import select
s = select([artist]).limit(10) 2
engine.execute(s).fetchall()
```

1- Listing the columns.
2- Selecting the first 10 records.

This results in:

```python
['ArtistId', 'Name']

[(1, u'AC/DC'),
 (2, u'Accept'),
 (3, u'Aerosmith'),
 (4, u'Alanis Morissette'),
 (5, u'Alice In Chains'),
 (6, u'Ant\xf4nio Carlos Jobim'),
 (7, u'Apocalyptica'),
 (8, u'Audioslave'),
 (9, u'BackBeat'),
 (10, u'Billy Cobham')]
```

Looking at the database schema, we can see that there is an Album table that is related to the Artist table. Let’s reflect it the same way as we did for the Artist table:

```python
album = Table('Album', metadata, autoload=True, autoload_with=engine)
```

Now let’s check the Album table’s metadata to see what got reflected. Example 5-4 shows how to do that.

```python
metadata.tables['album']

Table('album',
      MetaData(bind=None),
      Column('AlbumId', INTEGER(), table=<album>, primary_key=True, nullable=False),
      Column('Title', NVARCHAR(length=160), table=<album>, nullable=False),
      Column('ArtistId', INTEGER(), table=<album>, nullable=False),
             schema=None)
)
```

Interestingly, the foreign key to the Artist table does not appear to have been reflected. Let’s check the foreign_keys attribute of the Album table to make sure that it is not present:

```python
album.foreign_keys

set()
```

So it really didn’t get reflected. This occurred because the two tables weren’t reflected at the same time, and the target of the foreign key was not present during the reflection. In an effort to not leave you in a semi-broken state, SQLAlchemy discarded the one-sided relationship. We can use what we learned in Chapter 1 to add the missing ForeignKey, and restore the relationship:

```python
from sqlalchemy import ForeignKeyConstraint
album.append_constraint(
    ForeignKeyConstraint(['ArtistId'], ['artist.ArtistId'])
)
```

Now if we rerun Example 5-4, we can see the ArtistId column is a ForeignKey:

```python
Table('album',
      MetaData(bind=None),
      Column('AlbumId', INTEGER(), table=<album>, primary_key=True,
              nullable=False),
      Column('Title', NVARCHAR(length=160), table=<album>, nullable=False),
      Column('ArtistId', INTEGER(), ForeignKey('artist.ArtistId'), table=<album>,
              nullable=False),
      schema=None)
```

Now let’s see if we can use the relationship to join the tables properly. We can run the following code to test the relationship:

```python
str(artist.join(album))

'artist JOIN album ON artist."ArtistId" = album."ArtistId"'
```

Excellent! Now we can perform queries that use this relationship. It works just like the queries discussed in “Joins”.

It would be quite a bit of work to repeat the reflection process for each individual table in our database. Fortunately, SQLAlchemy lets you reflect an entire database at once.

## Reflecting a Whole Database

In order to reflect a whole database, we can use the reflect method on the metadata object. The reflect method will scan everything available on the engine supplied, and reflect everything it can. Let’s use our existing metadata and engine objects to reflect the entire Chinook database:

```python
metadata.reflect(bind=engine)
```

This statement doesn’t return anything if it succeeds; however, we can run the following code to retrieve a list of table names to see what was reflected into our metadata:

```python
metadata.tables.keys()

dict_keys(['InvoiceLine', 'Employee', 'Invoice', 'album', 'Genre',
           'PlaylistTrack', 'Album', 'Customer', 'MediaType', 'Artist',
           'Track', 'artist', 'Playlist'])
```

The tables we manually reflected are listed twice but with different case letters. This is due to that fact that SQLAlchemy reflects the tables as they are named, and in the Chinook database they are uppercase. Due to SQLite’s handling of case sensitivity, both the lower- and uppercase names point to the same tables in the database.


    CAUTION
    Be careful, as case sensitivity can trip you up on other databases, such as Oracle.

Now that we have our database reflected, we are ready to discuss using reflected tables in queries.

## Query Building with Reflected Objects

As you saw in Example 5-3, querying tables that we reflected and stored in a variable works just like it did in Chapter 2. However, for the rest of the tables that were reflected when we reflected the entire database, we’ll need a way to refer to them in our query. We can do that by assigning them to a variable from the tables attribute of the metadata, as shown in Example 5-5.

```python
playlist = metadata.tables['Playlist'] 1

from sqlalchemy import select
s = select([playlist]).limit(10) 2
engine.execute(s).fetchall()
```

1- Establish a variable to be a reference to the table.
2- Use that variable in the query.

Running the code in Example 5-5 gives us this result:

```python
engine.execute(s).fetchall()
[(1, 'Music'),
 (2, 'Movies'),
 (3, 'TV Shows'),
 (4, 'Audiobooks'),
 (5, '90’s Music'),
 (6, 'Audiobooks'),
 (7, 'Movies'),
 (8, 'Music'),
 (9, 'Music Videos'),
 (10, 'TV Shows')]
```

By assigning the reflected tables we want to use into a variable, they can be used in the same way as previous chapters.

Reflection is a very useful tool; however, as of version 1.0 of SQLAlchemy, we cannot reflect CheckConstraints, comments, or triggers. You also can’t reflect client-side defaults or an association between a sequence and a column. However, it is possible to add them manually using the methods described in Chapter 1.

You now understand how to reflect individual tables and repair issues like missing relationships in those tables. Additionally, you learned how to reflect an entire database, and use individual tables in queries from the reflected tables.

This chapter wraps up the essential parts of SQLAlchemy Core and the SQL Expression Language. Hopefully, you’ve gotten a sense for how powerful these parts of SQLAlchemy are. They are often overlooked due to the SQLAlchemy ORM, but using them can add even more capabilities to your applications. Let’s move on to learning about the SQLAlchemy ORM.