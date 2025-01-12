# Inquiry

Inquiry is a simple library for Android that makes construction and use of SQLite databases super easy.

Read and write class objects from tables in a database. Let Inquiry handle the heavy lifting.

---

# Gradle Dependency

First, add JitPack.io to the repositories list in your app module's build.gradle file:

```gradle
repositories {
    maven { url "https://jitpack.io" }
}
```

Then, add Inquiry to your dependencies list:

```gradle
dependencies {
    compile 'com.afollestad:inquiry:1.2.0'
}
```

[ ![JitPack Badge](https://img.shields.io/github/release/afollestad/inquiry.svg?label=inquiry) ](https://jitpack.io/#afollestad/inquiry)

---

# Table of Contents

1. [Quick Setup](https://github.com/afollestad/inquiry#quick-setup)
2. [Example Row](https://github.com/afollestad/inquiry#example-row)
3. [Querying Rows](https://github.com/afollestad/inquiry#querying-rows)
    1. [Basics](https://github.com/afollestad/inquiry#basics)
    2. [Where](https://github.com/afollestad/inquiry#wheren)
    3. [Sorting and Limiting](https://github.com/afollestad/inquiry#sorting-and-limiting)
4. [Inserting Rows](https://github.com/afollestad/inquiry#inserting-rows)
5. [Updating Rows](https://github.com/afollestad/inquiry#updating-rows)
    1. [Basics](https://github.com/afollestad/inquiry#basics-1)
    2. [Updating Specific Columns](https://github.com/afollestad/inquiry#updating-specific-columns)
6. [Deleting Rows](https://github.com/afollestad/inquiry#deleting-rows)
7. [Dropping Tables](https://github.com/afollestad/inquiry#dropping-tables)
8. [Extra: Accessing Content Providers](https://github.com/afollestad/inquiry#extra-accessing-content-providers)
    1. [Initialization](https://github.com/afollestad/inquiry#initialization)
    2. [Basics](https://github.com/afollestad/inquiry#basics-2)

---

# Quick Setup

When your app starts, you need to initialize Inquiry. You can do so from an `Application` class,
which must be registered in your manifest. You could also put this inside of `onCreate()` in your
main `Activity`.

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        Inquiry.init(this, "myDatabase");
    }

    @Override
    public void onTerminate() {
        super.onTerminate();
        Inquiry.deinit();
    }
}
```

`init()` takes a `Context` in the first parameter, and the name of the database that'll you be using
in the second parameter. Think of a database like a file that contains a set of tables (a table is basically
a spreadsheet; it contains rows and columns).

When your app is done with Inquiry, you *should* call `deinit()` to help clean up references.

---

# Example Row

In Inquiry, a row is just an object which contains a set of values that can be read from and written to
a table in your database.

```java
public class Person {

    public Person() {
        // Default constructor is needed so Inquiry can auto construct instances
    }

    public Person(String name, int age, float rank, boolean admin) {
        this.name = name;
        this.age = age;
        this.rank = rank;
        this.admin = admin;
    }

    @Column(primaryKey = true, notNull = true, autoIncrement = true)
    public long _id;
    @Column
    public String name;
    @Column
    public int age;
    @Column
    public float rank;
    @Column
    public boolean admin;
}
```

Notice that all the fields are annotated with the `@Column` annotation. If you have fields without that
annotation, they will be ignored by Inquiry.

Notice that the `_id` field contains optional parameters in its annotation:

* `primaryKey` indicates its column is the main column used to identify the row. No other row in the
table can have the same value for that specific column. This is commonly used with IDs.
* `notNull` indicates that you can never insert null as a value for that column.
* `autoIncrement` indicates that you don't manually set the value of this column. Every time
you insert a row into the table, this column will be incremented by one automatically. This can
only be used with INTEGER columns (short, int, or long fields), however.

---

# Querying Rows

#### Basics

Querying retrieves rows, whether its every row in a table or rows that match a specific criteria.
Here's how you would retrieve all rows from a table called *"people"*:

```java
Person[] result = Inquiry.get()
    .selectFrom("people", Person.class)
    .all();
```

If you only needed one row, using `one()` instead of `all()` is more efficient:

```java
Person result = Inquiry.get()
    .selectFrom("people", Person.class)
    .one();
```

---

You can also perform the query on a separate thread using a callback:

```java
Inquiry.get()
    .selectFrom("people", Person.class)
    .all(new GetCallback<Person>() {
        @Override
        public void result(Person[] result) {
            // Do something with result
        }
    });
```

Inquiry will automatically fill in your `@Column` fields with matching columns in each row of the table.

#### Where

If you wanted to find rows with specific values in their columns, you can use `where` selection:

```java
Person[] result = Inquiry.get()
    .selectFrom("people", Person.class)
    .where("name = ? AND age = ?", "Aidan", 20)
    .all();
```

The first parameter is a string, specifying two conditions that must be true (`AND` is used instead of `OR`).
The question marks are placeholders, which are replaced by the values you specify in the second comma-separated
vararg (or array) parameter.

---

If you wanted, you could skip using the question marks and only use one parameter:

```java
.where("name = 'Aidan' AND age = 20");
```

*However*, using the question marks and filler parameters can be easier to read if you're filling them in
with variables. Plus, this will automatically escape any strings that contain reserved SQL characters.

#### Sorting and Limiting

This code would limit the maximum number of rows returned to 100. It would sort the results by values
in the "name" column, in descending (Z-A, or greater to smaller) order:

```java
Person[] result = Inquiry.get()
    .selectFrom("people", Person.class)
    .limit(100)
    .sort("name DESC")
    .all();
```

If you understand SQL, you'll know you can specify multiple sort parameters separated by commas.

```java
.sort("name DESC, age ASC");
```

The above sort value would sort every column by name descending (large to small, Z-A) first, *and then* by age ascending (small to large).

# Inserting Rows

Insertion is pretty straight forward. This inserts three `People` into the table *"people"*:

```java
Person one = new Person("Waverly", 18, 8.9f, false);
Person two = new Person("Natalie", 42, 10f, false);
Person three = new Person("Aidan", 20, 5.7f, true);

long insertedCount = Inquiry.get()
        .insertInto("people", Person.class)
        .values(one, two, three)
        .run();
```

Inquiry will automatically pull your `@Column` fields out and insert them into the table `people`.

Like `all()`, `run()` has a callback variation that will run the operation in a separate thread:

```java
Inquiry.get()
    .insertInto("people", Person.class)
    .values(one, two, three)
    .run(new RunCallback() {
        @Override
        public void result(long insertedCount) {
            // Do something
        }
    });
```

# Updating Rows

#### Basics

Updating is similar to insertion, however it results in changed rows rather than new rows:

```java
Person two = new Person("Natalie", 42, 10f, false);

long updatedCount = Inquiry.get()
    .update("people", Person.class)
    .values(two)
    .where("name = ?", "Aidan")
    .run();
```

The above will update all rows whose name is equal to *"Aidan"*, setting all columns to the values in the `Person`
object called `two`. If you didn't specify `where()` args, every row in the table would be updated.

#### Updating Specific Columns

Sometimes, you don't want to change every column in a row when you update them. You can choose specifically
what columns you want to be changed using `onlyUpdate`:

```java
Person two = new Person("Natalie", 42, 10f, false);

long updatedCount = Inquiry.get()
    .update("people", Person.class)
    .values(two)
    .where("name = ?", "Aidan")
    .onlyUpdate("age", "rank")
    .run();
```

The above code will update any rows with their name equal to *"Aidan"*, however it will only modify
the `age` and `rank` columns of the updated rows. The other columns will be left alone.

# Deleting Rows

Deletion is simple:

```java
int deletedCount = Inquiry.get()
    .deleteFrom("people")
    .where("age = ?", 20)
    .run();
```

The above code results in any rows with their age column set to *20* removed. If you didn't
specify `where()` args, every row in the table would be deleted.

# Dropping Tables

Dropping a table means deleting it. It's pretty straight forward:

```java
Inquiry.get()
    .dropTable("people");
```

Just pass table name, and it's gone.

---

# Extra: Accessing Content Providers

Inquiry allows you to access content providers, which are basically external databases used in other apps.
A common usage of content providers is Android's MediaStore. Most local media players use content providers
to get a list of audio and video files scanned by the system; the system logs all of their meta data
so the title, duration, album art, etc. can be quickly accessed.

#### Initialization

Inquiry initialization is still the same, but passing a database name is not required for content providers.

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        Inquiry.init(this);
    }

    @Override
    public void onTerminate() {
        super.onTerminate();
        Inquiry.deinit();
    }
}
```

#### Basics

This small example will read artists (for songs) on your phone. Here's the row class:

```java
public class Artist {

    public Artist() {
    }

    @Column
    public long _id;
    @Column
    public String artist;
    @Column
    public String artist_key;
    @Column
    public int number_of_albums;
    @Column
    public int number_of_tracks;
}
```

You can perform all the same operations, but you pass a `content://` URI instead of a table name:

```java
Uri artistsUri = Uri.parse("content://media/internal/audio/artists");
Artist[] artists = Inquiry.get()
    .selectFrom(artistsUri, Artist.class)
    .all();
```

Insert, update, and delete work the same way. Just pass that URI.
