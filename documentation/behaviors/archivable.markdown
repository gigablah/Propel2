---
layout: documentation
title: Archivable Behavior
---

# Archivable Behavior #

The `archivable` behavior gives model objects the ability to be copied to an archive table. By default, the behavior archives objects on deletion, which makes it the Propel implementation of the "soft delete" pattern.

## Basic Usage ##

In the `schema.xml`, use the `<behavior>` tag to add the `archivable` behavior to a table:

```xml
<table name="book">
  <column name="id" required="true" primaryKey="true" autoIncrement="true" type="INTEGER" />
  <column name="title" type="VARCHAR" required="true" primaryString="true" />
  <behavior name="archivable" />
</table>
```

Rebuild your model, insert the table creation sql again, and you're ready to go. The model now has one new table, `book_archive`, with the same columns as the original `book` table. This table stores the archived `books` together with their archive date. To archive an object, call the `archive()` method:

```php
<?php
$book = new Book();
$book->setTitle('War And Peace');
$book->save();
// copy the current Book to a BookArchive object and save it
$archivedBook = $book->archive();
```

The archive table contains only the freshest copy of each archived objects. Archiving an object twice doesn't create a new record in the archive table, but updates the existing archive.

The `book_archive` table has generated ActiveRecord and ActiveQuery classes, so you can browse the archive at will. The archived objects have the same primary key as the original objects. In addition, they contain an `ArchivedAt` property storing the date where the object was archived.

```php
<?php
// find the archived book
$archivedBook = BookArchiveQuery::create()->findPk($book->getId());
echo $archivedBook->getTitle(); // 'War And Peace'
echo $archivedBook->getArchivedAt(); // 2011-08-23 18:14:23
```

The ActiveRecord class of an `archivable` model has more methods to deal with the archive:

```php
// restore an object to the state it had when last archived
$book->restoreFromArchive();
// find the archived version of an existing book
$archivedBook = $book->getArchive();
// populate a book based on an archive
$book = new book();
$book->populateFromArchive($archivedBook);
```

By default, an `archivable` model is archived just before deletion:

```php
<?php
$book = new Book();
$book->setTitle('Sense and Sensibility');
$book->save();
// delete and archive the book
$book->delete();
echo BookQuery::create()->count(); // 0
// find the archived book
$archivedBook = BookArchiveQuery::create()
  ->findOneByTitle('Sense and Sensibility');
```

>**Tip**<br />The behavior does not take care of archiving the related objects. This may be surprising on deletions if the deleted object has 'ON DELETE CASCADE' foreign keys. If you want to archive relations, override the generated `archive()` method in the ActiveRecord class with your custom logic.

To recover deleted objects, use `populateFromArchive()` on a new object and save it:

```php
<?php
// create a new object based on the archive
$book = new Book();
$book->populateFromArchive($archivedBook);
$book->save();
echo $book->getTitle(); // 'Sense and Sensibility'
```

If you want to delete an `archivable` object without archiving it, use the `deleteWithoutArchive()` method generated by the behavior:

```php
<?php
// delete the book but don't archive it
$book->deleteWithoutArchive();
```

## Archiving A Set Of Objects ##

The `archivable` behavior also generates an `archive()` method on the generated ActiveQuery class. That means you can easily archive a set of objects, in the same way you archive a single object:

```php
<?php
// archive all books having a title starting with "war"
$nbArchivedObjects = BookQuery::create()
  ->filterByTitle('War%')
  ->archive();
```

`archive()` returns the number of archived objects, and not the current ActiveQuery object, so it's a termination method.

>**Tip**<br />Since the `archive()` method doesn't duplicate archived objects, it must iterate over the results of the query to check whether each object has already been archived. In practice, `archive()` issues 2n+1 database queries, where `n` is the number of results of the query as returned by a `count()`.

As explained earlier, an `archivable` model is archived just before deletion by default. This is also true when using the `delete()` and `deleteAll()` methods of the ActiveQuery class:

```php
<?php
// delete and archive all books having a title starting with "war"
$nbDeletedObjects = BookQuery::create()
  ->filterByTitle('War%')
  ->delete();

// use deleteWithoutArchive() if you just want to delete
$nbDeletedObjects = BookQuery::create()
  ->filterByTitle('War%')
  ->deleteWithoutArchive();

// you can also turn off the query alteration on the current query
// by calling setArchiveOnDelete(false) before deleting
$nbDeletedObjects = BookQuery::create()
  ->filterByTitle('War%')
  ->setArchiveOnDelete(false)
  ->delete();
```

