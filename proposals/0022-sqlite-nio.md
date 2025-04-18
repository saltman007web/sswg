# SQLiteNIO: An Swift-native driver for SQLite

* Proposal: [SSWG-0022](https://github.com/swift-server/sswg/blob/main/proposals/0022-sqlite-nio.md)
* Author(s): [Gwynne Raskind](https://github.com/gwynne)
* Review Manager: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Status: **Accepted (Sandbox)**
* Implementation: [vapor/sqlite-nio](https://github.com/vapor/sqlite-nio)
* Forum Threads: [Pitch](https://forums.swift.org/t/sqlitenio/46664/25), Discussion (Pending)

## Package Description

`SQLiteNIO` is a client package which provides an asynchronous Swift API for SQLite based on NIO's `EventLoopFuture`.

|||
|--:|:--|
| **Package Name** | sqlite-nio |
| **Module Name** | SQLiteNIO |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [MIT](https://choosealicense.com/licenses/mit/) |
| **Dependencies** | [swift-nio](https://github.com/apple/swift-nio), [swift-log](https://github.com/apple/swift-log) |

## Motiviation

SQLite is a C API which does not import into Swift in an easily managed form, especially in terms of marshalling data types and correctly handling the underlying API's multithreading requirements. However, SQLite - unlike most relational databases - is a complete database management library unto itself with no separate client or server components, reimplementing it in pure Swift would be, at best, a massive undertaking (and of questionable wisdom as well, especially considering the almost unheard-of _extremely_ high quality of SQLite's source code). As such, SQLiteNIO concerns itself solely with providing a more Swift-friendly, asynchronous interface to the C implementation. (Currently SQLiteNIO only supports `EventLoopFuture`-style asynchronous usage; providing a Concurrency-based alternative is planned in the near future.)

The underlying SQLite implementation is an embedded version of `libsqlite`, specifically the `sqlite3.c` amalgamation, built as a SwiftPM target and statically linked, with configuration flags chosen appropriately on a per-platform basis and vendor prefixes added to all APIs to avoid any collisions. The embedded source code is kept up to date with upstream releases (this is currently done by manually invoking a script when new releases are made; full automation is planned).

## Usage

This section goes into detail on a few distinct types from this module to give an idea of how they work together and what using the package looks like.

### SQLiteConnection

The base connection type, `SQLiteConnection`, provides the basic interface to an SQLite database stored on disk or in memory.

```swift
import SQLiteNIO

// create a new event loop group and thread pool
let elg = MultiThreadedEventLoopGroup(numberOfThreads: 1)
let pool = NIOThreadPool(numberOfThreads: 1)
pool.start()
defer {
    try! pool.syncShutdownGracefully()
    try! elg.syncShutdownGracefully()
}

// create a new connection to an in-memory database
let conn = try await SQLiteConnection.open(
    storage: .memory,
    threadPool: pool,
    on: elg.any()
).get()

// ready to query
print(conn) // SQLiteConnection
```

#### Closing

A connection _must_ be closed before it deinitializes. `SQLiteConnection` ensures this by asserting that it has been closed in its `deinit` method. This is meant to help developers implement proper graceful shutdown early and avoid leaking memory or sockets. 

### Query

Assuming we have an active `SQLiteConnection`, we can query the active database using SQLite's query command:

```swift
import SQiteLNIO

let conn = try await SQLiteConnection.open(...).get()
defer { try! conn.close().wait() }

// select the current version
let rows = try await conn.query("SELECT sqlite_version() AS version").get()
print(rows) // [SQLiteRow]

// fetch the version column from the first row by attempting to
// read it as a Swift string
let version = rows.first?.column("version")?.string
print(version) // Optional(3.39.5)
```

It is also possible to perform parameterized queries with an active `SQLiteConnection` by providing input parameter bindings.

Input parameters are passed as an array of `SQLiteData` objects along with the SQL string. In the query string, input parameters are referenced by one of the supported placeholder markers, such as `?` ("next" bound parameter) or `?N` (Nth bound parameter). Named parameter bindings are not yet supported.

```swift
import SQLiteNIO

let conn = try await SQLiteConnection.open(...).get()
defer { try! conn.close().wait() }

// selects all planets where name is equal to the first bound parameter
let rows = try await conn.query("SELECT * FROM planets WHERE name=?1", [.text("Earth")]).get()

// fetch the "name" column from the first row as a string
let foo = rows.first?.column("name")?.string
print(foo) // Optional("Earth")
```

### SQLiteData

`SQLiteData` represents data both going to and coming from SQLite. Data is represented by one of the four "basic" data types supported by SQLite: integer, float, text, or blob. The "typeless" `null` value is also represented.

### SQLiteRow

The `query()` method returns an array of `SQLiteRow`s. Each row can be thought of as a dictionary with column names as the key and data as the value. While the actual storage implementation is private, `SQLiteRow` provides a `columms` property for accessing all columns in a row as an array, and the `column(_:)` method for accessing an individual column's data by name:

```swift
struct SQLiteColumn {
    let name: String
    let data: SQLiteData
}

struct SQLiteRow {
    var columns: [SQLiteColumn]
    func column(_ name: String) -> SQLiteData?
}
```

If no column with the given name exists in the row, `column(_:)` returns `nil`.

### SQLiteError

The `SQLiteError` type provides a convenient interface to the various "well-known" SQLite result code constants.

```swift
enum SQLiteError: Error {
    let reason: Reason
    let message: String
    
    enum Reason {
        case error // SQLITE_ERROR
        case intern // SQLITE_INTERNAL
        case permission // SQLITE_PERM
        // etc.
    }
}
```

### Other features

SQLiteNIO also provides APIs for working with prepared statements (`SQLiteStatement`) and registering custom scalar and aggregate functions (`SQLiteCustomFunction`)

### How to use

To try this package out at home, add the following dependency to your `Package.swift` file:

```swift
.package(url: "https://github.com/vapor/sqlite-nio.git", from: "1.5.0"),
```

Then add `.product(name: "SQLiteNIO", package: "sqlite-nio")` to your module target's dependencies array.

### Seeking Feedback

* If anything, what does this proposal *not cover* that you will definitely need?
* If anything, what could we remove from this and still be happy?
* API-wise: what do you like, what don't you like?

Feel free to post feedback as response to this post and/or GitHub issues on [vapor/sqlite-nio](https://github.com/vapor/sqlite-nio).
