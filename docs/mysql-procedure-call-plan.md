# Plan: Add Remote MySQL Stored Procedure Calls

## 1. Scope and current state

The MySQL extension currently supports attaching remote MySQL databases, scanning remote tables, executing raw SQL, and clearing its cache. It does not provide a dedicated API for invoking remote stored procedures, including procedures that return result sets.

This plan intentionally narrows the scope to remote stored procedures, not remote scalar functions in general. The extension already exposes raw SQL execution through `mysql_query`, which can be used for expressions like `SELECT my_function(?)` or `SELECT * FROM mysql_query('db', 'SELECT my_function(?)', params=row(...))`. If a dedicated function wrapper is needed later, it should be justified by a concrete ergonomic or metadata need.

The implementation should target the separate MySQL extension repository/source tree used by DuckDB, especially:

- `src/mysql_connection.cpp` and `src/include/mysql_connection.hpp`
- `src/mysql_result.cpp` and `src/include/mysql_result.hpp`
- `src/mysql_scanner.cpp` and `src/include/mysql_scanner.hpp`
- `src/mysql_extension.cpp`
- `src/mysql_parameter.cpp`
- `src/include/mysql_statement.hpp`
- `src/storage/mysql_connection_pool.*`
- `src/storage/mysql_optimizer.cpp`
- `src/CMakeLists.txt`
- SQL tests under `test/sql/`
- `README.md`

## 2. Goals

1. Add a safe DuckDB-facing API for invoking remote MySQL stored procedures.
2. Support `IN` parameters using prepared statements and parameter binding.
3. Support procedures that return a result set.
4. Ensure the remote result stream is fully consumed before a pooled connection is returned.
5. Preserve connection-pool correctness by keeping the complete `CALL` + result-draining sequence on one pinned connection.
6. Return clear errors for missing procedures, invalid argument counts, unsupported output shapes, and remote MySQL errors.
7. Document the feature and cover it with integration tests.

## 3. Proposed SQL interface

Initial public API: extension table function, not native DuckDB `CALL` integration.

```sql
ATTACH 'host=... user=... password=... database=...' AS mysql_db (TYPE mysql);

SELECT *
FROM mysql_call('mysql_db', 'report_sales', params := row(2026, 'US'));
```

Alternative accepted variant if the function is designed like `mysql_query`:

```sql
SELECT *
FROM mysql_call('mysql_db', 'report_sales', params = row(2026, 'US'));
```

The exact argument pattern should be finalized during implementation, but it must keep the same properties as the existing extension APIs:

- parameter values must be passed as a `STRUCT`/`ROW` list, not as string concatenation;
- the catalog/attached database name is explicit;
- the procedure name is explicit and identifier-safe;
- the API remains inside the extension, with no parser/core changes.

For scalar functions, the recommended path remains:

```sql
SELECT *
FROM mysql_query('mysql_db', 'SELECT my_function(?)', params := row(42));
```

This is already consistent with the extension's approach to remote SQL execution. A dedicated `mysql_function` wrapper should be added only if it provides real value beyond the `mysql_query` based pattern.

## 4. Design comparison: `mysql_call` versus native DuckDB `CALL`

### 4.1 Native DuckDB `CALL`

Native DuckDB `CALL` is the natural SQL syntax for invoking a procedure registered in DuckDB's catalog:

```sql
CALL local_procedure(2026, 'US');
```

This is a good fit for local procedures that are part of DuckDB's catalog, binder, and planner. However, it is not automatically the right mechanism for a remote MySQL procedure, because a MySQL procedure is not a local DuckDB procedure and has MySQL-specific semantics around:

- parameter modes (`IN`, `OUT`, `INOUT`);
- multi-result procedures;
- result-set cardinality;
- remote transaction behavior;
- server-specific metadata.

A direct remote invocation such as:

```sql
CALL mysql_db.report_sales(2026, 'US');
```

would require remote catalog registration or a special remote-call path in the planner. That is a bigger and higher-risk design than the extension-native API.

### 4.2 Extension table function: `mysql_call`

The extension-native approach is:

```sql
SELECT *
FROM mysql_call('mysql_db', 'report_sales', params := row(2026, 'US'));
```

Advantages:

- no DuckDB core parser changes;
- no new core procedure contract is required;
- the feature remains inside the MySQL extension;
- result-set handling is explicitly defined by the extension;
- it can reuse the existing `mysql_query`, connection-pool, prepared-statement, and result-streaming architecture;
- it keeps remote catalog and procedure names explicit;
- it is low-risk and additive.

Disadvantages:

- it is more verbose than native `CALL`;
- it does not integrate with DuckDB's native procedure metadata/autocomplete model;
- it cannot naturally express a remote procedure call with the same ergonomics as a local DuckDB procedure.

