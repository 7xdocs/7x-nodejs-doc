# SQLite

<!--introduced_in=v22.5.0-->

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active development.

<!-- source_link=lib/sqlite.js -->

`node:sqlite` 模块用于操作 SQLite 数据库。
可以通过以下方式访问它：

```mjs
import sqlite from 'node:sqlite';
```

```cjs
const sqlite = require('node:sqlite');
```

此模块仅在 `node:` 方案下可用。

以下示例展示了使用 `node:sqlite` 模块打开一个内存数据库、向数据库写入数据，然后读取数据的基本用法。

```mjs
import { DatabaseSync } from 'node:sqlite';
const database = new DatabaseSync(':memory:');

// Execute SQL statements from strings.
database.exec(`
  CREATE TABLE data(
    key INTEGER PRIMARY KEY,
    value TEXT
  ) STRICT
`);
// Create a prepared statement to insert data into the database.
const insert = database.prepare('INSERT INTO data (key, value) VALUES (?, ?)');
// Execute the prepared statement with bound values.
insert.run(1, 'hello');
insert.run(2, 'world');
// Create a prepared statement to read data from the database.
const query = database.prepare('SELECT * FROM data ORDER BY key');
// Execute the prepared statement and log the result set.
console.log(query.all());
// Prints: [ { key: 1, value: 'hello' }, { key: 2, value: 'world' } ]
```

```cjs
'use strict';
const { DatabaseSync } = require('node:sqlite');
const database = new DatabaseSync(':memory:');

// Execute SQL statements from strings.
database.exec(`
  CREATE TABLE data(
    key INTEGER PRIMARY KEY,
    value TEXT
  ) STRICT
`);
// Create a prepared statement to insert data into the database.
const insert = database.prepare('INSERT INTO data (key, value) VALUES (?, ?)');
// Execute the prepared statement with bound values.
insert.run(1, 'hello');
insert.run(2, 'world');
// Create a prepared statement to read data from the database.
const query = database.prepare('SELECT * FROM data ORDER BY key');
// Execute the prepared statement and log the result set.
console.log(query.all());
// Prints: [ { key: 1, value: 'hello' }, { key: 2, value: 'world' } ]
```

## 类：`DatabaseSync`

<!-- YAML
added: v22.5.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57752
    description: Add `timeout` option.
  - version:
    - v23.10.0
    - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/56991
    description: The `path` argument now supports Buffer and URL objects.
-->

此类表示与 SQLite 数据库的单个[连接][]。此类暴露的所有 API 都是同步执行的。

### `new DatabaseSync(path[, options])`

<!-- YAML
added: v22.5.0
changes:
  - version: v24.4.0
    pr-url: https://github.com/nodejs/node/pull/58697
    description: Add new SQLite database options.
-->

* `path` {string | Buffer | URL} 数据库的路径。SQLite 数据库可以存储在文件中或完全[内存中][]。要使用文件支持的数据库，路径应为文件路径。要使用内存数据库，路径应为特殊名称 `':memory:'`。
* `options` {Object} 数据库连接的配置选项。支持以下选项：
  * `open` {boolean} 如果为 `true`，数据库由构造函数打开。当此值为 `false` 时，必须通过 `open()` 方法打开数据库。**默认值:** `true`。
  * `readOnly` {boolean} 如果为 `true`，数据库以只读模式打开。如果数据库不存在，打开操作将失败。**默认值:** `false`。
  * `enableForeignKeyConstraints` {boolean} 如果为 `true`，则启用外键约束。推荐启用，但为了兼容旧版数据库模式可以禁用它。外键约束的强制执行可以在打开数据库后使用 [`PRAGMA foreign_keys`][] 来启用和禁用。**默认值:** `true`。
  * `enableDoubleQuotedStringLiterals` {boolean} 如果为 `true`，SQLite 将接受[双引号字符串字面量][]。不推荐启用，但为了兼容旧版数据库模式可以启用。**默认值:** `false`。
  * `allowExtension` {boolean} 如果为 `true`，则启用 `loadExtension` SQL 函数和 `loadExtension()` 方法。之后可以调用 `enableLoadExtension(false)` 来禁用此功能。**默认值:** `false`。
  * `timeout` {number} [繁忙超时][]时间，以毫秒为单位。这是 SQLite 在返回错误之前等待数据库锁释放的最长时间。**默认值:** `0`。
  * `readBigInts` {boolean} 如果为 `true`，整数字段将作为 JavaScript `BigInt` 值读取。如果为 `false`，整数字段将作为 JavaScript 数字读取。**默认值:** `false`。
  * `returnArrays` {boolean} 如果为 `true`，查询结果将以数组形式返回，而不是对象。**默认值:** `false`。
  * `allowBareNamedParameters` {boolean} 如果为 `true`，允许绑定不带前缀字符的命名参数（例如，使用 `foo` 而不是 `:foo`）。**默认值:** `true`。
  * `allowUnknownNamedParameters` {boolean} 如果为 `true`，绑定命名参数时忽略未知参数。如果为 `false`，对于未知命名参数会抛出异常。**默认值:** `false`。

构造一个新的 `DatabaseSync` 实例。

### `database.aggregate(name, options)`

<!-- YAML
added: v24.0.0
-->

向 SQLite 数据库注册一个新的聚合函数。此方法是 [`sqlite3_create_window_function()`][] 的封装。

