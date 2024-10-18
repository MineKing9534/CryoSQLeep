# KORMite
KORMite is a library that aims to abstract database operations to simple kotlin functions. 
Therefore, the library provides a `Table` interface with methods for database operations. 
See [Usage](#usage) for a detailed explanation. 

## Installation
KORMite is hosted on a custom repository at [https://maven.mineking.dev](https://maven.mineking.dev/#/releases/de/mineking/KORMite). Replace VERSION with the latest version (without the `v` prefix).
Alternatively, you can download the artifacts from jitpack (not recommended).

### Gradle

```kotlin
repositories {
    maven("https://maven.mineking.dev/releases")
}

dependencies {
    implementation("de.mineking:KORMite-core:VERSION")
}
```

### Maven

```xml
<repositories>
    <repository>
        <id>mineking</id>
        <url>https://maven.mineking.dev/releases</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>de.mineking</groupId>
        <artifactId>KORMite-core</artifactId>
        <version>VERSION</version>
    </dependency>
</dependencies>
```

## Vendors
The core library only contains the infrastructure for type mapping etc. To actually connect to a database and work with it, you also need the dependency for the database vendor you use, that then contains the actual implementation for SQL generation etc.

For postgreSQL (recommended), you can use `de.mineking:KORMite-postgres:VERSION`.

## Usage
The basic usage looks like this:
```kotlin
data class UserDao(
    @Autoincrement @Key @Column val id: Int = 0, //Autoincrementing key column with name id. When having the value '0', the value will not be passed in an insert statement and will therefore be generated by the database 
    @Column val name: String, 
    @Column val age: Int 
)

fun main() { 
    val connection = PostgresConnection("localhost:5432/test", user = "test", password = "test") 
    val table = connection.getTable(name = "users", create = true) { UserDao(name = "", age = 0) } //Create table with name and function to provide a base instance
    //Every time a value is read, an instance is created using this function and will that put in the values from the database into the fields of that instance
    //With 'create = true', the library will automatically create a table with the specified columns. However, this will NOT update an existing table to match the specifications, if your class changed since the table was created  
    
    assertEquals(table.insert(UserDao(name = "Max", age = 25)).isSucces())
    
    val result = table.select().list() //Will read all table rows, create an UserDao instance for every row and then put into a List
    assertEquals(1, result.size)
    assertEquals(1, result[0].id) //The id was generated by the database and also updated in the instance
    assertEquals("Max", result[0].name)
    
    table.update(result[0].copy(name = "Tom")) //We can update a row in the table. The row to update is identified by the key column(s) 
    assertEquals("Tom", table.select().first().name)
    
    table.delete() //This will delete all rows in the table (NOT the table itself)
    assertEquals(0, table.selectRowCount())
}
```

Note how all properties that should appear in the database have to be annotated with `@Column`.
Using this annotation, you can also specify the name the column should have in the database.

When you don't specify a name, the name of the kotlin property is used. However, the name is passed to a `NamingStrategy` beforehand.
By default, `NamingStrategy.SNAKE_CASE` is used. However, you can specify your own when calling `getTable` or in the `Connection` constructor.

### Conditions
In many cases, you need to specify conditions on what to select or delete. There is a simple way to do that:
```kotlin
fun main() {
	//Your connection and table declaration...
	
	table.select(where = (property("name") isEqualTo value("Max")) and (property("age").isBetween(value(18), value(30))))
}
```

Conditions are bases on two parts: `Node`s and `Where`s. 

#### Nodes
A node represents either a database column (`property`) or a constant passed as parameter (`value`).
Nodes cna also be appended together using the `+` operator.

##### property
A property node will reference a database column. Note: you have to specify the kotlin property name, even if you passed a custom name in `@Column`.

There is also a special syntax for accessing more complex columns:

| Name           | Example        | Description                                                                                                     |
|----------------|----------------|-----------------------------------------------------------------------------------------------------------------|
| Array Index    | array[0]       | Access an index of an array. This is zero based (like in kotlin) as opposed to how it behaves by default in SQL |
| Reference      | user->name     | Access a referenced tables' column (See [References](#references))                                              |
| Virtual Column | location.world | Access a virtual child of this column (See [Virtual Columns](#virtual-columns))                                 |


##### value
You can specify any object here. As long as a [TypeMapper](#type-mappers) for that object is found, it will be correctly passed as a statement parameter.
All values are passed using prepared statement, so you are safe to use them without having to worry about sql injection.

#### Wheres
Nodes can easily be converted to a where condition. You can simply call `Where(node)`. 
However, in most cases you shouldn't use this but instead make use of the default functions.

For example `Where(property("test") + " isEqualTo " + value(1))` can be written as `property("test") isEqualTo value(1)`, because all default operations have (infix) functions available.

### Single-Column access
There are also situations where you want to only select or update a single column, without handling the entire object for the row.

```kotlin
fun main() {
    //Your connection and table declaration...
	
    val names = table.select(property("name"), where = property("age") isGreaterThan value(5)).list() //This will only select the name column. You can also specify conditions and all parameters that the normal select supports
    val upperCase = table.select(upperCase(property("name"))).list() //You can also specify complex nodes in this case. This allows manipulation of the data on the database like calling functions
    //upperCase(property("name2)) is translated to `upper("name")` in SQL. You can also call custom SQL functions using "yourFunction"(node)
    
    table.update("name", value("Test"), where = property("age") isGreaterThan value(5)) //Update only the name column. You can also optionally specify a condition here 
}
```

### Type Mappers
TypeMappers are used by this library to define how a kotlin property is mapped to a database column and vice versa.

#### Default Types
For the basic types, this library already comes with a set of TypeMappers (otherwise you wouldn't be able to use it for even the simplest cases).
These include numbers, string, booleans etc.

Enums and arrays are also supported by default. 
Enums are simply stored as strings. 
Arrays, Lists and Sets will simply create a column with an array type. Every type will automatically support a corresponding array type by default, even for you custom type mappers.

#### Custom Type Mappers
you can also register custom TypeMappers for you own classes, to be able to store them as a database column. You can either create an implementation of the TypeMapper interface directly, however, there are two functions to simply the process for easy cases: 

```kotlin
fun main() {
    //Your connection declaration...
	
    //This is how you create and register a custom TypeMapper, on the example of the default implementation of Strings (you don't have to do this to use strings as this mapper is already added by default)
    connection.typeMappers += typeMapper<String?>(PostgresType.TEXT, ResultSet::getString) { value, statement, position -> statement.setString(position, value) }
    
    //You might have cases where you want to map a complex type, so a simpler type, that is then used for the column
    connection.typeMappers += typeMapper<Color?, String?>(PostgresMappers.INTEGER, { it?.let { Color(it) } }, Color::rgb)
    //This creates a TypeMapper with intermediate type INTEGER that then only has to specify how to convert between Color and Int, while the specified intermediate TypeMapper handles the direct access
}
```

### References
References are a way to store complex objects in a different table and store a reference to the corresponding row in a single column.

```kotlin
data class AuthorDao(
    @Autoincrement @Key @Column val id: Int = 0, 
    @Column val name: String
)

data class BookDao(
    @Autoincrement @Key @Column val id: Int = 0, 
    @Column val name: String,
    @Reference("authors") @Column val author: Author
)

fun main() {
    val connection = PostgresConnection("localhost:5432/test", user = "test", password = "test")
    val authorTable = connection.getTable(name = "authors", create = true) { AuthorDao(name = "") }
    val bookTable = connection.getTable(name = "authors", create = true) { BookDao(name = "", author = AuthorDao(name = "")) }
    
    val author = authorTable.insert(AuthorDao(name = "Test Author")).value!!
    val book = bookTable.insert(BookDao(name = "Test Book", author = author)).value!!
    
    assertEquals(author, bookTable.select<AuthorDao>(property("author")).first())
    assertEquals("Test Author", bookTable.select<String>(property("author->name")).first())
}
```

### Virtual Columns
Virtual columns can be used to store additional information for a property in a different column in the database. 
An example are `Location`s in Minecraft, where you might want to store the location itself as an array of doubles and the world as a separate column. (This is already implemented in the minecraft extension)

```kotlin
fun main() {
    //Your connection and table declaration...
	
    val world = table.select<World>(property("location.world")).first() //Select only the world
    table.update("location.world", value(world2)) //Update only the world
}
```

### Data Objects
DataObject is an interface that your row classes can implement to simplify some operations. 
It allows calling `.insert()`, `.update()` and `.delete()` directly on the object. 
It also has the methods `beforeWrite` and `afterRead` that you can override to listen to the corresponding event.

Lastly, it has a utility method that allows you to select a list of rows from another table, that reference this object with some column.