### 4.3 Recommended staged approach

Use `mysql_call` as the initial public API and treat native `CALL` integration as a later enhancement only if there is a clear need.

| Concern | `mysql_call` table function | Native DuckDB `CALL` integration |
| --- | --- | --- |
| Implementation location | MySQL extension only | DuckDB core + extension integration |
| Parser changes | None | Required for dotted remote procedure syntax |
| Result-set handling | Explicitly designed in the extension | Must fit a general DuckDB procedure contract |
| `OUT` / `INOUT` handling | Extension-specific API contract | Requires core-level representation and binder semantics |
| Multi-result sets | Can define a first-result policy | Needs a core-level representation or a truncation policy |
| Security boundary | Explicit remote catalog and procedure names | More implicit after catalog registration |
| Backward compatibility | Low risk and additive | Higher risk and more invasive |
| User ergonomics | More verbose | More concise |

### 4.4 Decision

The MVP will use `mysql_call` as the supported remote procedure API. The extension should continue using `mysql_query` for scalar function calls and arbitrary remote SQL. Native DuckDB `CALL` syntax remains a future enhancement for later catalog integration, not part of the first implementation.

## 5. Architecture

### 5.1 Add procedure-call bind and execution state

Add a `MySQLCallFunction` table function and `MySQLCallBindData`, modeled on `MySQLQueryFunction` and `MySQLQueryBindData`.

Required bind data fields:

- attached catalog name
- procedure identifier, safely quoted or validated
- input parameter values
- parameter-count/metadata contract (initially `IN` only)
- selected result-set policy
- prepared-statement metadata if the query is prepared during bind

The execution state must pin the connection for the complete remote call and keep the underlying result object alive until all rows are drained or an error occurs.

### 5.2 Multi-result handling

Stored procedures can emit more than one result set. The initial MVP must define a single, stable contract:

- if the procedure returns a row-producing result set, return that result set;
- if it does not, return a single `rowcount BIGINT` result or an explicit empty result as defined by the function contract;
- if it returns multiple result sets, only the first result set is surfaced to the user by default, and the remaining result sets are drained and discarded.

The implementation must drain all outstanding remote results before releasing the connection back to the pool.

This requires adding or extending the result API to support iteration over multiple MySQL result sets and to explicitly skip or drain the remaining ones. The plan must confirm whether the client library exposes `mysql_next_result` (basic API) or `mysql_stmt_next_result` (prepared-statement API) and implement one consistent flow around it.

### 5.3 Connection capabilities

Before changing connection flags, verify the current default connection config used by MySQLUtils/Connect. The current code already includes `CLIENT_MULTI_STATEMENTS` and should be checked for whether it also satisfies the required multi-result behavior for stored procedures.

The plan must not assume that multi-result capability is missing without verifying the actual client/library side. The implementation should:

- confirm whether `CLIENT_MULTI_RESULTS` is already present or required;
- avoid broad client-flag changes unless tests prove they are needed;
- audit the impact on `mysql_query` and `mysql_execute` behavior.

### 5.4 Parameter binding and safety

For `IN` parameters:

- use prepared statements and bound values;
- reuse existing value-to-MySQL type conversion code;
- never concatenate user input into SQL text;
- preserve `NULL`/`BOOLEAN`/`DATE`/`TIME`/large-text conversions through the existing conversion pipeline.

For procedure identifiers and schema names:

- validate them;
- quote them with the existing MySQL identifier writer;
- reject malformed or ambiguous names.

For `OUT` and `INOUT` parameters:

- defer them until after the `IN`-only MVP works;
- do not promise them in the first PR;
- if prototyping is required, use a connection-local session-variable pattern only as an implementation experiment, not as the final API contract.

This keeps the initial plan implementation-safe and reviewable.

### 5.5 Result schema and output contract

The MVP should fix a single contract:

- if the remote procedure returns rows, expose those columns directly;
- if it does not, expose `rowcount BIGINT` (or another explicit status shape) and document that output is status-only.

This is simpler and more consistent with the existing `mysql_query` design and avoids trying to merge multiple result-set schemas into one table.

### 5.6 Transaction and connection semantics

The plan must explicitly define transaction behavior. `mysql_query` already has a dedicated connection/pinning mechanism. `mysql_call` should follow the same rules:

- by default, use the same pinned/transaction connection semantics as `mysql_query`;
- allow an explicit connection override only if the existing API already supports that pattern;
- pin the connection until the full stored-procedure execution and result draining are complete;
- do not return the connection to the pool until all results are consumed and any output retrieval/cleanup is done.

This is critical because a partially consumed result set would leave the pool in a bad state and could poison subsequent queries.

## 6. Implementation phases

### Phase 1: investigate and define the contract