* `name` {string} 要创建的 SQLite 函数的名称。
* `options` {Object} 函数配置设置。
  * `deterministic` {boolean} 如果为 `true`，则在创建的函数上设置 [`SQLITE_DETERMINISTIC`][] 标志。**默认值:** `false`。
  * `directOnly` {boolean} 如果为 `true`，则在创建的函数上设置 [`SQLITE_DIRECTONLY`][] 标志。**默认值:** `false`。
  * `useBigIntArguments` {boolean} 如果为 `true`，传递给 `options.step` 和 `options.inverse` 的整数参数将转换为 `BigInt`。如果为 `false`，整数参数将作为 JavaScript 数字传递。**默认值:** `false`。
  * `varargs` {boolean} 如果为 `true`，`options.step` 和 `options.inverse` 可以用任意数量的参数调用（在零到 [`SQLITE_MAX_FUNCTION_ARG`][] 之间）。如果为 `false`，`inverse` 和 `step` 必须用恰好 `length` 个参数调用。**默认值:** `false`。
  * `start` {number | string | null | Array | Object | Function} 聚合函数的初始值。聚合函数初始化时使用此值。当传递 {Function} 时，初始值将是其返回值。
  * `step` {Function} 为聚合中的每一行调用的函数。该函数接收当前状态和行值。此函数的返回值应为新状态。
  * `result` {Function} 调用以获取聚合结果的函数。该函数接收最终状态并应返回聚合的结果。
  * `inverse` {Function} 当提供此函数时，`aggregate` 方法将作为窗口函数工作。该函数接收当前状态和移除的行值。此函数的返回值应为新状态。

当用作窗口函数时，`result` 函数将被多次调用。

```cjs
const { DatabaseSync } = require('node:sqlite');

const db = new DatabaseSync(':memory:');
db.exec(`
  CREATE TABLE t3(x, y);
  INSERT INTO t3 VALUES ('a', 4),
                        ('b', 5),
                        ('c', 3),
                        ('d', 8),
                        ('e', 1);
`);

db.aggregate('sumint', {
  start: 0,
  step: (acc, value) => acc + value,
});

db.prepare('SELECT sumint(y) as total FROM t3').get(); // { total: 21 }
```

```mjs
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:');
db.exec(`
  CREATE TABLE t3(x, y);
  INSERT INTO t3 VALUES ('a', 4),
                        ('b', 5),
                        ('c', 3),
                        ('d', 8),
                        ('e', 1);
`);

db.aggregate('sumint', {
  start: 0,
  step: (acc, value) => acc + value,
});

db.prepare('SELECT sumint(y) as total FROM t3').get(); // { total: 21 }
```

### `database.close()`

<!-- YAML
added: v22.5.0
-->

关闭数据库连接。如果数据库未打开，则抛出异常。此方法是 [`sqlite3_close_v2()`][] 的封装。

### `database.loadExtension(path)`

<!-- YAML
added:
  - v23.5.0
  - v22.13.0
-->

* `path` {string} 要加载的共享库的路径。

将共享库加载到数据库连接中。此方法是 [`sqlite3_load_extension()`][] 的封装。需要在构造 `DatabaseSync` 实例时启用 `allowExtension` 选项。

### `database.enableLoadExtension(allow)`

<!-- YAML
added:
  - v23.5.0
  - v22.13.0
-->

* `allow` {boolean} 是否允许加载扩展。

启用或禁用 `loadExtension` SQL 函数和 `loadExtension()` 方法。当构造时 `allowExtension` 为 `false` 时，出于安全原因，您无法启用加载扩展。

### `database.location([dbName])`

<!-- YAML
added: v24.0.0
-->

* `dbName` {string} 数据库的名称。可以是 `'main'`（默认的主数据库）或任何其他使用 [`ATTACH DATABASE`][] 添加的数据库。**默认值:** `'main'`。
* 返回值: {string | null} 数据库文件的位置。当使用内存数据库时，此方法返回 null。

此方法是 [`sqlite3_db_filename()`][] 的封装。

### `database.exec(sql)`

<!-- YAML
added: v22.5.0
-->

* `sql` {string} 要执行的 SQL 字符串。

此方法允许执行一个或多个 SQL 语句而不返回任何结果。当执行从文件中读取的 SQL 语句时，此方法很有用。此方法是 [`sqlite3_exec()`][] 的封装。

### `database.function(name[, options], function)`

<!-- YAML
added:
  - v23.5.0
  - v22.13.0
-->

* `name` {string} 要创建的 SQLite 函数的名称。
* `options` {Object} 函数的可选配置设置。支持以下属性：
  * `deterministic` {boolean} 如果为 `true`，则在创建的函数上设置 [`SQLITE_DETERMINISTIC`][] 标志。**默认值:** `false`。
  * `directOnly` {boolean} 如果为 `true`，则在创建的函数上设置 [`SQLITE_DIRECTONLY`][] 标志。**默认值:** `false`。
  * `useBigIntArguments` {boolean} 如果为 `true`，传递给 `function` 的整数参数将转换为 `BigInt`。如果为 `false`，整数参数将作为 JavaScript 数字传递。**默认值:** `false`。
  * `varargs` {boolean} 如果为 `true`，`function` 可以用任意数量的参数调用（在零到 [`SQLITE_MAX_FUNCTION_ARG`][] 之间）。如果为 `false`，`function` 必须用恰好 `function.length` 个参数调用。**默认值:** `false`。
* `function` {Function} 调用 SQLite 函数时要调用的 JavaScript 函数。此函数的返回值应为有效的 SQLite 数据类型：请参阅 [JavaScript 和 SQLite 之间的类型转换][]。如果返回值为 `undefined`，则结果默认为 `NULL`。

此方法用于创建 SQLite 用户定义函数。此方法是 [`sqlite3_create_function_v2()`][] 的封装。

### `database.setAuthorizer(callback)`

<!-- YAML
added: v24.10.0
-->

* `callback` {Function|null} 要设置的授权器函数，或 `null` 以清除当前授权器。

设置一个授权器回调，当 SQLite 尝试通过预编译语句访问数据或修改数据库模式时，将调用此回调。这可用于实现安全策略、审计访问或限制某些操作。此方法是 [`sqlite3_set_authorizer()`][] 的封装。

调用时，回调接收五个参数：

