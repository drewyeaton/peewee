Some typical usage scenarios
============================

Below are outlined some of the ways to perform typical database-related tasks
with peewee.

Examples will use the following models:

.. code-block:: python
    
    import peewee

    class Blog(peewee.Model):
        creator = peewee.CharField()
        name = peewee.CharField()


    class Entry(peewee.Model):
        blog = peewee.ForeignKeyField(Blog)
        title = peewee.CharField()
        body = peewee.TextField()
        pub_date = peewee.DateTimeField()
        published = peewee.BooleanField(default=True)


Creating a database connection and tables
-----------------------------------------

It is necessary to explicitly connect to the database before using it.  By
default, peewee provides a default database in the module-scope:

.. code-block:: python

    >>> peewee.database.connect()
    >>> Blog.create_table()
    >>> Entry.create_table()


It is possible to use multiple databases (provided that you don't try and mix
models from each):

.. code-block:: python

    >>> custom_db = peewee.SqliteDatabase('custom.db')
    
    >>> class CustomModel(peewee.Model):
    ...     whatev = peewee.CharField()
    ...     
    ...     class Meta:
    ...         database = custom_db
    ... 
    
    >>> custom_db.connect()
    >>> CustomModel.create_table()


Standard practice is to define a base model class that points at your custom
database, and then all your models will extend it:

.. code-block:: python

    custom_db = peewee.SqliteDatabase('custom.db')
    
    class CustomModel(peewee.Model):
        class Meta:
            database = custom_db
    
    class Blog(CustomModel):
        author = peewee.CharField()
        name = peewee.TextField()
    
    class Entry(CustomModel):
        # etc, etc


Creating a new record
---------------------

You can use the `create` method on the model:

.. code-block:: python

    >>> Blog.create(creator='Charlie', name='My Blog')
    <__main__.Blog object at 0x2529350>

Alternatively, you can build up a model instance programmatically and then
save it:

.. code-block:: python

    >>> blog = Blog()
    >>> blog.creator = 'Chuck'
    >>> blog.name = 'Another blog'
    >>> blog.save()
    >>> blog.id
    2

Once a model instance has a primary key, any attempt to re-save it will result
in an update rather than another insert:

.. code-block:: python

    >>> blog.save()
    >>> blog.id
    2
    >>> blog.save()
    >>> blog.id
    2


Getting a single record
-----------------------

.. code-block:: python

    >>> Blog.get(id=1)
    <__main__.Blog object at 0x25294d0>

    >>> Blog.get(id=1).name
    u'My Blog'

    >>> Blog.get(creator='Chuck')
    <__main__.Blog object at 0x2529410>

    >>> Blog.get(creator='Chuck').name
    u'Another blog'


Selecting some records
----------------------

To simply get all instances in a table, call the `select` method:

.. code-block:: python

    >>> for blog in Blog.select():
    ...     print blog.name
    ... 
    My Blog
    Another blog

To get all the related instances for an object, you can query the related name:

.. code-block:: python

    >>> for entry in blog.entry_set:
    ...     print entry.title
    ... 
    entry 1
    entry 2
    entry 3
    entry 4


Filtering records
-----------------

.. code-block:: python

    >>> for entry in Entry.select().where(blog=blog, published=True):
    ...     print '%s: %s (%s)' % (entry.blog.name, entry.title, entry.published)
    ... 
    My Blog: Some Entry (True)
    My Blog: Another Entry (True)

    >>> for entry in Entry.select().where(pub_date__lt=datetime.datetime(2011, 1, 1)):
    ...     print entry.title, entry.pub_date
    ... 
    Old entry 2010-01-01 00:00:00

You can also filter across joins:

.. code-block:: python

    >>> for entry in Entry.select().join(Blog).where(name='My Blog'):
    ...     print entry.title
    Old entry
    Some Entry
    Another Entry


Sorting records
---------------

.. code-block:: python

    >>> for e in Entry.select().order_by('pub_date'):
    ...     print e.pub_date
    ... 
    2010-01-01 00:00:00
    2011-06-07 14:08:48
    2011-06-07 14:12:57

    >>> for e in Entry.select().order_by(peewee.desc('pub_date')):
    ...     print e.pub_date
    ... 
    2011-06-07 14:12:57
    2011-06-07 14:08:48
    2010-01-01 00:00:00

You can also order across joins although its a little trickier.  Assuming you want
to order entries by the name of the blog, then by pubdate:

.. code-block:: python

    >>> qry = Entry.select().join(Blog).order_by('name').switch(Entry).order_by('pub_date')
    >>> qry.sql()
    ('SELECT t1.* FROM entry AS t1 INNER JOIN blog AS t2 ON t1.blog_id = t2.id ORDER BY t2.name ASC, t1.pub_date ASC', [])

The strangeness there is that you need to join on Blog first so that it can be ordered on,
then after specifying the ordering for Blog, switch back to Entry and order on it.


Paginating records
------------------

The paginate method makes it easy to grab a "page" or records -- it takes two
parameters, `page_number`, and `items_per_page`:

.. code-block:: python

    >>> for entry in Entry.select().order_by('id').paginate(2, 10):
    ...     print entry.title
    ... 
    entry 10
    entry 11
    entry 12
    entry 13
    entry 14
    entry 15
    entry 16
    entry 17
    entry 18
    entry 19


Counting records
----------------

You can count the number of rows in any select query:

.. code-block:: python

    >>> Entry.select().count()
    100
    >>> Entry.select().where(id__gt=50).count()
    50