## Archiving on Insert, Update, or Delete ##

As explained earlier, the `archivable` behavior archives objects on deletion by default, but insertions and updates don't trigger the `archive()` method. You can disable the auto archiving on deletion, as well as enable it for insertion and update, in the behavior `<parameter>` tags. Here is the default configuration:

```xml
<table name="book">
  <column name="id" required="true" primaryKey="true" autoIncrement="true" type="INTEGER" />
  <column name="title" type="VARCHAR" required="true" primaryString="true" />
  <behavior name="archivable">
    <parameter name="archive_on_insert" value="false" />
    <parameter name="archive_on_update" value="false" />
    <parameter name="archive_on_delete" value="true" />
  </behavior>
</table>
```

If you turn on `archive_on_insert`, a call to `save()` on a new ActiveRecord object archives it - unless you call `saveWithoutArchive()`.

If you turn on `archive_on_update`, a call to `save()` on an existing ActiveRecord object archives it, and a call to `update()` on an ActiveQuery object archives the results as well. You can still use `saveWithoutArchive()` on the ActiveRecord class and `updateWithoutArchive()` on the ActiveQuery class to skip archiving on updates.

Of course, even if `archive_on_insert` or any of the similar parameters isn't turned on, you can always archive manually an object after persisting it by simply calling `archive()`:

```php
<?php
// create a new object, save it, and archive it
$book = new Book();
$book->save();
$book->archive();
```

## Archiving To Another Database ##

The behavior can use another database connection for the archive table, to make it safer. To allow cross-database archives, you must declare the archive schema manually in another XML schema, and reference the archive class on in the behavior parameter:

```xml
<database name="main">
  <table name="book">
    <column name="id" required="true" primaryKey="true" autoIncrement="true" type="INTEGER" />
    <column name="title" type="VARCHAR" required="true" primaryString="true" />
    <behavior name="archivable">
      <parameter name="archive_class" value="MyBookArchive" />
    </behavior>
  </table>
</database>
<database name="backup">
  <table name="my_book_archive" phpName="MyBookArchive">
    <column name="id" required="true" primaryKey="true" type="INTEGER" />
    <column name="title" type="VARCHAR" required="true" primaryString="true" />
    <column name="archived_at" type="TIMESTAMP" />
  </table>
</database>
```

The archive table must have the same columns as the archivable table, but without autoIncrements, and without foreign keys.

With this setup, the behavior uses `MyBookArchive` and `MyBookArchiveQuery` for all operations on archives, and therefore uses the `backup` connection.

## Migrating From `soft_delete` ##

If you use `archivable` as a replacement for the `soft_delete` behavior, here is how you should update your code:

```php
<?php
// do a soft delete
$book->delete(); // with soft_delete
$book->delete(); // with archivable

// do a hard delete
// with soft_delete
$book->forceDelete();
// with archivable
$book->deleteWithoutArchive();

// find deleted objects
// with soft_delete
$books = BookQuery::create()
  ->includeDeleted()
  ->where('Book.DeletedAt IS NOT NULL')
  ->find();
// with archivable
$bookArchives = BookArchiveQuery::create()
  ->find();

// recover a deleted object
// with soft_delete
$book->unDelete();
// with archivable
$book = new Book();
$book->populateFromArchive($bookArchive);
$book->save();
```

## Additional Parameters ##

You can change the name of the archive table added by the behavior by setting the `archive_table` parameter. If the table doesn't exist, the behavior creates it for you.

```xml
<behavior name="archivable">
  <parameter name="archive_table" value="special_book_archive" />
</behavior>
```

>**Tip**<br />The `archive_table` and `archive_class` parameters are mutually exclusive. You can only use either one of the two.

You can also change the name of the column storing the archive date:

```xml
<behavior name="archivable">
  <parameter name="archived_at_column" value="archive_date" />
</behavior>
```

Alternatively, you can disable the addition of an archive date column altogether:

```xml
<behavior name="archivable">
  <parameter name="log_archived_at" value="false" />
</behavior>
```