* `actionCode` {number} 正在执行的操作类型（例如，`SQLITE_INSERT`、`SQLITE_UPDATE`、`SQLITE_SELECT`）。
* `arg1` {string|null} 第一个参数（依赖于上下文，通常是表名）。
* `arg2` {string|null} 第二个参数（依赖于上下文，通常是列名）。
* `dbName` {string|null} 数据库的名称。
* `triggerOrView` {string|null} 导致访问的触发器或视图的名称。

回调必须返回以下常量之一：

* `SQLITE_OK` - 允许操作。
* `SQLITE_DENY` - 拒绝操作（导致错误）。
* `SQLITE_IGNORE` - 忽略操作（静默跳过）。

```cjs
const { DatabaseSync, constants } = require('node:sqlite');
const db = new DatabaseSync(':memory:');

// Set up an authorizer that denies all table creation
db.setAuthorizer((actionCode) => {
  if (actionCode === constants.SQLITE_CREATE_TABLE) {
    return constants.SQLITE_DENY;
  }
  return constants.SQLITE_OK;
});

// This will work
db.prepare('SELECT 1').get();

// This will throw an error due to authorization denial
try {
  db.exec('CREATE TABLE blocked (id INTEGER)');
} catch (err) {
  console.log('Operation blocked:', err.message);
}
```

```mjs
import { DatabaseSync, constants } from 'node:sqlite';
const db = new DatabaseSync(':memory:');

// Set up an authorizer that denies all table creation
db.setAuthorizer((actionCode) => {
  if (actionCode === constants.SQLITE_CREATE_TABLE) {
    return constants.SQLITE_DENY;
  }
  return constants.SQLITE_OK;
});

// This will work
db.prepare('SELECT 1').get();

// This will throw an error due to authorization denial
try {
  db.exec('CREATE TABLE blocked (id INTEGER)');
} catch (err) {
  console.log('Operation blocked:', err.message);
}
```

### `database.isOpen`

<!-- YAML
added:
  - v23.11.0
  - v22.15.0
-->

* 类型: {boolean} 数据库当前是否打开。

### `database.isTransaction`

<!-- YAML
added: v24.0.0
-->

* 类型: {boolean} 数据库当前是否在事务中。此方法是 [`sqlite3_get_autocommit()`][] 的封装。

### `database.open()`

<!-- YAML
added: v22.5.0
-->

打开 `DatabaseSync` 构造函数的 `path` 参数中指定的数据库。仅当数据库未通过构造函数打开时才应使用此方法。如果数据库已经打开，则抛出异常。

### `database.prepare(sql)`

<!-- YAML
added: v22.5.0
-->

* `sql` {string} 要编译为预编译语句的 SQL 字符串。
* 返回值: {StatementSync} 预编译语句。

将 SQL 语句编译为[预编译语句][]。此方法是 [`sqlite3_prepare_v2()`][] 的封装。

### `database.createSQLTagStore([maxSize])`

<!-- YAML
added: v24.9.0
-->

* `maxSize` {integer} 要缓存的预编译语句的最大数量。**默认值:** `1000`。
* 返回值: {SQLTagStore} 一个新的 SQL 标签存储，用于缓存预编译语句。

创建一个新的 `SQLTagStore`，它是一个用于存储预编译语句的 LRU（最近最少使用）缓存。这允许通过使用唯一标识符标记预编译语句来有效地重用它们。

当执行带标签的 SQL 字面量时，`SQLTagStore` 会检查该特定 SQL 字符串的预编译语句是否已存在于缓存中。如果存在，则使用缓存的语句。如果不存在，则创建一个新的预编译语句，执行它，然后将其存储在缓存中以备将来使用。此机制有助于避免重复解析和准备相同 SQL 语句的开销。

```mjs
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:');
const sql = db.createSQLTagStore();

db.exec('CREATE TABLE users (id INT, name TEXT)');

// Using the 'run' method to insert data.
// The tagged literal is used to identify the prepared statement.
sql.run`INSERT INTO users VALUES (1, 'Alice')`;
sql.run`INSERT INTO users VALUES (2, 'Bob')`;

// Using the 'get' method to retrieve a single row.
const id = 1;
const user = sql.get`SELECT * FROM users WHERE id = ${id}`;
console.log(user); // { id: 1, name: 'Alice' }

// Using the 'all' method to retrieve all rows.
const allUsers = sql.all`SELECT * FROM users ORDER BY id`;
console.log(allUsers);
// [
//   { id: 1, name: 'Alice' },
//   { id: 2, name: 'Bob' }
// ]
```

### `database.createSession([options])`

<!-- YAML
added:
  - v23.3.0
  - v22.12.0
-->

* `options` {Object} 会话的配置选项。
  * `table` {string} 要跟踪更改的特定表。默认情况下，跟踪所有表的更改。
  * `db` {string} 要跟踪的数据库的名称。当使用 [`ATTACH DATABASE`][] 添加了多个数据库时，这很有用。**默认值**: `'main'`。
* 返回值: {Session} 会话句柄。

创建会话并将其附加到数据库。此方法是 [`sqlite3session_create()`][] 和 [`sqlite3session_attach()`][] 的封装。

### `database.applyChangeset(changeset[, options])`

<!-- YAML
added:
  - v23.3.0
  - v22.12.0
-->