1. Trace the query execution flow for `mysql_query` and `mysql_execute` through `MySQLConnection`, `MySQLStatement`, and the connection pool.
2. Confirm whether the library/client already supports multi-result handling in the current connection configuration.
3. Confirm how prepared statements and result metadata behave for `CALL` queries.
4. Decide the final `mysql_call` API shape (`params := row(...)` or equivalent) and the exact rowcount/status contract.
5. Decide whether the first release exposes only the first result set or supports a selector in the future.

### Phase 2: safe result-draining and connection lifecycle

1. Add a reusable multi-result iterator or helper in the connection/result layer.
2. Ensure all remote result sets are consumed before releasing a pooled connection.
3. Add tests for procedures generating two or more result sets.
4. Add failure-path tests where the remote procedure errors mid-cycle or leaves pending results.
5. Add check coverage for `DETACH`, prepared statements, and connection reuse after failure.

### Phase 3: `mysql_call` MVP

1. Add `MySQLCallBindData` and `MySQLCallFunction`.
2. Register the function in `mysql_extension.cpp`.
3. Resolve the attached catalog and acquire a pinned connection as needed.
4. Build a parameterized `CALL schema.proc(...)` statement using prepared values.
5. Execute the call and stream the first result set using the existing MySQL field/type conversion path.
6. Drain all remaining results.
7. Return clear errors for missing procedure, wrong arity, and remote execution failures.

### Phase 4: `OUT` / `INOUT` research and optional prototype

1. Confirm how MySQL/MariaDB exposes output parameters and how they are represented in the client API.
2. Evaluate whether native binding is available through the library and whether it fits the current architecture.
3. If warranted, prototype a session-variable approach only for testing and validation.
4. Defer any final support to a later PR after the `IN`-only contract is stable.

### Phase 5: metadata and validation

Optionally query `information_schema.ROUTINES` and `information_schema.PARAMETERS` to validate:

- procedure existence;
- parameter count;
- generic parameter modes;
- declared types;
- overload ambiguity, where supported by the server.

Do not require metadata lookup for the MVP. Runtime MySQL errors remain authoritative, and the extension should prefer correctness and low extra latency over introspection.

### Phase 6: optimizer and serialization support

1. Add `MySQLCatalog::IsMySQLCall` if needed and teach the optimizer how to handle the function's streaming behavior.
2. Decide whether `mysql_call` participates in the same prepared-statement serialization logic as `mysql_query`.
3. Ensure `DETACH` and catalog invalidation do not leave stale prepared calls behind.
4. Add tests covering statement reuse across attached/detached sessions.

### Phase 7: documentation and release hardening

Update `README.md` with:

- basic procedure-call example;
- parameter binding examples;
- result-set behavior and limitations;
- transaction and connection-pinning considerations;
- limitations around multi-result procedures and `OUT`/`INOUT` parameters.

Add integration tests for:

- no-argument procedure;
- `IN` parameters of several types;
- one result set;
- multiple result sets;
- no result set;
- missing procedure;
- wrong argument count;
- remote procedure error;
- connection reuse after success and failure;
- prepared statement reuse and detach behavior.

## 7. Security and correctness requirements

- Never interpolate user parameter values into the SQL string.
- Quote only validated identifiers.
- Pin the connection until all result sets and cleanup operations are complete.
- Drain all remaining result sets before a connection is released to the pool.
- Avoid leaking procedure arguments or sensitive values in logs or error messages.
- Verify behavior under concurrent calls using separate pooled connections.
- Ensure `mysql_call` does not break existing `mysql_query` or `mysql_execute` behavior.
- Keep the feature limited to the extension's shape and semantics rather than creating a vague, cross-layer procedure contract.

## 8. Recommended MVP

The first pull request should implement:

1. safe multi-result consumption and connection draining;
2. an extension-native `mysql_call(...)` function;
3. prepared `IN` parameter binding;
4. first-result-set streaming;
5. rowcount/status output when the procedure does not return a row set;
6. explicit documentation of the multi-result and non-row-return limitations;
7. integration tests covering lifecycle correctness and connection reuse.

Defer `OUT`/`INOUT`, metadata-driven validation, and native DuckDB `CALL` syntax until the core storage-procedure behavior is proven correct.

## 9. Acceptance criteria

The feature is ready for review when:

- a remote procedure with `IN` parameters can be invoked from DuckDB through the extension API;
- returned rows have correct names, types, NULL handling, and values;
- all remote results are consumed before connection reuse;
- existing `mysql_query` and `mysql_execute` behavior is unchanged;
- failures leave the connection pool in a reusable state;
- the API contract is documented and the limitations are explicit;
- tests cover both MySQL and MariaDB-compatible server behavior where supported.
