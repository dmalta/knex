# Creating a Knex Driver for IBM DB2 for z/OS

This guide provides comprehensive reference documentation for implementing a custom Knex dialect targeting **IBM DB2 for z/OS**. It covers every class you must extend or implement, explains every method you need to override, and explains the DB2 for z/OS-specific SQL differences you must account for. Use the existing MSSQL and Oracle drivers as the closest analogies throughout.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Class Hierarchy](#class-hierarchy)
3. [Client (Entry Point)](#1-client-entry-point)
4. [QueryCompiler (DML)](#2-querycompiler-dml)
5. [QueryBuilder (DML)](#3-querybuilder-dml)
6. [SchemaCompiler (DDL – Top Level)](#4-schemacompiler-ddl--top-level)
7. [TableCompiler (DDL – Table Level)](#5-tablecompiler-ddl--table-level)
8. [ColumnCompiler (DDL – Column Level)](#6-columncompiler-ddl--column-level)
9. [Transaction](#7-transaction)
10. [Formatter / Identifier Quoting](#8-formatter--identifier-quoting)
11. [Wire Protocol / Runner Hooks](#9-wire-protocol--runner-hooks)
12. [File Layout Recommendation](#10-file-layout-recommendation)
13. [Registration / Usage](#11-registration--usage)
14. [Quick-Reference: DB2 for z/OS SQL Specifics](#12-quick-reference-db2-for-zos-sql-specifics)

---

## Architecture Overview

Knex separates **building** from **compiling**. The builder classes (QueryBuilder, SchemaBuilder, TableBuilder, ColumnBuilder) are the public API that users call. The compiler classes (QueryCompiler, SchemaCompiler, TableCompiler, ColumnCompiler) traverse those builder objects and produce raw SQL strings and bindings arrays.

The **Client** class is the central registry that wires everything together and handles the actual database connection. When you create a dialect, you subclass `Client` and return your dialect-specific compiler and builder instances from its factory methods.

```
User code
    │
    ▼
SchemaBuilder / QueryBuilder / ColumnBuilder / TableBuilder
    │   (records method calls as a sequence of plain objects)
    │
    ▼
SchemaCompiler / QueryCompiler / TableCompiler / ColumnCompiler
    │   (converts recorded calls to SQL strings + binding arrays)
    │
    ▼
Client  ──►  Runner  ──►  DB driver (e.g. ibm_db / odbc)
    │             │
    │             └─► executes SQL, parses response rows
    └─► Transaction (wraps runner with BEGIN/COMMIT/ROLLBACK)
```

---

## Class Hierarchy

```
lib/client.js                     ← base Client
lib/query/querycompiler.js        ← base QueryCompiler
lib/query/querybuilder.js         ← base QueryBuilder (usually reused as-is)
lib/schema/compiler.js            ← base SchemaCompiler
lib/schema/tablecompiler.js       ← base TableCompiler
lib/schema/columncompiler.js      ← base ColumnCompiler
lib/execution/transaction.js      ← base Transaction
lib/formatter.js                  ← base Formatter

Your dialect:
  lib/dialects/db2-zos/index.js                       ← Client_DB2ZOS
  lib/dialects/db2-zos/query/db2zos-querycompiler.js  ← QueryCompiler_DB2ZOS
  lib/dialects/db2-zos/schema/db2zos-compiler.js      ← SchemaCompiler_DB2ZOS
  lib/dialects/db2-zos/schema/db2zos-tablecompiler.js ← TableCompiler_DB2ZOS
  lib/dialects/db2-zos/schema/db2zos-columncompiler.js← ColumnCompiler_DB2ZOS
  lib/dialects/db2-zos/transaction.js                 ← Transaction_DB2ZOS
```

---

## 1. Client (Entry Point)

**Base class:** `lib/client.js`

The Client class is the root object. A Knex instance is created by passing `{ client: 'db2-zos' }` (or the class itself) to `knex()`. Every compiler and builder instance is created via factory methods on the client.

### Mandatory properties (set on prototype)

| Property | Type | Example | Notes |
|---|---|---|---|
| `dialect` | string | `'db2-zos'` | Identifies the dialect in logs |
| `driverName` | string | `'ibm_db'` | Name of the npm package; triggers `initializeDriver()` |

### Factory methods to override

Every factory method must return an instance of your dialect-specific subclass:

```js
// lib/dialects/db2-zos/index.js
const Client = require('../../client');
const QueryCompiler = require('./query/db2zos-querycompiler');
const SchemaCompiler = require('./schema/db2zos-compiler');
const TableCompiler  = require('./schema/db2zos-tablecompiler');
const ColumnCompiler = require('./schema/db2zos-columncompiler');
const Transaction    = require('./transaction');

class Client_DB2ZOS extends Client {
  queryCompiler(builder, formatter) {
    return new QueryCompiler(this, builder, formatter);
  }
  schemaCompiler(builder) {
    return new SchemaCompiler(this, builder);
  }
  tableCompiler(tableBuilder) {
    return new TableCompiler(this, tableBuilder);
  }
  columnCompiler(tableCompiler, columnBuilder) {
    return new ColumnCompiler(this, tableCompiler, columnBuilder);
  }
  transaction(container, config, outerTx) {
    return new Transaction(this, container, config, outerTx);
  }
}

Object.assign(Client_DB2ZOS.prototype, {
  dialect:    'db2-zos',
  driverName: 'ibm_db',   // or 'odbc', depending on your chosen npm driver
});

module.exports = Client_DB2ZOS;
```

### Connection lifecycle methods

These must all be overridden to use your chosen Node.js DB2 driver. The pool calls `acquireRawConnection()` to create new connections and `destroyRawConnection()` to close them.

#### `_driver()`

Returns the required npm module. Called once during `initializeDriver()`.

```js
_driver() {
  return require('ibm_db'); // or require('odbc')
}
```

#### `acquireRawConnection() → Promise<connection>`

Opens a new raw connection to DB2 and returns it. The connection object is stored in the pool and passed to `_query()` / `_stream()`.

```js
acquireRawConnection() {
  return new Promise((resolve, reject) => {
    const connStr = this._buildConnectionString();
    this.driver.open(connStr, (err, conn) => {
      if (err) return reject(err);
      conn.__knex__disposed = false;
      resolve(conn);
    });
  });
}
```

#### `validateConnection(connection) → boolean`

Called by the pool before reusing a connection. Return `true` if the connection is still alive.

```js
validateConnection(connection) {
  return connection && !connection.__knex__disposed;
}
```

#### `destroyRawConnection(connection) → Promise<void>`

Called by the pool when a connection should be closed.

```js
destroyRawConnection(connection) {
  return new Promise((resolve, reject) => {
    connection.close((err) => {
      if (err) return reject(err);
      resolve();
    });
  });
}
```

#### `_query(connection, queryObject) → Promise<queryObject>`

Executes a single SQL statement. Mutates `queryObject.response` with the raw result rows. This is the lowest-level execution hook.

```js
_query(connection, queryObject) {
  return new Promise((resolve, reject) => {
    const { sql, bindings } = queryObject;
    connection.query(sql, bindings || [], (err, rows) => {
      if (err) return reject(err);
      queryObject.response = rows;
      resolve(queryObject);
    });
  });
}
```

#### `processResponse(queryObject, runner) → any`

Converts the raw driver response into the value Knex returns to the user. Must handle all DML verbs. The `method` property on `queryObject` is one of `'select'`, `'first'`, `'pluck'`, `'insert'`, `'update'`, `'del'`, `'counter'`, `'raw'`.

```js
processResponse(queryObject, runner) {
  if (queryObject == null) return;
  const { response, method } = queryObject;
  if (queryObject.output) return queryObject.output.call(runner, response);
  switch (method) {
    case 'select': return response;
    case 'first':  return response[0];
    case 'pluck':  return response.map(row => row[queryObject.pluck]);
    case 'insert':
    case 'del':
    case 'update':
    case 'counter': return response;
    default: return response;
  }
}
```

### Binding / identifier methods

#### `positionBindings(sql) → string`

DB2 uses `?` placeholders natively — the same as MySQL. However, many DB2 CLI drivers (ODBC, ibm_db) already accept `?` markers, so you may return `sql` unchanged. If your driver uses named or positional markers (`@p0`, `:1`) override this method.

```js
// DB2 CLI / ibm_db accepts '?' natively — no-op is fine
positionBindings(sql) {
  return sql;  // keep '?'
}
```

If you need ODBC-style `?` but your base implementation already does that, inherit from the base class without override.

#### `wrapIdentifierImpl(value) → string`

DB2 for z/OS uses double-quotes as the standard identifier delimiter (ANSI SQL). The base `Client` already implements `"value"`. You should still keep it explicit:

```js
wrapIdentifierImpl(value) {
  return value === '*' ? '*' : `"${value.replace(/"/g, '""')}"`;
}
```

> **Note:** DB2 object names default to **UPPERCASE** internally. You may want to call `.toUpperCase()` on the value here, or document that users must use uppercase names. This is a critical difference from other dialects.

#### `database() → string`

For DB2 for z/OS, the concept equivalent to "database" is the **subsystem** or the schema. Return the configured `database` connection option.

```js
database() {
  return this.connectionSettings.database;
}
```

---

## 2. QueryCompiler (DML)

**Base class:** `lib/query/querycompiler.js`  
**Analogous dialects:** MSSQL (`lib/dialects/mssql/query/mssql-querycompiler.js`), Oracle (`lib/dialects/oracle/query/oracle-querycompiler.js`)

The `QueryCompiler` converts the query builder's internal statement tree into SQL strings. Every public method on this class corresponds to one SQL verb or clause.

### Constructor

Always call `super(client, builder, formatter)`. You may throw on unsupported features in the constructor (e.g., `.onConflict()`).

```js
const QueryCompiler = require('../../../query/querycompiler');

class QueryCompiler_DB2ZOS extends QueryCompiler {
  constructor(client, builder, formatter) {
    super(client, builder, formatter);

    if (this.single.onConflict) {
      throw new Error('.onConflict() is not supported for db2-zos.');
    }

    // DB2 uses 'DEFAULT VALUES' for an empty insert
    this._emptyInsertValue = 'DEFAULT VALUES';
  }
}
```

### `select() → string`

Assembles the full SELECT statement. The base implementation joins the `components` array in order. DB2 does not have a `WITH (NOLOCK)` clause or `TOP N` syntax. Override only to change the component list.

DB2 for z/OS supports `FETCH FIRST n ROWS ONLY` for pagination (see `limit()` / `offset()` below), so the component list stays the same as the base class. The base `select()` is usually sufficient.

### `insert() → string | object`

DB2 requires standard `INSERT INTO table (cols) VALUES (...)` syntax. The base implementation covers the common case. The only specialization needed:

1. **Empty insert** — DB2 uses `INSERT INTO t (col) VALUES (DEFAULT)` rather than `DEFAULT VALUES`. You may need to point at a single nullable column or use a `DEFAULT VALUES` variant depending on DB2 version.
2. **RETURNING / identity** — DB2 for z/OS does not support `RETURNING`. Use `SELECT IDENTITY_VAL_LOCAL()` in a subsequent query to retrieve the last generated identity value.

```js
insert() {
  // DB2 does not support ON CONFLICT or RETURNING in INSERT.
  // Fall through to the base implementation for standard inserts.
  const sql = super.insert();
  if (this.single.returning) {
    // DB2 for z/OS: identity retrieval must be done separately.
    // Return a structured object that processResponse() can handle.
    return { sql, returning: this.single.returning };
  }
  return sql;
}
```

### `update() → string`

DB2 supports standard `UPDATE t SET col = val WHERE ...`. The base implementation is correct. Override only if you need to support `UPDATE ... FROM` join semantics (which DB2 handles via a correlated subquery or a `MERGE` statement instead).

### `del() → string`

The base implementation produces `DELETE FROM table WHERE ...`. DB2 for z/OS accepts this. No override required unless you need `DELETE ... FROM` join syntax — in that case use a subquery approach.

### `truncate() → string`

DB2 for z/OS supports `TRUNCATE TABLE name` (introduced in V9). Older systems (V8) used `DELETE FROM name` without a WHERE clause. The base class produces `truncate tableName` (no `TABLE` keyword). Override:

```js
truncate() {
  // DB2 for z/OS V9+:  TRUNCATE TABLE t IMMEDIATE
  // Older V8/pre-V9:   DELETE FROM t (no TRUNCATE support)
  return `TRUNCATE TABLE ${this.tableName} IMMEDIATE`;
}
```

### `limit() → string`

DB2 for z/OS does **not** support `LIMIT n`. It uses `FETCH FIRST n ROWS ONLY` appended to the query. Override:

```js
limit() {
  const noLimit = !this.single.limit && this.single.limit !== 0;
  if (noLimit) return '';
  // Do not emit here — emitted by offset() or appended at end of select()
  return '';
}
```

### `offset() → string`

DB2 for z/OS V12+ supports `OFFSET n ROWS FETCH NEXT m ROWS ONLY`. Earlier versions require wrapping the query in a `ROW_NUMBER()` subquery.

```js
offset() {
  const noLimit  = !this.single.limit && this.single.limit !== 0;
  const noOffset = !this.single.offset;

  if (noLimit && noOffset) return '';

  // DB2 V12+ (preferred):
  const offsetClause = noOffset
    ? ''
    : `OFFSET ${this._getValueOrParameterFromAttribute('offset')} ROWS `;
  const fetchClause  = noLimit
    ? ''
    : `FETCH NEXT ${this._getValueOrParameterFromAttribute('limit')} ROWS ONLY`;

  return (offsetClause + fetchClause).trim();
}
```

For **pre-V12 compatibility**, wrap the whole `select()` result in a `ROW_NUMBER()` outer query inside the `select()` override:

```js
// Pre-V12 OFFSET emulation via ROW_NUMBER()
select() {
  const inner  = super.select();
  const limit  = this.single.limit;
  const offset = this.single.offset;

  const hasLimit  = limit != null;
  const hasOffset = offset != null;

  if (!hasOffset) return inner;   // FETCH FIRST handled in offset()

  const start = Number(offset) + 1;
  const end   = hasLimit ? Number(offset) + Number(limit) : null;

  return (
    `SELECT * FROM (SELECT inner__.*, ROW_NUMBER() OVER() AS "rn__" ` +
    `FROM (${inner}) AS inner__) AS outer__ ` +
    `WHERE "rn__" >= ${start}` +
    (end != null ? ` AND "rn__" <= ${end}` : '')
  );
}
```

### `forUpdate() → string`

DB2 uses `FOR UPDATE` or `FOR UPDATE WITH RS` (repeatable-read isolation).

```js
forUpdate() {
  return 'FOR UPDATE WITH RS';
}
```

### `forShare() → string`

DB2 uses `FOR FETCH ONLY` or `FOR READ ONLY` for read locks.

```js
forShare() {
  return 'FOR FETCH ONLY';
}
```

### `columnInfo() → object`

Returns a query object (with `.sql`, `.bindings`, and `.output`) that queries DB2 system catalog tables to describe a table's columns. In DB2 for z/OS the catalog is `SYSIBM.SYSCOLUMNS`.

```js
columnInfo() {
  const column = this.single.columnInfo;
  const table  = this.client.customWrapIdentifier(
    this.single.table,
    (v) => v  // identity — no quoting for catalog values
  ).toUpperCase(); // DB2 stores names in UPPERCASE

  const schema   = (this.single.schema || this.client.connectionSettings.currentSchema || '').toUpperCase();
  const bindings = [table];
  let sql = `SELECT NAME, COLTYPE, LENGTH, SCALE, NULLS, DEFAULT
             FROM SYSIBM.SYSCOLUMNS
             WHERE TBNAME = ?`;

  if (schema) {
    sql += ' AND TBCREATOR = ?';
    bindings.push(schema);
  }

  return {
    sql,
    bindings,
    output(resp) {
      const out = resp.reduce((cols, row) => {
        cols[row.NAME.trim()] = {
          type:         row.COLTYPE.trim(),
          maxLength:    row.LENGTH,
          scale:        row.SCALE,
          nullable:     row.NULLS === 'Y',
          defaultValue: row.DEFAULT,
        };
        return cols;
      }, {});
      return (column && out[column.toUpperCase()]) || out;
    },
  };
}
```

> **Important:** DB2 for z/OS stores all unquoted object names as UPPERCASE in the system catalog. Normalize table/column name inputs with `.toUpperCase()` when querying catalog tables.

### `whereLike() / whereILike()`

DB2 uses `LIKE` for case-insensitive comparison depending on the collation. DB2 for z/OS is case-sensitive by default. To implement case-insensitive `LIKE`, wrap the column in `UPPER()`:

```js
whereILike(statement) {
  return `UPPER(${this._columnClause(statement)}) ${this._not(statement, 'LIKE ')}UPPER(${this._valueClause(statement)})`;
}
```

### `with() / CTE`

DB2 for z/OS V9+ supports `WITH cte AS (...)` common table expressions. The base implementation generates standard CTE syntax and should work unchanged. Recursive CTEs (`WITH RECURSIVE`) use `WITH` + a self-referential union in DB2 (the `RECURSIVE` keyword is not used — identical to MSSQL).

```js
with() {
  // Strip 'recursive' keyword — DB2 does not use it
  const undoList = [];
  if (this.grouped.with) {
    for (const stmt of this.grouped.with) {
      if (stmt.recursive) {
        undoList.push(stmt);
        stmt.recursive = false;
      }
    }
  }
  const result = super.with();
  for (const stmt of undoList) stmt.recursive = true;
  return result;
}
```

---

## 3. QueryBuilder (DML)

**Base class:** `lib/query/querybuilder.js`

In most cases you do **not** need a custom QueryBuilder for DB2. The base class implements all builder methods (`select`, `where`, `join`, `limit`, etc.) as chainable calls that push plain objects onto `this._statements`. The QueryCompiler then reads those objects.

Override `QueryBuilder` only if you need to:
- **Disallow** certain methods (e.g., `.onConflict()` — throw in the constructor, not here)
- **Add new DB2-specific builder methods** (e.g., `.isolation()`, `.queryOptimization()`)

If you do create a custom QueryBuilder, register it in your Client:

```js
const QueryBuilder = require('../../query/querybuilder');

class QueryBuilder_DB2ZOS extends QueryBuilder {
  // Example: DB2-specific OPTIMIZE FOR n ROWS hint
  optimizeFor(n) {
    this._single.optimizeFor = n;
    return this;
  }
}
```

Then in the QueryCompiler, read `this.single.optimizeFor` and append `OPTIMIZE FOR n ROWS` to the SELECT.

---

## 4. SchemaCompiler (DDL – Top Level)

**Base class:** `lib/schema/compiler.js`  
**Analogous dialect:** MSSQL (`lib/dialects/mssql/schema/mssql-compiler.js`), PostgreSQL (`lib/dialects/postgres/schema/pg-compiler.js`)

`SchemaCompiler` handles the top-level DDL operations dispatched from `knex.schema.*`. It converts method names like `createTable`, `dropTable`, `hasTable` into SQL statements.

```js
const SchemaCompiler = require('../../../schema/compiler');

class SchemaCompiler_DB2ZOS extends SchemaCompiler {
  constructor(client, builder) {
    super(client, builder);
  }
}

// DB2 uses uppercase DROP TABLE by convention
SchemaCompiler_DB2ZOS.prototype.dropTablePrefix = 'DROP TABLE ';

module.exports = SchemaCompiler_DB2ZOS;
```

### Methods to override

#### `hasTable(tableName)`

Queries `SYSIBM.SYSTABLES` to check existence.

```js
hasTable(tableName) {
  const table    = tableName.toUpperCase();
  const schema   = (this.schema || this.client.connectionSettings.currentSchema || '').toUpperCase();
  const bindings = [table];

  let sql = `SELECT 1 FROM SYSIBM.SYSTABLES WHERE NAME = ? AND TYPE = 'T'`;
  if (schema) {
    sql += ' AND CREATOR = ?';
    bindings.push(schema);
  }

  this.pushQuery({
    sql,
    bindings,
    output: (resp) => resp.length > 0,
  });
}
```

#### `hasColumn(tableName, columnName)`

Queries `SYSIBM.SYSCOLUMNS`.

```js
hasColumn(tableName, columnName) {
  const table    = tableName.toUpperCase();
  const column   = columnName.toUpperCase();
  const schema   = (this.schema || this.client.connectionSettings.currentSchema || '').toUpperCase();
  const bindings = [table, column];

  let sql = `SELECT 1 FROM SYSIBM.SYSCOLUMNS WHERE TBNAME = ? AND NAME = ?`;
  if (schema) {
    sql += ' AND TBCREATOR = ?';
    bindings.push(schema);
  }

  this.pushQuery({
    sql,
    bindings,
    output: (resp) => resp.length > 0,
  });
}
```

#### `renameTable(from, to)`

DB2 for z/OS uses `RENAME TABLE old TO new` (V8+).

```js
renameTable(from, to) {
  this.pushQuery(
    `RENAME TABLE ${this.formatter.wrap(prefixedTableName(this.schema, from))} TO ${this.formatter.wrap(to)}`
  );
}
```

#### `dropTableIfExists(tableName)`

DB2 for z/OS does **not** support `DROP TABLE IF EXISTS`. You must check `SYSIBM.SYSTABLES` first, or catch the SQLCODE -204 error. Here is the safe approach using a conditional pattern:

```js
dropTableIfExists(tableName) {
  const name   = this.formatter.wrap(prefixedTableName(this.schema, tableName));
  const raw    = tableName.toUpperCase();
  const schema = (this.schema || '').toUpperCase();

  // Emit a conditional drop using a helper stored procedure pattern,
  // or check existence first. Some teams use a shell stored-procedure.
  // The simplest approach that works in a script context:
  this.pushQuery({
    sql: `BEGIN
            DECLARE CONTINUE HANDLER FOR SQLSTATE '42704' BEGIN END;
            EXECUTE IMMEDIATE 'DROP TABLE ${name}';
          END`,
    // Note: This is COMPOUND SQL (dynamic). Requires DB2 V9+.
    // For V8 compatibility, check SYSIBM.SYSTABLES in a separate query.
  });
}
```

> **DB2 z/OS alternative (pre-V9):** issue a `SELECT COUNT(*) FROM SYSIBM.SYSTABLES WHERE ...` check first, then conditionally issue `DROP TABLE`.

#### Schema / Collection operations

DB2 for z/OS uses **schemas** (called *collections* in earlier documentation). The base `createSchema` / `dropSchema` throw a "Postgres only" error. Override them:

```js
createSchema(schemaName) {
  // DB2 for z/OS: CREATE SCHEMA creates an implicit schema
  this.pushQuery(`CREATE SCHEMA ${this.formatter.wrap(schemaName)}`);
}

createSchemaIfNotExists(schemaName) {
  // No IF NOT EXISTS for CREATE SCHEMA in DB2 — check SYSIBM.SYSSCHEMAS
  const sql = schemaName.toUpperCase();
  this.pushQuery({
    sql: `SELECT 1 FROM SYSIBM.SYSSCHEMAS WHERE SCHEMANAME = ?`,
    bindings: [sql],
    output: (rows) => {
      if (rows.length === 0) {
        return this.client.raw(`CREATE SCHEMA ${this.formatter.wrap(schemaName)}`);
      }
    },
  });
}

dropSchema(schemaName) {
  this.pushQuery(`DROP SCHEMA ${this.formatter.wrap(schemaName)} RESTRICT`);
}

dropSchemaIfExists(schemaName) {
  // Same issue as dropTableIfExists — no native IF EXISTS
  const name = this.formatter.wrap(schemaName);
  this.pushQuery({
    sql: `BEGIN
            DECLARE CONTINUE HANDLER FOR SQLSTATE '42704' BEGIN END;
            EXECUTE IMMEDIATE 'DROP SCHEMA ${name} RESTRICT';
          END`,
  });
}
```

#### `generateDdlCommands()`

The base `SchemaCompiler.generateDdlCommands()` returns `{ pre: [], sql: [...], check: null, post: [] }`. DB2 for z/OS sometimes requires commands to be issued in a specific order (e.g., auxiliary tables for LOB columns). Override this method if your driver needs to emit pre/post DDL.

---

## 5. TableCompiler (DDL – Table Level)

**Base class:** `lib/schema/tablecompiler.js`  
**Analogous dialects:** MSSQL (`lib/dialects/mssql/schema/mssql-tablecompiler.js`), Oracle (`lib/dialects/oracle/schema/oracle-tablecompiler.js`)

`TableCompiler` builds the DDL for individual tables — `CREATE TABLE`, `ALTER TABLE`, and all index/constraint operations.

### Prototype properties (override on the prototype)

```js
TableCompiler_DB2ZOS.prototype.lowerCase            = false; // DB2 uses UPPERCASE keywords
TableCompiler_DB2ZOS.prototype.addColumnsPrefix     = 'ADD COLUMN ';
TableCompiler_DB2ZOS.prototype.alterColumnsPrefix   = 'ALTER COLUMN ';
TableCompiler_DB2ZOS.prototype.dropColumnPrefix     = 'DROP COLUMN ';
// Methods that should be inlined in CREATE TABLE rather than separate ALTER TABLE:
TableCompiler_DB2ZOS.prototype.createAlterTableMethods = ['foreign', 'primary'];
```

### `createQuery(columns, ifNot, like)`

Emits the `CREATE TABLE` statement.

```js
createQuery(columns, ifNot, like) {
  let createStatement = 'CREATE TABLE ';

  if (like) {
    // DB2 for z/OS does not have CREATE TABLE ... LIKE with including constraints.
    // Use: CREATE TABLE new AS (SELECT * FROM old) WITH NO DATA
    createStatement += `${this.tableName()} AS (SELECT * FROM ${this.tableNameLike()}) WITH NO DATA`;
  } else {
    createStatement +=
      this.tableName() +
      (this._formatting ? ' (\n    ' : ' (') +
      columns.sql.join(this._formatting ? ',\n    ' : ', ') +
      this._addChecks() +
      ')';
  }

  this.pushQuery(createStatement);

  if (this.single.comment) this.comment(this.single.comment);
  if (like) this.addColumns(columns, this.addColumnsPrefix);
}
```

> **DB2 for z/OS note:** `IF NOT EXISTS` is not valid in `CREATE TABLE`. Use the `hasTable()` check on `SchemaCompiler` before issuing `createTable`.

### `comment(comment)`

DB2 for z/OS uses `COMMENT ON TABLE ... IS '...'`.

```js
comment(comment) {
  this.pushQuery(
    `COMMENT ON TABLE ${this.tableName()} IS '${comment.replace(/'/g, "''")}'`
  );
}
```

### `addColumns(columns, prefix)`

DB2 requires one `ALTER TABLE ... ADD COLUMN` per column (unlike PostgreSQL which allows multiple in one statement, and MySQL which allows them separated by commas). DB2 V10+ allows multiple columns in one `ADD`:

```js
addColumns(columns, prefix) {
  if (columns.sql.length === 0) return;
  prefix = prefix || this.addColumnsPrefix;

  // DB2 V10+: ALTER TABLE t ADD COLUMN c1 ..., ADD COLUMN c2 ...
  // DB2 pre-V10: one ALTER TABLE per column
  columns.sql.forEach((col) => {
    this.pushQuery({
      sql: `ALTER TABLE ${this.tableName()} ${prefix}${col}`,
      bindings: columns.bindings,
    });
  });
}
```

### `alterColumns(columns, colBuilders)`

DB2 uses `ALTER TABLE ... ALTER COLUMN col SET DATA TYPE ...` for type changes and `ALTER COLUMN col SET DEFAULT ...` for default changes. The alter syntax differs significantly:

```js
alterColumns(columns, colBuilders) {
  columns.sql.forEach((sql, i) => {
    this.pushQuery({
      sql: `ALTER TABLE ${this.tableName()} ALTER COLUMN ${sql}`,
      bindings: columns.bindings,
    });
  });
}
```

### `dropColumn()`

```js
dropColumn() {
  const columns = require('../../../util/helpers').normalizeArr.apply(null, arguments);
  columns.forEach((column) => {
    this.pushQuery(`ALTER TABLE ${this.tableName()} DROP COLUMN ${this.formatter.wrap(column)}`);
  });
}
```

> **DB2 z/OS note:** After dropping a column, a `REORG TABLE` is required for the change to take physical effect. You may want to push a `CALL SYSPROC.ADMIN_REORG_TABLE(...)` or document this requirement.

### `renameColumn(from, to)`

DB2 for z/OS supports `ALTER TABLE t ALTER COLUMN old RENAME TO new` (V11+). Pre-V11 requires add + copy + drop.

```js
renameColumn(from, to) {
  this.pushQuery(
    `ALTER TABLE ${this.tableName()} RENAME COLUMN ${this.formatter.wrap(from)} TO ${this.formatter.wrap(to)}`
  );
}
```

### `primary(columns, constraintName)`

```js
primary(columns, constraintName) {
  constraintName = constraintName
    ? this.formatter.wrap(constraintName)
    : this.formatter.wrap(`${this.tableNameRaw}_PKEY`);

  if (!this.forCreate) {
    this.pushQuery(
      `ALTER TABLE ${this.tableName()} ADD CONSTRAINT ${constraintName} PRIMARY KEY (${this.formatter.columnize(columns)})`
    );
  } else {
    this.pushQuery(
      `CONSTRAINT ${constraintName} PRIMARY KEY (${this.formatter.columnize(columns)})`
    );
  }
}
```

### `unique(columns, indexName)`

DB2 for z/OS unique constraints are inline with the table or via `CREATE UNIQUE INDEX`. Note that unique *indexes* on nullable columns allow multiple NULLs (like Postgres).

```js
unique(columns, indexName) {
  indexName = indexName
    ? this.formatter.wrap(indexName)
    : this._indexCommand('UNIQUE', this.tableNameRaw, columns);

  if (!this.forCreate) {
    this.pushQuery(
      `CREATE UNIQUE INDEX ${indexName} ON ${this.tableName()} (${this.formatter.columnize(columns)})`
    );
  } else {
    this.pushQuery(
      `CONSTRAINT ${indexName} UNIQUE (${this.formatter.columnize(columns)})`
    );
  }
}
```

### `index(columns, indexName, options)`

```js
index(columns, indexName, options) {
  indexName = indexName
    ? this.formatter.wrap(indexName)
    : this._indexCommand('INDEX', this.tableNameRaw, columns);

  this.pushQuery(
    `CREATE INDEX ${indexName} ON ${this.tableName()} (${this.formatter.columnize(columns)})`
  );
}
```

### `dropIndex(columns, indexName)`

DB2 uses `DROP INDEX indexName` (no `ON table`).

```js
dropIndex(columns, indexName) {
  indexName = indexName
    ? this.formatter.wrap(indexName)
    : this._indexCommand('INDEX', this.tableNameRaw, columns);
  this.pushQuery(`DROP INDEX ${indexName}`);
}
```

### `dropUnique(column, indexName)` / `dropForeign(columns, indexName)` / `dropPrimary(constraintName)`

All use `ALTER TABLE ... DROP CONSTRAINT`:

```js
dropUnique(column, indexName) {
  indexName = indexName
    ? this.formatter.wrap(indexName)
    : this._indexCommand('UNIQUE', this.tableNameRaw, column);
  this.pushQuery(`DROP INDEX ${indexName}`);
}

dropForeign(columns, indexName) {
  indexName = indexName
    ? this.formatter.wrap(indexName)
    : this._indexCommand('FOREIGN', this.tableNameRaw, columns);
  this.pushQuery(`ALTER TABLE ${this.tableName()} DROP FOREIGN KEY ${indexName}`);
}

dropPrimary(constraintName) {
  constraintName = constraintName
    ? this.formatter.wrap(constraintName)
    : this.formatter.wrap(`${this.tableNameRaw}_PKEY`);
  this.pushQuery(`ALTER TABLE ${this.tableName()} DROP PRIMARY KEY`);
  // DB2 drops the PK by name or just DROP PRIMARY KEY (there can only be one)
}
```

### `_setNullableState(column, isNullable)`

DB2 uses `ALTER COLUMN col SET NOT NULL` / `ALTER COLUMN col DROP NOT NULL` but only from V11. Earlier versions need a different approach (recreating the column).

```js
_setNullableState(column, isNullable) {
  const colName = this.formatter.columnize(column);
  const action  = isNullable ? 'DROP NOT NULL' : 'SET NOT NULL';
  this.pushQuery(`ALTER TABLE ${this.tableName()} ALTER COLUMN ${colName} ${action}`);
}
```

---

## 6. ColumnCompiler (DDL – Column Level)

**Base class:** `lib/schema/columncompiler.js`  
**Analogous dialects:** MSSQL (`lib/dialects/mssql/schema/mssql-columncompiler.js`), Oracle (`lib/dialects/oracle/schema/oracle-columncompiler.js`)

`ColumnCompiler` translates Knex's abstract column types and modifiers into DB2-specific SQL fragments. Each column type method returns a DB2 type string.

### Constructor / modifiers list

```js
class ColumnCompiler_DB2ZOS extends ColumnCompiler {
  constructor(client, tableCompiler, columnBuilder) {
    super(client, tableCompiler, columnBuilder);
    // Order matters — these are applied left to right to build the column DDL
    this.modifiers = ['nullable', 'defaultTo', 'generated', 'comment'];
    this._addCheckModifiers(); // adds check*, checkPositive, checkNegative, etc.
  }
}
```

### Data type mappings

The following table maps every Knex abstract type to the recommended DB2 for z/OS type:

| Knex method | DB2 for z/OS type | Notes |
|---|---|---|
| `increments(name)` | `INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1)` | Identity column; primary key added separately |
| `bigincrements(name)` | `BIGINT NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1)` | |
| `integer(name)` | `INTEGER` | |
| `smallint(name)` | `SMALLINT` | |
| `tinyint(name)` | `SMALLINT` | DB2 has no TINYINT |
| `mediumint(name)` | `INTEGER` | DB2 has no MEDIUMINT |
| `biginteger(name)` | `BIGINT` | |
| `float(name, p, s)` | `FLOAT(p)` or `REAL` | |
| `double(name, p, s)` | `DOUBLE` | |
| `decimal(name, p, s)` | `DECIMAL(p, s)` | |
| `boolean(name)` | `SMALLINT` + CHECK (col IN (0,1)) | DB2 for z/OS has no BOOLEAN |
| `string(name, length)` | `VARCHAR(length)` | |
| `varchar(name, length)` | `VARCHAR(length)` | |
| `char(name, length)` | `CHAR(length)` | |
| `text(name)` | `CLOB(32000)` | Or `VARCHAR(32672)` for short text |
| `mediumtext(name)` | `CLOB(16M)` | |
| `longtext(name)` | `CLOB(2G)` | |
| `binary(name, length)` | `VARBINARY(length)` or `BLOB(length)` | |
| `date(name)` | `DATE` | |
| `datetime(name)` | `TIMESTAMP` | DB2 has no DATETIME; use TIMESTAMP |
| `time(name)` | `TIME` | |
| `timestamp(name)` | `TIMESTAMP` | |
| `json(name)` | `CLOB(2M)` | DB2 for z/OS V13+ has JSON type; earlier use CLOB |
| `jsonb(name)` | `CLOB(2M)` | Same as json |
| `uuid(name)` | `CHAR(36)` | DB2 has no native UUID type |
| `enu(name, values)` | `VARCHAR(max_val_len)` + CHECK (col IN (...)) | |
| `bit(name, length)` | `CHAR(n) FOR BIT DATA` | |
| `geometry / geography / point` | Not natively supported | Use `BLOB` or third-party spatial extension |

Implement the type methods:

```js
increments(options = { primaryKey: true }) {
  return (
    'INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1)' +
    (this.tableCompiler._canBeAddPrimaryKey(options) ? ' PRIMARY KEY' : '')
  );
}

bigincrements(options = { primaryKey: true }) {
  return (
    'BIGINT NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1)' +
    (this.tableCompiler._canBeAddPrimaryKey(options) ? ' PRIMARY KEY' : '')
  );
}

varchar(length) {
  return `VARCHAR(${toNumber(length, 255)})`;
}

decimal(precision, scale) {
  if (precision === null) return 'DECIMAL';
  return `DECIMAL(${toNumber(precision, 8)}, ${toNumber(scale, 2)})`;
}

floating(precision, scale) {
  return `FLOAT(${toNumber(precision, 8)})`;
}

double() {
  return 'DOUBLE';
}

timestamp({ useTz = false } = {}) {
  // DB2 for z/OS V10+: TIMESTAMP WITH TIME ZONE
  return useTz ? 'TIMESTAMP WITH TIME ZONE' : 'TIMESTAMP';
}

datetime({ useTz = false } = {}) {
  return this.timestamp({ useTz });
}

bool() {
  // No native BOOLEAN in DB2 for z/OS
  this.columnBuilder._modifiers.checkIn = [[0, 1]];
  return 'SMALLINT';
}

enu(allowed) {
  const maxLength = allowed.reduce((m, v) => Math.max(m, String(v).length), 1);
  this.columnBuilder._modifiers.checkIn = [allowed];
  return `VARCHAR(${maxLength})`;
}

uuid({ useBinaryUuid = false } = {}) {
  return useBinaryUuid ? 'BINARY(16)' : 'CHAR(36)';
}
```

Set the simple type aliases on the prototype:

```js
ColumnCompiler_DB2ZOS.prototype.integer   = 'INTEGER';
ColumnCompiler_DB2ZOS.prototype.tinyint   = 'SMALLINT';
ColumnCompiler_DB2ZOS.prototype.smallint  = 'SMALLINT';
ColumnCompiler_DB2ZOS.prototype.mediumint = 'INTEGER';
ColumnCompiler_DB2ZOS.prototype.biginteger = 'BIGINT';
ColumnCompiler_DB2ZOS.prototype.text      = 'CLOB(32000)';
ColumnCompiler_DB2ZOS.prototype.mediumtext = 'CLOB(16M)';
ColumnCompiler_DB2ZOS.prototype.longtext  = 'CLOB(2G)';
ColumnCompiler_DB2ZOS.prototype.json      = 'CLOB(2M)';
ColumnCompiler_DB2ZOS.prototype.jsonb     = 'CLOB(2M)';
ColumnCompiler_DB2ZOS.prototype.date      = 'DATE';
ColumnCompiler_DB2ZOS.prototype.time      = 'TIME';
ColumnCompiler_DB2ZOS.prototype.bit       = 'CHAR(1) FOR BIT DATA';
ColumnCompiler_DB2ZOS.prototype.binary    = 'BLOB';
ColumnCompiler_DB2ZOS.prototype.bool      = 'SMALLINT'; // overridden by method above
```

### Modifiers

#### `nullable(value)`

DB2 uses the standard `NOT NULL` / `NULL` syntax. The base implementation is correct:

```js
// Inherited from base — returns 'not null' or 'null'
```

#### `defaultTo(value)`

The base implementation calls `formatDefault()` which handles booleans, strings, `null`, and `knex.raw()`. DB2 accepts `DEFAULT value` as standard syntax. Override only if you need to handle DB2-specific defaults like `DEFAULT CURRENT_TIMESTAMP`.

```js
defaultTo(value) {
  // knex.raw('CURRENT TIMESTAMP') works without override.
  // The base formatDefault() is sufficient in most cases.
  return super.defaultTo(value);
}
```

#### `comment(comment)`

DB2 for z/OS uses a separate `COMMENT ON COLUMN table.col IS '...'` statement, emitted as an additional query via `pushAdditional`.

```js
comment(comment) {
  const columnName = this.args[0] || this.defaults('columnName');
  this.pushAdditional(function () {
    this.pushQuery(
      `COMMENT ON COLUMN ${this.tableCompiler.tableName()}.` +
      this.formatter.wrap(columnName) +
      ` IS '${(comment || '').replace(/'/g, "''")}'`
    );
  }, comment);
  return '';
}
```

#### `generated` (GENERATED ALWAYS/BY DEFAULT)

DB2 supports generated columns. If you expose a custom `.generatedAs(expression)` builder method, handle it here:

```js
generated(expr) {
  return `GENERATED ALWAYS AS (${expr})`;
}
```

---

## 7. Transaction

**Base class:** `lib/execution/transaction.js`

For most DB2 drivers (ibm_db, ODBC), the base `Transaction` class works without modification because it uses `BEGIN`/`COMMIT`/`ROLLBACK` SQL statements which DB2 supports. However, the base class actually does NOT issue `BEGIN` — instead it sets `autocommit = false` on the connection.

Check whether your driver requires explicit `BEGIN WORK` / `COMMIT WORK` / `ROLLBACK WORK`:

```js
const Transaction = require('../../execution/transaction');

class Transaction_DB2ZOS extends Transaction {
  async begin(connection) {
    // DB2 requires autocommit to be off; ibm_db does this automatically.
    // Some ODBC drivers require:
    // await connection.setIsolationLevel(...);
    // For ibm_db, autocommit is handled by:
    connection.beginTransaction((err) => { /* ... */ });
  }

  async commit(connection, value) {
    return new Promise((resolve, reject) => {
      connection.commitTransaction((err) => {
        if (err) return reject(err);
        resolve(value);
      });
    });
  }

  async rollback(connection, error) {
    return new Promise((resolve, reject) => {
      connection.rollbackTransaction((err) => {
        if (err) return reject(err);
        resolve(error);
      });
    });
  }
}
```

### Isolation levels

DB2 for z/OS isolation levels map to standard SQL terms but use different names internally:

| Standard SQL | DB2 for z/OS name | DB2 BIND option |
|---|---|---|
| Read Uncommitted | Uncommitted Read (UR) | `ISOLATION(UR)` |
| Read Committed | Cursor Stability (CS) | `ISOLATION(CS)` |
| Repeatable Read | Read Stability (RS) | `ISOLATION(RS)` |
| Serializable | Repeatable Read (RR) | `ISOLATION(RR)` |

Knex uses isolation levels as SQL `SET TRANSACTION ISOLATION LEVEL ...` statements. In DB2 for z/OS, isolation is typically controlled at the **package/plan bind level** or at the statement level using `WITH UR/CS/RS/RR` appended to each query (see `forUpdate()` / `forShare()` above).

You can also emit `SET CURRENT ISOLATION = UR` etc. as a transaction-level override in DB2 V9+:

```js
// In Transaction_DB2ZOS:
savepoint(transactionSavepoint) {
  return this.query(this.connection, `SAVEPOINT ${transactionSavepoint} ON ROLLBACK RETAIN CURSORS`);
}

release(transactionSavepoint) {
  return this.query(this.connection, `RELEASE TO SAVEPOINT ${transactionSavepoint}`);
}

rollbackTo(transactionSavepoint) {
  return this.query(this.connection, `ROLLBACK TO SAVEPOINT ${transactionSavepoint}`);
}
```

> **Note:** DB2 for z/OS supports savepoints from V8. Ensure you use the correct syntax `SAVEPOINT name ON ROLLBACK RETAIN CURSORS`.

---

## 8. Formatter / Identifier Quoting

**Base class:** `lib/formatter.js`

The Formatter is used internally by compilers for wrapping identifiers and handling schema-prefixed names. You rarely need to subclass it.

### Identifier quoting rules for DB2 for z/OS

- Object names that are **unquoted** are folded to **UPPERCASE**.
- Object names that are **double-quoted** preserve case exactly.
- Knex's `wrapIdentifierImpl` wraps identifiers with `"..."`. This means `knex.schema.createTable('Users', ...)` creates a table named `"Users"` (mixed case). If you want the DB2 default uppercase behavior, either:
  1. Override `wrapIdentifierImpl` to uppercase before wrapping, or
  2. Document that users should always use uppercase table/column names.

```js
// Option 1 — uppercase all identifiers (DB2 default behavior):
wrapIdentifierImpl(value) {
  if (value === '*') return '*';
  return `"${value.toUpperCase().replace(/"/g, '""')}"`;
}

// Option 2 — preserve case (ANSI standard, user's responsibility):
wrapIdentifierImpl(value) {
  if (value === '*') return '*';
  return `"${value.replace(/"/g, '""')}"`;
}
```

### Schema / three-part names

DB2 for z/OS does not support three-part (database.schema.table) names the same way SQL Server does. The equivalent is `schema.table` (two-part). The `withSchema()` call on `SchemaBuilder` / `QueryBuilder` provides the schema prefix, and `formatter.wrap()` handles schema-qualified names correctly in the base implementation.

---

## 9. Wire Protocol / Runner Hooks

Beyond the Client methods described above, the Runner (`lib/execution/runner.js`) calls these hooks if you define them:

| Hook | When called | Notes |
|---|---|---|
| `_query(conn, queryObj)` | Every SQL statement | **Required.** Set `queryObj.response`. |
| `_stream(conn, queryObj, stream)` | Streaming queries | Implement for `.stream()` support |
| `processResponse(queryObj, runner)` | After `_query` returns | **Required.** Map raw rows → Knex result |
| `processPassedConnection(conn)` | When user passes a raw connection | Optional cleanup |

For **ibm_db**:

```js
_query(connection, queryObject) {
  return new Promise((resolve, reject) => {
    const { sql, bindings } = queryObject;
    connection.query(sql, bindings || [], (err, result) => {
      if (err) return reject(err);
      queryObject.response = Array.isArray(result) ? result : [];
      resolve(queryObject);
    });
  });
}
```

For **ODBC**:

```js
_query(connection, queryObject) {
  return new Promise((resolve, reject) => {
    connection.query(queryObject.sql, queryObject.bindings || [], (err, result) => {
      if (err) return reject(err);
      queryObject.response = result;
      resolve(queryObject);
    });
  });
}
```

---

## 10. File Layout Recommendation

```
lib/dialects/db2-zos/
├── index.js                            ← Client_DB2ZOS
├── transaction.js                      ← Transaction_DB2ZOS
├── db2zos-formatter.js                 ← (optional) Formatter subclass
├── query/
│   └── db2zos-querycompiler.js         ← QueryCompiler_DB2ZOS
└── schema/
    ├── db2zos-compiler.js              ← SchemaCompiler_DB2ZOS
    ├── db2zos-tablecompiler.js         ← TableCompiler_DB2ZOS
    ├── db2zos-columncompiler.js        ← ColumnCompiler_DB2ZOS
    └── db2zos-columnbuilder.js         ← (optional) ColumnBuilder_DB2ZOS
```

---

## 11. Registration / Usage

### Register as a named client

```js
// In lib/dialects/index.js (add to the dialect map):
module.exports = {
  // ... existing dialects ...
  'db2-zos': require('./db2-zos'),
};
```

### Use it

```js
const knex = require('knex');

const db = knex({
  client: 'db2-zos',   // OR pass the class directly: client: require('./lib/dialects/db2-zos')
  connection: {
    host:           'DB2HOST.example.com',
    port:           446,        // SSL/TLS port; standard is 50000
    database:       'DSNDB06',  // DB2 subsystem / location name
    user:           'MYUSER',
    password:       'MYPASS',
    currentSchema:  'MYSCHEMA', // SET CURRENT SCHEMA equivalent
  },
  pool: { min: 2, max: 10 },
});
```

### Connection string for ibm_db

The `ibm_db` driver uses a DSN-style connection string:

```js
acquireRawConnection() {
  const c = this.connectionSettings;
  const connStr =
    `DATABASE=${c.database};` +
    `HOSTNAME=${c.host};` +
    `PORT=${c.port || 446};` +
    `PROTOCOL=TCPIP;` +
    `UID=${c.user};` +
    `PWD=${c.password};` +
    (c.currentSchema ? `CURRENTSCHEMA=${c.currentSchema};` : '');

  return new Promise((resolve, reject) => {
    this.driver.open(connStr, (err, conn) => {
      if (err) return reject(err);
      conn.__knexUid = require('lodash/uniqueId')('__knexUid');
      resolve(conn);
    });
  });
}
```

---

## 12. Quick-Reference: DB2 for z/OS SQL Specifics

This section is a concise cheat sheet of the most important DB2 for z/OS SQL differences versus standard SQL or other databases, grouped by the compiler layer they affect.

### DML (QueryCompiler)

| Feature | DB2 for z/OS syntax | Notes |
|---|---|---|
| Parameter placeholder | `?` | Same as MySQL; ibm_db and odbc both support `?` |
| Identifier delimiter | `"name"` (ANSI) | Unquoted names are uppercased |
| Row limiting (V12+) | `FETCH FIRST n ROWS ONLY` | Append after ORDER BY |
| Row limiting + offset (V12+) | `OFFSET n ROWS FETCH NEXT m ROWS ONLY` | |
| Row limiting (pre-V12) | `ROW_NUMBER() OVER()` subquery wrap | |
| FOR UPDATE | `FOR UPDATE WITH RS` | RS = Read Stability |
| FOR READ | `FOR FETCH ONLY` | |
| Truncate | `TRUNCATE TABLE t IMMEDIATE` | V9+; older: `DELETE FROM t` |
| CTE | `WITH cte AS (...)` | No `RECURSIVE` keyword needed |
| INSERT default | `INSERT INTO t (col) VALUES (DEFAULT)` | One column required for pure-default |
| RETURNING/OUTPUT | Not supported | Use `VALUES IDENTITY_VAL_LOCAL()` after INSERT |
| MERGE (upsert) | `MERGE INTO target USING source ON (...) WHEN MATCHED THEN ... WHEN NOT MATCHED THEN ...` | DB2 V8+ |
| JSON functions (V13+) | `JSON_VALUE`, `JSON_OBJECT`, `JSON_ARRAY`, `JSON_EXISTS` | |
| JSON pre-V13 | Store in CLOB; parse in application | |
| `DISTINCT ON` | Not supported | Use subquery |
| `ILIKE` | Not supported | Use `UPPER(col) LIKE UPPER(val)` |
| `||` string concat | Supported | |
| `CURRENT_TIMESTAMP` | `CURRENT TIMESTAMP` (no parentheses) | |
| `CURRENT_DATE` | `CURRENT DATE` | |
| `CURRENT_TIME` | `CURRENT TIME` | |

### DDL (Schema / Table / Column Compilers)

| Feature | DB2 for z/OS syntax | Notes |
|---|---|---|
| Create table | `CREATE TABLE schema.name (...)` | No `IF NOT EXISTS` |
| Drop table | `DROP TABLE name` | No `IF EXISTS`; use COMPOUND SQL or catch -204 |
| Drop column | `ALTER TABLE t DROP COLUMN col` | Requires REORG after |
| Rename column (V11+) | `ALTER TABLE t RENAME COLUMN old TO new` | |
| Rename column (pre-V11) | Add new + copy data + drop old | |
| Add column | `ALTER TABLE t ADD COLUMN col type` | One per statement pre-V10; multiple allowed V10+ |
| Alter column type | `ALTER TABLE t ALTER COLUMN col SET DATA TYPE type` | |
| Set/drop NOT NULL | `ALTER TABLE t ALTER COLUMN col SET NOT NULL` / `DROP NOT NULL` | V11+ |
| Auto-increment | `GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1)` | Per column |
| Boolean | `SMALLINT` with `CHECK (col IN (0,1))` | No native BOOLEAN |
| UUID | `CHAR(36)` | No native UUID type |
| JSON column | `CLOB(2M)` (pre-V13) / `JSON` (V13+) | |
| Index | `CREATE [UNIQUE] INDEX name ON schema.table (cols)` | |
| Drop index | `DROP INDEX name` | No `ON table` clause |
| Primary key | `CONSTRAINT name PRIMARY KEY (cols)` | |
| Foreign key | `CONSTRAINT name FOREIGN KEY (col) REFERENCES table(col) ON DELETE ... ON UPDATE ...` | |
| Table comment | `COMMENT ON TABLE t IS '...'` | |
| Column comment | `COMMENT ON COLUMN t.col IS '...'` | |
| Rename table | `RENAME TABLE old TO new` | |
| Create schema | `CREATE SCHEMA name` | |
| Drop schema | `DROP SCHEMA name RESTRICT` | RESTRICT = fail if objects exist; RESTRICT is the only option in z/OS |
| Savepoint | `SAVEPOINT name ON ROLLBACK RETAIN CURSORS` | |
| Release savepoint | `RELEASE TO SAVEPOINT name` | |
| Rollback to savepoint | `ROLLBACK TO SAVEPOINT name` | |
| Check if table exists | `SELECT 1 FROM SYSIBM.SYSTABLES WHERE NAME='T' AND CREATOR='S' AND TYPE='T'` | |
| Check if column exists | `SELECT 1 FROM SYSIBM.SYSCOLUMNS WHERE TBNAME='T' AND NAME='C'` | |
| Column info | `SELECT NAME, COLTYPE, LENGTH, SCALE, NULLS, DEFAULT FROM SYSIBM.SYSCOLUMNS WHERE TBNAME=?` | |

### Catalog Tables

| Knex need | DB2 for z/OS catalog table | Key columns |
|---|---|---|
| Table existence | `SYSIBM.SYSTABLES` | `NAME`, `CREATOR`, `TYPE` ('T' = table) |
| Column info | `SYSIBM.SYSCOLUMNS` | `TBNAME`, `TBCREATOR`, `NAME`, `COLTYPE`, `LENGTH`, `SCALE`, `NULLS`, `DEFAULT` |
| Index info | `SYSIBM.SYSINDEXES` | `NAME`, `TBNAME`, `TBCREATOR`, `UNIQUERULE` |
| Constraint info | `SYSIBM.SYSTABCONST` | `CONSTNAME`, `TBNAME`, `TBCREATOR`, `TYPE` |
| Schema existence | `SYSIBM.SYSSCHEMAS` | `SCHEMANAME` |
| View existence | `SYSIBM.SYSVIEWS` | `NAME`, `CREATOR` |

---

## Additional Notes and Caveats

### REORG requirement

Many DDL changes on DB2 for z/OS (dropping columns, altering column types) leave the table in a "REORG pending" state. Subsequent DML on the table may fail with `SQL0668N`. You should document that users must run `CALL SYSPROC.ADMIN_REORG_TABLE(tableName)` or `REORG TABLESPACE` after certain schema alterations. You may optionally push a REORG call after relevant DDL in the TableCompiler.

### Sequence vs. IDENTITY

Older DB2 for z/OS applications use explicit `CREATE SEQUENCE` objects. IDENTITY columns (the `GENERATED ALWAYS AS IDENTITY` syntax) were introduced in V8 and are the recommended approach for new code. The `increments()` column type should use IDENTITY.

### LOB columns

`TEXT`, `BLOB`, `CLOB` columns in DB2 for z/OS require an **auxiliary table** and **auxiliary index** to be created alongside the base table. This is an automatic process in modern DB2 versions but requires attention:
- LOB columns must specify a LOB table space in older configurations.
- `CLOB(32000)` or smaller actually fits in the base row without an auxiliary table.
- `CLOB(32001)` or larger requires an auxiliary table.

### Package / Plan binding

DB2 for z/OS applications run through a **package** that is bound to the subsystem. When using a Node.js driver via DRDA (distributed), this is handled automatically, but static SQL plans require explicit `BIND PACKAGE`. Dynamic SQL (which is what Knex generates) does not require pre-binding.

### Special registers

These DB2 for z/OS special registers are useful in Knex `raw()` calls:

| Register | Description |
|---|---|
| `CURRENT DATE` | Current date |
| `CURRENT TIME` | Current time |
| `CURRENT TIMESTAMP` | Current timestamp |
| `CURRENT SCHEMA` | Current default schema |
| `CURRENT USER` | Current authorization ID |
| `CURRENT SERVER` | Connected location name |
| `IDENTITY_VAL_LOCAL()` | Last generated identity value |
| `CURRENT ISOLATION` | Current isolation level |

Example:

```js
// Use CURRENT TIMESTAMP as a default
knex.schema.createTable('events', (t) => {
  t.timestamp('created_at').defaultTo(knex.raw('CURRENT TIMESTAMP'));
});
```

### Naming length limits

DB2 for z/OS imposes strict identifier length limits:

| Object type | Max name length |
|---|---|
| Table, view, alias | 128 characters (V12+), 18 characters (V11 and earlier) |
| Column | 30 characters (V12+), 30 characters (V11) |
| Index | 128 characters (V12+), 18 characters (V11 and earlier) |
| Constraint | 128 characters |
| Schema / collection | 128 characters |

> **Important:** The auto-generated index names from `_indexCommand()` use the pattern `tableName_col1_col2_type`. This can easily exceed 18 characters for V11 and earlier. Override `_indexCommand()` in your TableCompiler to truncate or hash long names.

```js
_indexCommand(type, tableName, columns) {
  const name = super._indexCommand(type, tableName, columns);
  // DB2 V11 and earlier: 18 char limit
  const maxLen = this.client.version && parseFloat(this.client.version) < 12 ? 18 : 128;
  const inner  = name.slice(1, -1); // strip quotes
  if (inner.length <= maxLen) return name;
  // truncate with a short hash suffix to avoid collisions
  const crypto = require('crypto');
  const hash   = crypto.createHash('md5').update(inner).digest('hex').slice(0, 4);
  return this.formatter.wrap(inner.slice(0, maxLen - 5) + '_' + hash);
}
```