* `changeset` {Uint8Array} 二进制变更集或补丁集。
* `options` {Object} 应用更改的配置选项。
  * `filter` {Function} 跳过那些当目标表名提供给此函数时返回真值的更改。默认情况下，尝试所有更改。
  * `onConflict` {Function} 一个决定如何处理冲突的函数。该函数接收一个参数，可以是以下值之一：

    * `SQLITE_CHANGESET_DATA`: `DELETE` 或 `UPDATE` 更改不包含预期的“之前”值。
    * `SQLITE_CHANGESET_NOTFOUND`: 与 `DELETE` 或 `UPDATE` 更改的主键匹配的行不存在。
    * `SQLITE_CHANGESET_CONFLICT`: `INSERT` 更改导致重复的主键值。
    * `SQLITE_CHANGESET_FOREIGN_KEY`: 应用更改将导致外键冲突。
    * `SQLITE_CHANGESET_CONSTRAINT`: 应用更改导致 `UNIQUE`、`CHECK` 或 `NOT NULL` 约束冲突。

    该函数应返回以下值之一：

    * `SQLITE_CHANGESET_OMIT`: 忽略冲突的更改。
    * `SQLITE_CHANGESET_REPLACE`: 用冲突的更改替换现有值（仅对 `SQLITE_CHANGESET_DATA` 或 `SQLITE_CHANGESET_CONFLICT` 冲突有效）。
    * `SQLITE_CHANGESET_ABORT`: 在冲突时中止并回滚数据库。

    当冲突处理程序中抛出错误或处理程序返回任何其他值时，应用变更集会中止并且数据库会回滚。

    **默认值**: 返回 `SQLITE_CHANGESET_ABORT` 的函数。
* 返回值: {boolean} 变更集是否成功应用而未中止。

如果数据库未打开，则抛出异常。此方法是 [`sqlite3changeset_apply()`][] 的封装。

```js
const sourceDb = new DatabaseSync(':memory:');
const targetDb = new DatabaseSync(':memory:');

sourceDb.exec('CREATE TABLE data(key INTEGER PRIMARY KEY, value TEXT)');
targetDb.exec('CREATE TABLE data(key INTEGER PRIMARY KEY, value TEXT)');

const session = sourceDb.createSession();

const insert = sourceDb.prepare('INSERT INTO data (key, value) VALUES (?, ?)');
insert.run(1, 'hello');
insert.run(2, 'world');

const changeset = session.changeset();
targetDb.applyChangeset(changeset);
// Now that the changeset has been applied, targetDb contains the same data as sourceDb.
```

### `database[Symbol.dispose]()`

<!-- YAML
added:
  - v23.11.0
  - v22.15.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

关闭数据库连接。如果数据库连接已关闭，则此操作无效。

## 类：`Session`

<!-- YAML
added:
  - v23.3.0
  - v22.12.0
-->

### `session.changeset()`

<!-- YAML
added:
  - v23.3.0
  - v22.12.0
-->

* 返回值: {Uint8Array} 可以应用于其他数据库的二进制变更集。

检索自创建变更集以来所有更改的变更集。可以多次调用。如果数据库或会话未打开，则抛出异常。此方法是 [`sqlite3session_changeset()`][] 的封装。

### `session.patchset()`

<!-- YAML
added:
  - v23.3.0
  - v22.12.0
-->

* 返回值: {Uint8Array} 可以应用于其他数据库的二进制补丁集。

与上述方法类似，但生成更紧凑的补丁集。请参阅 SQLite 文档中的[变更集和补丁集][]。如果数据库或会话未打开，则抛出异常。此方法是 [`sqlite3session_patchset()`][] 的封装。

### `session.close()`.

关闭会话。如果数据库或会话未打开，则抛出异常。此方法是 [`sqlite3session_delete()`][] 的封装。

## 类：`StatementSync`

<!-- YAML
added: v22.5.0
-->

此类表示单个[预编译语句][]。此类无法通过其构造函数实例化。相反，实例是通过 `database.prepare()` 方法创建的。此类暴露的所有 API 都是同步执行的。

预编译语句是用于创建它的 SQL 的高效二进制表示。预编译语句是可参数化的，并且可以使用不同的绑定值多次调用。参数还提供针对 [SQL 注入][]攻击的保护。由于这些原因，在处理用户输入时，预编译语句优于手动构建的 SQL 字符串。

## 类：`SQLTagStore`

<!-- YAML
added: v24.9.0
-->

此类表示用于存储预编译语句的单个 LRU（最近最少使用）缓存。

此类的实例通过 database.createSQLTagStore() 方法创建，而不是使用构造函数。存储根据提供的 SQL 查询字符串缓存预编译语句。当再次看到相同的查询时，存储会检索缓存的语句并通过参数绑定安全地应用新值，从而防止 SQL 注入等攻击。

缓存有一个 maxSize，默认为 1000 个语句，但可以提供自定义大小（例如，database.createSQLTagStore(100)）。此类暴露的所有 API 都是同步执行的。

### `sqlTagStore.all(sqlTemplate[, ...values])`

<!-- YAML
added: v24.9.0
-->

* `sqlTemplate` {Template Literal} 包含 SQL 查询的模板字面量。
* `...values` {any} 要插入到模板字面量中的值。
* 返回值: {Array} 表示查询返回的行的对象数组。

执行给定的 SQL 查询并将所有结果行作为对象数组返回。

### `sqlTagStore.get(sqlTemplate[, ...values])`

<!-- YAML
added: v24.9.0
-->

* `sqlTemplate` {Template Literal} 包含 SQL 查询的模板字面量。
* `...values` {any} 要插入到模板字面量中的值。
* 返回值: {Object | undefined} 表示查询返回的第一行的对象，如果没有返回行，则为 `undefined`。

执行给定的 SQL 查询并将第一个结果行作为对象返回。

### `sqlTagStore.iterate(sqlTemplate[, ...values])`

<!-- YAML
added: v24.9.0
-->

* `sqlTemplate` {Template Literal} 包含 SQL 查询的模板字面量。
* `...values` {any} 要插入到模板字面量中的值。
* 返回值: {Iterator} 一个迭代器，生成表示查询返回的行的对象。

执行给定的 SQL 查询并返回一个遍历结果行的迭代器。

### `sqlTagStore.run(sqlTemplate[, ...values])`

<!-- YAML
added: v24.9.0
-->

