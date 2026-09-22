# Plan: Add Remote MySQL Procedure and Function Calls

## 1. Scope and current state

The MySQL extension currently supports attaching remote MySQL databases, scanning remote tables, executing raw SQL, and clearing its cache. It does not provide a dedicated API for invoking remote stored procedures, including procedures that return result sets or use `OUT`/`INOUT` parameters.

The implementation should target the separate MySQL extension repository/source tree used by DuckDB, especially:

- `src/mysql_connection.cpp` and `src/include/mysql_connection.hpp`
- `src/mysql_result.cpp` and `src/include/mysql_result.hpp`
- `src/mysql_scanner.cpp` and `src/include/mysql_scanner.hpp`
- `src/mysql_extension.cpp`
- `src/mysql_parameter.cpp`
- `src/include/mysql_statement.hpp`
- `src/storage/mysql_connection_pool.*`
- SQL tests and `README.md`

## 2. Goals

1. Provide a safe DuckDB-facing API for calling remote MySQL stored procedures.
2. Support `IN` parameters using prepared statements and parameter binding.
3. Support procedures that return one or more result sets.
4. Support `OUT` and `INOUT` parameters, initially using connection-local session variables if native connector support is insufficient.
5. Preserve connection-pool correctness by keeping the complete call and output-parameter retrieval sequence on one pinned connection.
6. Provide clear errors for missing procedures, invalid argument counts, unsupported output shapes, and remote MySQL errors.
7. Document the feature and cover it with integration tests.

## 3. Proposed SQL interface

Start with a table-function interface so the feature can be implemented entirely in the extension without changing DuckDB's SQL parser:

```sql
ATTACH 'host=... user=... password=... database=...' AS mysql_db (TYPE mysql);

SELECT *
FROM mysql_call('mysql_db', 'report_sales', 2026, 'US');
```

The first version should support a procedure name and positional values. A later version may add explicit parameter metadata or named parameters.

Remote scalar functions already fit naturally through a remote query such as:

```sql
SELECT *
FROM mysql_query('mysql_db', 'SELECT my_function(?)', [42]);
```

Therefore, first verify whether a dedicated function wrapper is actually needed. Avoid adding a second API unless it provides capabilities that `mysql_query` cannot provide.

## 4. Architecture

### 4.1 Add procedure-call bind and execution state

Add a `MySQLCallFunction` table function and corresponding bind data, following the patterns used by `MySQLQueryFunction` and `MySQLQueryBindData`.

The bind data should contain at least:

- attached catalog name
- validated and safely quoted procedure identifier
- input parameter values
- output-parameter descriptors
- selected result-set policy
- prepared-statement metadata, if preparation occurs during bind

The execution state should own or pin the connection for the complete operation and retain the current result set while rows are streamed to DuckDB.

### 4.2 Add multi-result handling

A MySQL procedure can produce multiple results. Extend the connector/result abstraction to:

1. execute the `CALL` statement;
2. expose the current result set;
3. iterate through subsequent results using the MariaDB/MySQL client API;
4. drain every remaining result before returning the connection to the pool;
5. preserve the first result set for the initial table-function implementation.

The initial output policy should be explicit and documented:

- If there is one result set, return it.
- If there are multiple result sets, return the first result set and drain the rest.
- If there are no row-producing result sets, return a single status row or an empty result according to the selected API contract.

A later enhancement can expose a result-set index or a nested result representation.

### 4.3 Connection capabilities

Review connection creation in `mysql_connection.cpp` and ensure the required multi-result capability is requested. Add only the minimum client flags necessary.

Audit the effect on existing operations, especially `mysql_execute`, because enabling multi-statement behavior can change how semicolon-separated SQL is handled. Do not weaken parameter binding or introduce string concatenation for values.

### 4.4 Parameter binding

For `IN` parameters:

- use prepared statements and bound values;
- reuse existing value-to-MySQL type conversion code;
- never concatenate user-provided values into SQL text.

For procedure and schema identifiers:

- validate identifiers;
- quote them with the existing MySQL identifier writer;
- reject malformed or qualified names that are ambiguous under the API contract.

For `OUT` and `INOUT` parameters, implement the MVP with connection-local variables:

```sql
CALL `schema`.`proc`(?, @duckdb_out_0, @duckdb_out_1);
SELECT @duckdb_out_0, @duckdb_out_1;
```

Use unique generated variable names and execute the `CALL` plus output retrieval on the same pinned connection. Document that output-parameter metadata or an explicit parameter mode API is required to distinguish `IN`, `OUT`, and `INOUT` values.

Native output binding through `MYSQL_STMT` can be considered after the session-variable approach is working.

### 4.5 Result schema

For the MVP:

