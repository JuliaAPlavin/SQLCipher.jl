# SQLCipher

Drop-in replacement for [SQLite.jl](https://github.com/JuliaDatabases/SQLite.jl), but uses [SQLCipher](https://github.com/sqlcipher/sqlcipher) instead of plain SQLite. This is a fork of `SQLite.jl`, keeping the same underlying code and package versions.

See `SQLite.jl` docs for the interface and usage information, and SQLCipher docs for details on encryption features.

Briefly, encryption is enabled by running
```julia
SQLite.execute(db, """PRAGMA key="<your password>" """)
```
after opening the database.