* `sqlTemplate` {Template Literal} 包含 SQL 查询的模板字面量。
* `...values` {any} 要插入到模板字面量中的值。
* 返回值: {Object} 包含有关执行信息的对象，包括 `changes` 和 `lastInsertRowid`。

执行给定的 SQL 查询，预期不返回任何行（例如，INSERT、UPDATE、DELETE）。

### `sqlTagStore.size()`

<!-- YAML
added: v24.9.0
-->

* 返回值: {integer} 当前缓存中的预编译语句数量。

一个只读属性，返回当前缓存中的预编译语句数量。

### `sqlTagStore.capacity`

<!-- YAML
added: v24.9.0
-->

* 返回值: {integer} 缓存可以容纳的预编译语句的最大数量。

一个只读属性，返回缓存可以容纳的预编译语句的最大数量。

### `sqlTagStore.db`

<!-- YAML
added: v24.9.0
-->

* {DatabaseSync} 创建此 `SQLTagStore` 的 `DatabaseSync` 实例。

一个只读属性，返回与此 `SQLTagStore` 关联的 `DatabaseSync` 对象。

### `sqlTagStore.reset()`

<!-- YAML
added: v24.9.0
-->

重置 LRU 缓存，清除所有存储的预编译语句。

### `sqlTagStore.clear()`

<!-- YAML
added: v24.9.0
-->

`sqlTagStore.reset()` 的别名。

### `statement.all([namedParameters][, ...anonymousParameters])`

<!-- YAML
added: v22.5.0
changes:
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56385
    description: Add support for `DataView` and typed array objects for `anonymousParameters`.
-->

* `namedParameters` {Object} 用于绑定命名参数的可选对象。此对象的键用于配置映射。
* `...anonymousParameters` {null|number|bigint|string|Buffer|TypedArray|DataView} 零个或多个要绑定到匿名参数的值。
* 返回值: {Array} 对象数组。每个对象对应于执行预编译语句返回的一行。每个对象的键和值对应于行的列名和值。

此方法执行预编译语句并将所有结果作为对象数组返回。如果预编译语句未返回任何结果，则此方法返回空数组。预编译语句的[参数使用][] `namedParameters` 和 `anonymousParameters` 中的值进行绑定。

### `statement.columns()`

<!-- YAML
added: v23.11.0
-->

* 返回值: {Array} 对象数组。每个对象对应于预编译语句中的一列，并包含以下属性：

  * `column` {string|null} 源表中列的非别名名称，如果列是表达式或子查询的结果，则为 `null`。此属性是 [`sqlite3_column_origin_name()`][] 的结果。
  * `database` {string|null} 源数据库的非别名名称，如果列是表达式或子查询的结果，则为 `null`。此属性是 [`sqlite3_column_database_name()`][] 的结果。
  * `name` {string} 在 `SELECT` 语句的结果集中分配给列的名称。此属性是 [`sqlite3_column_name()`][] 的结果。
  * `table` {string|null} 源表的非别名名称，如果列是表达式或子查询的结果，则为 `null`。此属性是 [`sqlite3_column_table_name()`][] 的结果。
  * `type` {string|null} 列的声明数据类型，如果列是表达式或子查询的结果，则为 `null`。此属性是 [`sqlite3_column_decltype()`][] 的结果。

此方法用于检索有关预编译语句返回的列的信息。

### `statement.expandedSQL`

<!-- YAML
added: v22.5.0
-->

* 类型: {string} 扩展后包含参数值的源 SQL。

预编译语句的源 SQL 文本，其中参数占位符被替换为最近执行此预编译语句时使用的值。此属性是 [`sqlite3_expanded_sql()`][] 的封装。

### `statement.get([namedParameters][, ...anonymousParameters])`

<!-- YAML
added: v22.5.0
changes:
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56385
    description: Add support for `DataView` and typed array objects for `anonymousParameters`.
-->

* `namedParameters` {Object} 用于绑定命名参数的可选对象。此对象的键用于配置映射。
* `...anonymousParameters` {null|number|bigint|string|Buffer|TypedArray|DataView} 零个或多个要绑定到匿名参数的值。
* 返回值: {Object|undefined} 对应于执行预编译语句返回的第一行的对象。对象的键和值对应于行的列名和值。如果数据库未返回任何行，则此方法返回 `undefined`。

此方法执行预编译语句并将第一个结果作为对象返回。如果预编译语句未返回任何结果，则此方法返回 `undefined`。预编译语句的[参数使用][] `namedParameters` 和 `anonymousParameters` 中的值进行绑定。

### `statement.iterate([namedParameters][, ...anonymousParameters])`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
changes:
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56385
    description: Add support for `DataView` and typed array objects for `anonymousParameters`.
-->

* `namedParameters` {Object} 用于绑定命名参数的可选对象。此对象的键用于配置映射。
* `...anonymousParameters` {null|number|bigint|string|Buffer|TypedArray|DataView} 零个或多个要绑定到匿名参数的值。
* 返回值: {Iterator} 对象可迭代迭代器。每个对象对应于执行预编译语句返回的一行。每个对象的键和值对应于行的列名和值。

此方法执行预编译语句并返回对象的迭代器。如果预编译语句未返回任何结果，则此方法返回空迭代器。预编译语句的[参数使用][] `namedParameters` 和 `anonymousParameters` 中的值进行绑定。

### `statement.run([namedParameters][, ...anonymousParameters])`

<!-- YAML
added: v22.5.0
changes:
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56385
    description: Add support for `DataView` and typed array objects for `anonymousParameters`.
-->

* `namedParameters` {Object} 用于绑定命名参数的可选对象。此对象的键用于配置映射。
* `...anonymousParameters` {null|number|bigint|string|Buffer|TypedArray|DataView} 零个或多个要绑定到匿名参数的值。
* 返回值: {Object}
  * `changes` {number|bigint} 最近完成的 `INSERT`、`UPDATE` 或 `DELETE` 语句修改、插入或删除的行数。此字段是数字或 `BigInt`，具体取决于预编译语句的配置。此属性是 [`sqlite3_changes64()`][] 的结果。
  * `lastInsertRowid` {number|bigint} 最近插入的 rowid。此字段是数字或 `BigInt`，具体取决于预编译语句的配置。此属性是 [`sqlite3_last_insert_rowid()`][] 的结果。

