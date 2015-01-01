SQLite.jl
=========

[![Build Status](https://travis-ci.org/quinnj/SQLite.jl.png)](https://travis-ci.org/quinnj/SQLite.jl)
[![Coverage Status](https://img.shields.io/coveralls/quinnj/SQLite.jl.svg)](https://coveralls.io/r/quinnj/SQLite.jl)
[![SQLite](http://pkg.julialang.org/badges/SQLite_release.svg)](http://pkg.julialang.org/?pkg=SQLite&ver=release)

A Julia interface to the SQLite library and support for operations on DataFrames

**Installation**: `julia> Pkg.add("SQLite")`

## Package Documentation

#### Types/Functions
* `SQLiteDB(file::String; UTF16::Bool=false)`

  `SQLiteDB` requires the `file` string argument as the name of either a pre-defined SQLite database to be opened, or if the file doesn't exist, a database will be created.

  The `SQLiteDB` object represents a single connection to an SQLite database. All other SQLite.jl functions take an `SQLiteDB` as the first argument as context.

  The keyword argument `UTF16` can be set to true to force the creation of a database with UTF16-encoded strings.

  To create an in-memory temporary database, one can also call `SQLiteDB(":memory:")`.

* `close(db::SQLiteDB)`

  Closes an open database connection.

* `SQLiteStmt(db::SQLiteDB, sql::String)`

  Constructs and prepares (compiled by SQLite library) an SQL statement in the context of the provided `db`. Note the SQL statement is not actually executed, but only compiled (mainly for usage where the same statement is repeated with different parameters bound as values. See `bind` below).

* `close(stmt::SQLiteStmt)`

  Closes or finalizes an SQLiteStmt. A closed `SQLiteStmt` can no longer be executed.

* `bind(stmt::SQLiteStmt,index,value)`

  Used to bind values to parameter placeholders in an prepared `SQLiteStmt`. From the SQLite documentation:

  > Usually, though, it is not useful to evaluate exactly the same SQL statement more than once. More often, one wants to evaluate similar statements. For example, you might want to evaluate an INSERT statement multiple times though with different values to insert. To accommodate this kind of flexibility, SQLite allows SQL statements to contain parameters which are "bound" to values prior to being evaluated. These values can later be changed and the same prepared statement can be evaluated a second time using the new values.

  > In SQLite, wherever it is valid to include a string literal, one can use a parameter in one of the following forms:

  > ?
  > ?NNN
  > :AAA
  > $AAA
  > @AAA

  > In the examples above, NNN is an integer value and AAA is an identifier. A parameter initially has a value of NULL. Prior to calling sqlite3_step() for the first time or immediately after sqlite3_reset(), the application can invoke one of the sqlite3_bind() interfaces to attach values to the parameters. Each call to sqlite3_bind() overrides prior bindings on the same parameter.

* `execute(stmt::SQLiteStmt)`
  `execute(db::SQLiteDB, sql::String)`

  Used to execute prepared `SQLiteStmt`. The 2nd method is a convenience method to pass in an SQL statement as a string which gets prepared and executed in one call. This method does not check for or return any results, hence it is only useful for database manipulation methods (i.e. ALTER, CREATE, UPDATE, DROP). To return results, see `query` below. Also consider the `create`, `drop`, and `append` methods for manipulation statements as further SQLite performance tricks are incorporated automatically.

* `query(db::SQLiteDB, sql::String, values=[])`
  
  An SQL statement `sql` is prepared, executed in the context of `db`, and results, if any, are returned. The return values are a `(String[],Any[])` tuple representing `(column names, result values)`.

  The values in `values` are used in parameter binding (see `bind` above). If your statement uses nameless parameters `values` must be a `Vector` of the values you wish to bind to your statment. If your statement uses named parameters `values` must be a Dict where the keys are of type `Symbol`. The key must match an identifier name in the statement (the name **does not** include the ':', '@' or '$' prefix).

* `create(db::SQLiteDB,name::String,table::AbstractMatrix,
            colnames=String[],coltypes=DataType[];temp::Bool=false)`

  Convenience method for "CREATE TABLE" and "INSERT" statements to insert `table` as an SQLite table in the `db` database. `name` will be the name of the SQLite table. `table` can be any AbstractMatrix that supports the `table[i,j]` getindex method. `colnames` is an optional vector to be used as the names of the columns for the SQLite table. `coltypes` is also an optional vector to specify the Julia types of the columns in `table`. The optional keyword `temp` can be set to `true` to specify the creation of a temporary table that will be destroyed when the database connection is closed.

  This method automatically takes care of SQLite transaction handling and other performance enhancements.

* `append(db::SQLiteDB,name::String,table::AbstractMatrix)`

  Takes the values in `table` and appends (by repeated inserts) to the SQLite table `name`. No column checking is done to ensure correct types, so care should be taken as SQLite is "typeless" in that it allows items of any type to be stored in columns. Transaction handling is automatic as well as performance enhancements.

* `drop(db::SQLiteDB,table::String)`

  `drop` is pretty self-explanatory. It's really just a convenience wrapper around `query` to execute a DROP TABLE command, while also calling "VACUUM" to clean out freed memory from the database.

* `register(db::SQLiteDB, func::Function; nargs::Int=-1, name::AbstractString=string(func), isdeterm::Bool=true)`
* `register(db::SQLiteDB, init, step::Function, final::Function=identity; nargs::Int=-1, name::AbstractString=string(final), isdeterm::Bool=true)`

  Register a scalar (first method) or aggregate (second method) function with a `SQLiteDB`.

* `@register db function`

  Automatically define then register `function` with a `SQLiteDB`.

* `sr"..."`

  This string literal is used to escape all special characters in the string, useful for using regex in a query.

* `sqlreturn(contex, val)`

  This function should never be called explicitly. Instead it is exported so that it can be overloaded when necessary, see below.

#### User Defined Functions

##### SQLite Regular Expressions

SQLite provides syntax for calling the [`regexp` function](http://sqlite.org/lang_expr.html#regexp) from inside `WHERE` clauses. Unfortunately, however, SQLite does not provide a default implementation of the `regexp` function so SQLite.jl creates one automatically when you open a database. The function can be called in the following ways (examples using the [Chinook Database](http://chinookdatabase.codeplex.com/))

```julia
julia> using SQLite

julia> db = SQLiteDB("Chinook_Sqlite.sqlite")

julia> # using SQLite's in-built syntax

julia> query(db, "SELECT FirstName, LastName FROM Employee WHERE LastName REGEXP 'e(?=a)'")
1x2 ResultSet
| Row | "FirstName" | "LastName" |
|-----|-------------|------------|
| 1   | "Jane"      | "Peacock"  |

julia> # explicitly calling the regexp() function

julia> query(db, "SELECT * FROM Genre WHERE regexp('e[trs]', Name)")
6x2 ResultSet
| Row | "GenreId" | "Name"               |
|-----|-----------|----------------------|
| 1   | 3         | "Metal"              |
| 2   | 4         | "Alternative & Punk" |
| 3   | 6         | "Blues"              |
| 4   | 13        | "Heavy Metal"        |
| 5   | 23        | "Alternative"        |
| 6   | 25        | "Opera"              |

julia> # you can even do strange things like this if you really want

julia> query(db, "SELECT * FROM Genre ORDER BY GenreId LIMIT 2")
2x2 ResultSet
| Row | "GenreId" | "Name" |
|-----|-----------|--------|
| 1   | 1         | "Rock" |
| 2   | 2         | "Jazz" |

julia> query(db, "INSERT INTO Genre VALUES (regexp('^word', 'this is a string'), 'My Genre')")
1x1 ResultSet
| Row | "Rows Affected" |
|-----|-----------------|
| 1   | 0               |

julia> query(db, "SELECT * FROM Genre ORDER BY GenreId LIMIT 2")
2x2 ResultSet
| Row | "GenreId" | "Name"     |
|-----|-----------|------------|
| 1   | 0         | "My Genre" |
| 2   | 1         | "Rock"     |
```

Due to the heavy use of escape characters you may run into problems where julia parses out some backslashes in your query, for example `"\y"` simlpy becomes `"y"`. For example the following two queries are identical

```julia
julia> query(db, "SELECT * FROM MediaType WHERE Name REGEXP '-\d'")
1x1 ResultSet
| Row | "Rows Affected" |
|-----|-----------------|
| 1   | 0               |

julia> query(db, "SELECT * FROM MediaType WHERE Name REGEXP '-d'")
1x1 ResultSet
| Row | "Rows Affected" |
|-----|-----------------|
| 1   | 0               |
```

This can be avoided in two ways. You can either escape each backslash yourself or you can use the sr"..." string literal that SQLite.jl exports. The previous query can then successfully be run like so

```julia
julia> # manually escaping backslashes

julia> query(db, "SELECT * FROM MediaType WHERE Name REGEXP '-\\d'")
1x2 ResultSet
| Row | "MediaTypeId" | "Name"                        |
|-----|---------------|-------------------------------|
| 1   | 3             | "Protected MPEG-4 video file" |

julia> # using sr"..."

julia> query(db, sr"SELECT * FROM MediaType WHERE Name REGEXP '-\d'")
1x2 ResultSet
| Row | "MediaTypeId" | "Name"                        |
|-----|---------------|-------------------------------|
| 1   | 3             | "Protected MPEG-4 video file" |
```

The sr"..." currently escapes all special characters in a string but it may be changed in the future to escape only characters which are part of a regex.

##### Custom Scalar Functions

SQLite.jl also provides a way that you can implement your own [Scalar Functions](https://www.sqlite.org/lang_corefunc.html). This is done using the `register` function and macro.

`@register` takes a `SQLiteDB` and a function. The function can be in block syntax

```julia
julia> @register db function add3(x)
       x + 3
       end
```

inline function syntax

```julia
julia> @register db mult3(x) = 3 * x
```

and previously defined functions

```julia
julia> @register db sin
```

The `register` function takes optional arguments; `nargs` which defaults to `-1`, `name` which defaults to the name of the function, `isdeterm` which defaults to `true`. In practice these rarely need to be used.

The `register` function uses the `sqlreturn` function to return your function's return value to SQLite. By default, `sqlreturn` maps the returned value to a [native SQLite type](http://sqlite.org/c3ref/result_blob.html) or, failing that, serializes the julia value and stores it as a `BLOB`. To change this behaviour simply define a new method for `sqlreturn` which then calls a previously defined method for `sqlreturn`. Methods which map to native SQLite types are

```julia
sqlreturn(context, ::NullType)
sqlreturn(context, val::Int32)
sqlreturn(context, val::Int64)
sqlreturn(context, val::Float64)
sqlreturn(context, val::UTF16String)
sqlreturn(context, val::String)
sqlreturn(context, val::Any)
```

As an example, say you would like `BigInt`s to be stored as `TEXT` rather than a `BLOB`. You would simply need to define the following method

```julia
sqlreturn(context, val::BigInt) = sqlreturn(context, string(val))
```

Another example is the `sqlreturn` used by the `regexp` function. For `regexp` to work correctly it must return it must return an `Int` (more specifically a `0` or `1`) but `ismatch` (used by `regexp`) returns a `Bool`. For this reason the following method was defined

```julia
sqlreturn(context, val::Bool) = sqlreturn(context, int(val))
```

Any new method defined for `sqlreturn` must take two arguments and must pass the first argument straight through as the first argument.

#### Custom Aggregate Functions

Using the `register` function, you can also define your own aggregate functions with largely the same semantics.

The `register` function for aggregates must take a `SQLiteDB`, an initial value, a step function and a final function. The first argument to the step function will be the return value of the previous function (or the initial value if it is the first iteration). The final function must take a single argument which will be the return value of the last step function.

```julia
julia> dsum(prev, cur) = prev + cur

julia> dsum(prev) = 2 * prev

julia> register(db, 0, dsum, dsum)
```

If no name is given the name of the first (step) function is used (in this case "dsum"). You can also use lambdas, the following does the same as the previous code snippet

```julia
julia> register(db, 0, (p,c) -> p+c, p -> 2p, name="dsum")
```