- expose the columns of the first remote result set directly;
- if no result set exists, return a status row containing affected-row count and/or success status;
- expose output parameters either as trailing columns or as a separate single-row result, choosing one contract and documenting it clearly.

Prefer a stable schema per invocation. Do not attempt to combine unrelated schemas from multiple result sets into one table.

## 5. Implementation phases

### Phase 1: investigate and define contracts

1. Trace the current query execution path from `MySQLQueryFunction` through connection pooling and `MySQLResult`.
2. Confirm whether the connector currently supports `mysql_next_result` or `mysql_stmt_next_result`.
3. Confirm how prepared statements, result metadata, NULL values, and type conversion are represented.
4. Decide the exact SQL signature and output schema for `mysql_call`.
5. Decide whether the first release returns only the first result set or allows a result-set selector.

### Phase 2: safe result draining

1. Add a reusable multi-result iterator to the connection/result layer.
2. Ensure all result packets are consumed before a pooled connection is released.
3. Add tests for a procedure that returns two `SELECT` result sets.
4. Add failure-path tests where result iteration is interrupted by a remote error.

### Phase 3: `mysql_call` MVP

1. Add `MySQLCallBindData` and `MySQLCallFunction`.
2. Register the function in `mysql_extension.cpp`.
3. Resolve the attached `MySQLCatalog` and acquire a pinned connection.
4. Construct and execute `CALL schema.proc(?, ?, ...)` with prepared input parameters.
5. Stream the first result set using existing MySQL field/type conversion logic.
6. Drain all remaining result sets.
7. Return clear errors for missing procedure, bad argument count, and remote execution failures.

### Phase 4: `OUT` and `INOUT` parameters

1. Add an explicit parameter-mode representation to bind data.
2. Generate collision-resistant session-variable names.
3. Execute the call and output retrieval on one pinned connection.
4. Convert output values through existing MySQL-to-DuckDB conversion code.
5. Define and test behavior for NULL output values, binary values, decimals, dates, and large text values.

### Phase 5: metadata and validation

Optionally query `information_schema.ROUTINES` and `information_schema.PARAMETERS` to validate:

- procedure existence;
- parameter count;
- parameter modes;
- declared types;
- overloaded or ambiguous names, where supported by MySQL/MariaDB.

Do not require metadata discovery for the MVP if it would add latency or privilege requirements. Runtime MySQL errors remain authoritative.

### Phase 6: scalar and table functions

Verify whether `mysql_query` already supports remote scalar function invocation through `SELECT function_name(...)`. If it does, document that path rather than adding redundant extension APIs.

If a dedicated API is needed, define it separately from procedure calls because scalar functions have expression semantics and table-valued functions have different result-shape requirements.

### Phase 7: documentation and release hardening

Update `README.md` with:

- basic procedure call example;
- parameter binding examples;
- result-set behavior;
- `OUT`/`INOUT` behavior;
- transaction and connection-pinning considerations;
- limitations around multiple result sets and privileges.

Add SQL logic/integration tests for:

- no-argument procedure;
- `IN` parameters of several types;
- one result set;
- multiple result sets;
- no result set;
- `OUT` and `INOUT` parameters;
- NULL and large values;
- missing procedure;
- wrong argument count;
- remote procedure error;
- connection reuse after success and failure.

## 6. Security and correctness requirements

- Never interpolate parameter values into SQL.
- Quote only validated identifiers.
- Ensure generated session-variable names cannot collide with application variables or user input.
- Pin the connection until all result sets and output-variable reads are complete.
- Drain all pending results before returning a connection to the pool.
- Avoid leaking passwords or procedure arguments in error messages and logs.
- Verify behavior under concurrent calls using separate pooled connections.
- Audit whether enabling multi-result or multi-statement client flags changes existing APIs.

## 7. Recommended MVP

The first pull request should implement:

1. multi-result consumption and safe draining;
2. `mysql_call(catalog, procedure_name, ...)`;
3. prepared `IN` parameter binding;
4. first-result-set streaming;
5. clear limitation/documentation for additional result sets;
6. comprehensive integration tests;
7. README examples.

Defer `OUT`/`INOUT` parameters, information-schema validation, and nested multi-result output until the MVP proves the connection and result lifecycle is correct.

## 8. Acceptance criteria

The feature is ready for review when:

- a remote procedure with `IN` parameters can be invoked from DuckDB;
- returned rows have correct names, types, NULL handling, and values;
- all remote results are consumed before connection reuse;
- existing table scans, `mysql_query`, and `mysql_execute` tests remain unchanged and pass;
- failures leave the connection pool in a reusable state;
- the SQL API and limitations are documented;
- tests cover both MySQL and MariaDB-compatible server behavior where supported.