此方法执行预编译语句并返回一个总结结果更改的对象。预编译语句的[参数使用][] `namedParameters` 和 `anonymousParameters` 中的值进行绑定。

### `statement.setAllowBareNamedParameters(enabled)`

<!-- YAML
added: v22.5.0
-->

* `enabled` {boolean} 启用或禁用支持绑定不带前缀字符的命名参数。

SQLite 参数的名称以前缀字符开头。默认情况下，`node:sqlite` 要求在绑定参数时存在此前缀字符。但是，除了美元符号字符外，这些前缀字符在用作对象键时还需要额外的引号。

为了提高人体工程学，可以使用此方法来允许裸命名参数，这些参数在 JavaScript 代码中不需要前缀字符。启用裸命名参数时需要注意几个注意事项：

* SQL 中仍然需要前缀字符。
* JavaScript 中仍然允许前缀字符。实际上，带前缀的名称在绑定性能上会稍好一些。
* 在同一个预编译语句中使用模糊的命名参数，例如 `$k` 和 `@k`，将导致异常，因为无法确定如何绑定裸名称。

### `statement.setAllowUnknownNamedParameters(enabled)`

<!-- YAML
added:
  - v23.11.0
  - v22.15.0
-->

* `enabled` {boolean} 启用或禁用支持未知命名参数。

默认情况下，如果在绑定参数时遇到未知名称，则会抛出异常。此方法允许忽略未知命名参数。

### `statement.setReturnArrays(enabled)`

<!-- YAML
added: v24.0.0
-->

* `enabled` {boolean} 启用或禁用将查询结果作为数组返回。

启用后，`all()`、`get()` 和 `iterate()` 方法返回的查询结果将作为数组而不是对象返回。

### `statement.setReadBigInts(enabled)`

<!-- YAML
added: v22.5.0
-->

* `enabled` {boolean} 启用或禁用从数据库读取 `INTEGER` 字段时使用 `BigInt`。

从数据库读取时，SQLite `INTEGER` 默认映射到 JavaScript 数字。但是，SQLite `INTEGER` 可以存储比 JavaScript 数字能够表示的值更大的值。在这种情况下，可以使用此方法使用 JavaScript `BigInt` 读取 `INTEGER` 数据。此方法对数据库写操作没有影响，在写操作中数字和 `BigInt` 始终都受支持。

### `statement.sourceSQL`

<!-- YAML
added: v22.5.0
-->

* 类型: {string} 用于创建此预编译语句的源 SQL。

预编译语句的源 SQL 文本。此属性是 [`sqlite3_sql()`][] 的封装。

### JavaScript 和 SQLite 之间的类型转换

当 Node.js 写入或读取 SQLite 时，需要在 JavaScript 数据类型和 SQLite 的[数据类型][]之间进行转换。由于 JavaScript 支持的数据类型比 SQLite 多，因此仅支持 JavaScript 类型的子集。尝试将不受支持的数据类型写入 SQLite 将导致异常。

| SQLite    | JavaScript                 |
| --------- | -------------------------- |
| `NULL`    | {null}                     |
| `INTEGER` | {number} 或 {bigint}       |
| `REAL`    | {number}                   |
| `TEXT`    | {string}                   |
| `BLOB`    | {TypedArray} 或 {DataView} |

## `sqlite.backup(sourceDb, path[, options])`

<!-- YAML
added: v23.8.0
changes:
  - version: v23.10.0
    pr-url: https://github.com/nodejs/node/pull/56991
    description: The `path` argument now supports Buffer and URL objects.
-->

* `sourceDb` {DatabaseSync} 要备份的数据库。源数据库必须打开。
* `path` {string | Buffer | URL} 备份创建的位置。如果文件已存在，其内容将被覆盖。
* `options` {Object} 备份的可选配置。支持以下属性：
  * `source` {string} 源数据库的名称。可以是 `'main'`（默认的主数据库）或任何其他使用 [`ATTACH DATABASE`][] 添加的数据库。**默认值:** `'main'`。
  * `target` {string} 目标数据库的名称。可以是 `'main'`（默认的主数据库）或任何其他使用 [`ATTACH DATABASE`][] 添加的数据库。**默认值:** `'main'`。
  * `rate` {number} 每个备份批次中要传输的页面数。**默认值:** `100`。
  * `progress` {Function} 一个可选的回调函数，将在每个备份步骤后调用。传递给此回调的参数是一个 {Object}，具有 `remainingPages` 和 `totalPages` 属性，描述备份操作的当前进度。
* 返回值: {Promise} 一个 Promise，在完成时以备份的总页数兑现，或者在发生错误时拒绝。

此方法进行数据库备份。此方法抽象了 [`sqlite3_backup_init()`][]、[`sqlite3_backup_step()`][] 和 [`sqlite3_backup_finish()`][] 函数。

备份的数据库在备份过程中可以正常使用。来自同一连接（同一个 {DatabaseSync} 对象）的变更将立即反映在备份中。但是，来自其他连接的变更将导致备份过程重新启动。

```cjs
const { backup, DatabaseSync } = require('node:sqlite');

(async () => {
  const sourceDb = new DatabaseSync('source.db');
  const totalPagesTransferred = await backup(sourceDb, 'backup.db', {
    rate: 1, // Copy one page at a time.
    progress: ({ totalPages, remainingPages }) => {
      console.log('Backup in progress', { totalPages, remainingPages });
    },
  });

  console.log('Backup completed', totalPagesTransferred);
})();
```

