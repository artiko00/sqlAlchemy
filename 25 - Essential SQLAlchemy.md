# Chapter 10. Reflection with SQLAlchemy ORM and Automap

As you learned in Chapter 5, reflection lets you populate a SQLAlchemy object from an existing database; reflection works on tables, views, indexes, and foreign keys. But what if you want to reflect a database schema into ORM-style classes? Fortunately, the handy SQLAlchemy extension automap lets you do just that.

Reflection via automap is a very useful tool; however, as of version 1.0 of SQLAlchemy we cannot reflect CheckConstraints, comments, or triggers. You also can’t reflect client-side defaults or an association between a sequence and a column. However, it is possible to add them manually using the methods we learned in Chapter 6.

Just like in Chapter 5, we are going to use the Chinook database for testing. We’ll be using the SQLite version, which is available in the CH11/ folder of this book’s sample code. That folder also contains an image of the database schema so you can visualize the schema we’ll be working with throughout this chapter.

## Reflecting a Database with Automap

In order to reflect a database, instead of using the declarative_base we’ve been using with the ORM so far, we’re going to use the automap_base. Let’s start by creating a Base object to work with, as shown in Example 10-1

Example 10-1. Creating a Base object with automap_base
```python
from sqlalchemy.ext.automap import automap_base 1

Base = automap_base() 2
```
* Imports the automap_base from the automap extension.
* Initializes a Base object.

Next, we need an engine connected to the database that we want to reflect. Example 10-2 demonstrates how to connect to the Chinook database.

Example 10-2. Initializaing an engine for the Chinook database
```python
from sqlalchemy import create_engine

engine = create_engine('sqlite:///Chinook_Sqlite.sqlite') 1
```

This connection string assumes you are in the same directory as the example database.

With the Base and engine setup, we have everything we need to reflect the database. Using the prepare method on the Base object we created in Example 10-1 will scan everything available on the engine we just created, and reflect everything it can. Here’s how you reflect the database using the automap Base object:

```python
Base.prepare(engine, reflect=True)
```

That one line of code is all you need to reflect the entire database! This reflection has created ORM objects for each table that is accessible under the class property of the automap Base. To print a list of those objects, simply run this line of code:

```python
Base.classes.keys()
```

Here’s the output you get when you do that:

```python
['Album',
 'Customer',
 'Playlist',
 'Artist',
 'Track',
 'Employee',
 'MediaType',
 'InvoiceLine',
 'Invoice',
 'Genre']
```

Now let’s create some objects to reference the Artist and Album tables:

```python
Artist = Base.classes.Artist
Album = Base.classes.Album
```

The first line of code creates a reference to the Artist ORM object we reflected, and the second line creates a reference to the Album ORM object we reflected. We can use the Artist object just as we did in Chapter 7 with our manually defined ORM objects. Example 10-3 demonstrates how to perform a simple query with the object to get the first 10 records in the table.

Example 10-3. Using the Artist table
```python
from sqlalchemy.orm import Session

session = Session(engine)
for artist in session.query(Artist).limit(10):
    print(artist.ArtistId, artist.Name)
```

Example 10-3 will output the following:

```python
(1, u'AC/DC')
(2, u'Accept')
(3, u'Aerosmith')
(4, u'Alanis Morissette')
(5, u'Alice In Chains')
(6, u'Ant\xf4nio Carlos Jobim')
(7, u'Apocalyptica')
(8, u'Audioslave')
(9, u'BackBeat')
(10, u'Billy Cobham')
```
Now that we know how to reflect a database and map it to objects, let’s look at how relationships are reflected via automap.

## Reflected Relationships

Automap can automatically reflect and establish many-to-one, one-to-many, and many-to-many relationships. Let’s look at how the relationship between our Album and Artist tables was established. When automap creates a relationship, it creates a <related_object>_collection property on the object, as shown on the Artist object in Example 10-4.

Example 10-4. Using the relationship between Artist and Album to print related data
```python
artist = session.query(Artist).first()
for album in artist.album_collection:
    print('{} - {}'.format(artist.Name, album.Title))
```
This will output:

```python
AC/DC - For Those About To Rock We Salute You
AC/DC - Let There Be Rock
```

You can also configure automap and override certain aspects of its behavior to tailor the classes it creates to precise specifications; however, that is well beyond the scope of this book. You can learn much more about this in the SQLAlchemy documenation.

This chapter wraps up the essential parts of SQLAlchemy ORM. Hopefully, you’ve gotten a sense for how powerful this part of SQLAlchemy is. This book has given you a solid introduction to ORM, but there’s plenty more to learn; for more information, check out the documentation.

In the next section, we’re going to learn how to use Alembic to manage database migrations so that we can change our database schema without having to destroy and re-create the database.