```mjs
import { backup, DatabaseSync } from 'node:sqlite';

const sourceDb = new DatabaseSync('source.db');
const totalPagesTransferred = await backup(sourceDb, 'backup.db', {
  rate: 1, // Copy one page at a time.
  progress: ({ totalPages, remainingPages }) => {
    console.log('Backup in progress', { totalPages, remainingPages });
  },
});

console.log('Backup completed', totalPagesTransferred);
```

## `sqlite.constants`

<!-- YAML
added:
  - v23.5.0
  - v22.13.0
-->

* 类型: {Object}

包含 SQLite 操作常用常量的对象。

### SQLite 常量

以下常量由 `sqlite.constants` 对象导出。

#### 冲突解决常量

以下常量之一可作为参数传递给传递给 [`database.applyChangeset()`][] 的 `onConflict` 冲突解决处理程序。另请参阅 SQLite 文档中的[传递给冲突处理程序的常量][]。

<table>
  <tr>
    <th>常量</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_DATA</code></td>
    <td>当处理 DELETE 或 UPDATE 更改时，如果数据库中存在具有所需 PRIMARY KEY 字段的行，但更新修改的一个或多个其他（非主键）字段不包含预期的“之前”值，则使用此常量调用冲突处理程序。</td>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_NOTFOUND</code></td>
    <td>当处理 DELETE 或 UPDATE 更改时，如果数据库中不存在具有所需 PRIMARY KEY 字段的行，则使用此常量调用冲突处理程序。</td>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_CONFLICT</code></td>
    <td>在处理 INSERT 更改时，如果操作会导致重复的主键值，则将此常量传递给冲突处理程序。</td>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_CONSTRAINT</code></td>
    <td>如果启用了外键处理，并且应用变更集使数据库处于包含外键冲突的状态，则在提交变更集之前，使用此常量调用冲突处理程序一次。如果冲突处理程序返回 <code>SQLITE_CHANGESET_OMIT</code>，则提交更改，包括导致外键约束冲突的更改。或者，如果返回 <code>SQLITE_CHANGESET_ABORT</code>，则回滚变更集。</td>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_FOREIGN_KEY</code></td>
    <td>如果在应用更改时发生任何其他约束冲突（即 UNIQUE、CHECK 或 NOT NULL 约束），则使用此常量调用冲突处理程序。</td>
  </tr>
</table>

以下常量之一必须从传递给 [`database.applyChangeset()`][] 的 `onConflict` 冲突解决处理程序返回。另请参阅 SQLite 文档中的[从冲突处理程序返回的常量][]。

<table>
  <tr>
    <th>常量</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_OMIT</code></td>
    <td>忽略冲突的更改。</td>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_REPLACE</code></td>
    <td>冲突的更改替换现有值。请注意，仅当冲突类型为 <code>SQLITE_CHANGESET_DATA</code> 或 <code>SQLITE_CHANGESET_CONFLICT</code> 时才能返回此值。</td>
  </tr>
  <tr>
    <td><code>SQLITE_CHANGESET_ABORT</code></td>
    <td>当更改遇到冲突时中止并回滚数据库。</td>
  </tr>
</table>

#### 授权常量

以下常量与 [`database.setAuthorizer()`][] 方法一起使用。

##### 授权结果码

以下常量之一必须从传递给 [`database.setAuthorizer()`][] 的授权器回调函数返回。

<table>
  <tr>
    <th>常量</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>SQLITE_OK</code></td>
    <td>允许操作正常进行。</td>
  </tr>
  <tr>
    <td><code>SQLITE_DENY</code></td>
    <td>拒绝操作并导致返回错误。</td>
  </tr>
  <tr>
    <td><code>SQLITE_IGNORE</code></td>
    <td>忽略操作并继续，就像从未请求过一样。</td>
  </tr>
</table>

##### 授权操作码

以下常量作为第一个参数传递给授权器回调函数，以指示正在授权的操作类型。

<table>
  <tr>
    <th>常量</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_INDEX</code></td>
    <td>创建索引</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_TABLE</code></td>
    <td>创建表</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_TEMP_INDEX</code></td>
    <td>创建临时索引</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_TEMP_TABLE</code></td>
    <td>创建临时表</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_TEMP_TRIGGER</code></td>
    <td>创建临时触发器</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_TEMP_VIEW</code></td>
    <td>创建临时视图</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_TRIGGER</code></td>
    <td>创建触发器</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_VIEW</code></td>
    <td>创建视图</td>
  </tr>
  <tr>
    <td><code>SQLITE_DELETE</code></td>
    <td>从表中删除</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_INDEX</code></td>
    <td>删除索引</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_TABLE</code></td>
    <td>删除表</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_TEMP_INDEX</code></td>
    <td>删除临时索引</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_TEMP_TABLE</code></td>
    <td>删除临时表</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_TEMP_TRIGGER</code></td>
    <td>删除临时触发器</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_TEMP_VIEW</code></td>
    <td>删除临时视图</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_TRIGGER</code></td>
    <td>删除触发器</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_VIEW</code></td>
    <td>删除视图</td>
  </tr>
  <tr>
    <td><code>SQLITE_INSERT</code></td>
    <td>插入到表中</td>
  </tr>
  <tr>
    <td><code>SQLITE_PRAGMA</code></td>
    <td>执行 PRAGMA 语句</td>
  </tr>
  <tr>
    <td><code>SQLITE_READ</code></td>
    <td>从表中读取</td>
  </tr>
  <tr>
    <td><code>SQLITE_SELECT</code></td>
    <td>执行 SELECT 语句</td>
  </tr>
  <tr>
    <td><code>SQLITE_TRANSACTION</code></td>
    <td>开始、提交或回滚事务</td>
  </tr>
  <tr>
    <td><code>SQLITE_UPDATE</code></td>
    <td>更新表</td>
  </tr>
  <tr>
    <td><code>SQLITE_ATTACH</code></td>
    <td>附加数据库</td>
  </tr>
  <tr>
    <td><code>SQLITE_DETACH</code></td>
    <td>分离数据库</td>
  </tr>
  <tr>
    <td><code>SQLITE_ALTER_TABLE</code></td>
    <td>更改表</td>
  </tr>
  <tr>
    <td><code>SQLITE_REINDEX</code></td>
    <td>重新索引</td>
  </tr>
  <tr>
    <td><code>SQLITE_ANALYZE</code></td>
    <td>分析数据库</td>
  </tr>
  <tr>
    <td><code>SQLITE_CREATE_VTABLE</code></td>
    <td>创建虚拟表</td>
  </tr>
  <tr>
    <td><code>SQLITE_DROP_VTABLE</code></td>
    <td>删除虚拟表</td>
  </tr>
  <tr>
    <td><code>SQLITE_FUNCTION</code></td>
    <td>使用函数</td>
  </tr>
  <tr>
    <td><code>SQLITE_SAVEPOINT</code></td>
    <td>创建、释放或回滚保存点</td>
  </tr>
  <tr>
    <td><code>SQLITE_COPY</code></td>
    <td>复制数据（旧版）</td>
  </tr>
  <tr>
    <td><code>SQLITE_RECURSIVE</code></td>
    <td>递归查询</td>
  </tr>
</table>

[Changesets and Patchsets]: https://www.sqlite.org/sessionintro.html#changesets_and_patchsets
[Constants Passed To The Conflict Handler]: https://www.sqlite.org/session/c_changeset_conflict.html
[Constants Returned From The Conflict Handler]: https://www.sqlite.org/session/c_changeset_abort.html
[SQL injection]: https://en.wikipedia.org/wiki/SQL_injection
[Type conversion between JavaScript and SQLite]: #type-conversion-between-javascript-and-sqlite
[`ATTACH DATABASE`]: https://www.sqlite.org/lang_attach.html
[`PRAGMA foreign_keys`]: https://www.sqlite.org/pragma.html#pragma_foreign_keys
[`SQLITE_DETERMINISTIC`]: https://www.sqlite.org/c3ref/c_deterministic.html
[`SQLITE_DIRECTONLY`]: https://www.sqlite.org/c3ref/c_deterministic.html
[`SQLITE_MAX_FUNCTION_ARG`]: https://www.sqlite.org/limits.html#max_function_arg
[`database.applyChangeset()`]: #databaseapplychangesetchangeset-options
[`database.setAuthorizer()`]: #databasesetauthorizercallback
[`sqlite3_backup_finish()`]: https://www.sqlite.org/c3ref/backup_finish.html#sqlite3backupfinish
[`sqlite3_backup_init()`]: https://www.sqlite.org/c3ref/backup_finish.html#sqlite3backupinit
[`sqlite3_backup_step()`]: https://www.sqlite.org/c3ref/backup_finish.html#sqlite3backupstep
[`sqlite3_changes64()`]: https://www.sqlite.org/c3ref/changes.html
[`sqlite3_close_v2()`]: https://www.sqlite.org/c3ref/close.html
[`sqlite3_column_database_name()`]: https://www.sqlite.org/c3ref/column_database_name.html
[`sqlite3_column_decltype()`]: https://www.sqlite.org/c3ref/column_decltype.html
[`sqlite3_column_name()`]: https://www.sqlite.org/c3ref/column_name.html
[`sqlite3_column_origin_name()`]: https://www.sqlite.org/c3ref/column_database_name.html
[`sqlite3_column_table_name()`]: https://www.sqlite.org/c3ref/column_database_name.html
[`sqlite3_create_function_v2()`]: https://www.sqlite.org/c3ref/create_function.html
[`sqlite3_create_window_function()`]: https://www.sqlite.org/c3ref/create_function.html
[`sqlite3_db_filename()`]: https://sqlite.org/c3ref/db_filename.html
[`sqlite3_exec()`]: https://www.sqlite.org/c3ref/exec.html
[`sqlite3_expanded_sql()`]: https://www.sqlite.org/c3ref/expanded_sql.html
[`sqlite3_get_autocommit()`]: https://sqlite.org/c3ref/get_autocommit.html
[`sqlite3_last_insert_rowid()`]: https://www.sqlite.org/c3ref/last_insert_rowid.html
[`sqlite3_load_extension()`]: https://www.sqlite.org/c3ref/load_extension.html
[`sqlite3_prepare_v2()`]: https://www.sqlite.org/c3ref/prepare.html
[`sqlite3_set_authorizer()`]: https://sqlite.org/c3ref/set_authorizer.html
[`sqlite3_sql()`]: https://www.sqlite.org/c3ref/expanded_sql.html
[`sqlite3changeset_apply()`]: https://www.sqlite.org/session/sqlite3changeset_apply.html
[`sqlite3session_attach()`]: https://www.sqlite.org/session/sqlite3session_attach.html
[`sqlite3session_changeset()`]: https://www.sqlite.org/session/sqlite3session_changeset.html
[`sqlite3session_create()`]: https://www.sqlite.org/session/sqlite3session_create.html
[`sqlite3session_delete()`]: https://www.sqlite.org/session/sqlite3session_delete.html
[`sqlite3session_patchset()`]: https://www.sqlite.org/session/sqlite3session_patchset.html
[busy timeout]: https://sqlite.org/c3ref/busy_timeout.html
[connection]: https://www.sqlite.org/c3ref/sqlite3.html
[data types]: https://www.sqlite.org/datatype3.html
[double-quoted string literals]: https://www.sqlite.org/quirks.html#dblquote
[in memory]: https://www.sqlite.org/inmemorydb.html
[parameters are bound]: https://www.sqlite.org/c3ref/bind_blob.html
[prepared statement]: https://www.sqlite.org/c3ref/stmt.html
