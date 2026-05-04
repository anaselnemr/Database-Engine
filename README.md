# Database Engine — Java Relational Engine with Grid Index

A from-scratch Java relational database engine built for the **Databases II (CSEN 604)** course at the **German University in Cairo (GUC)**. The engine implements page-based row storage on top of Java serialization, a **multi-dimensional grid (cell-bucket) secondary index** for fast range and equality queries, full CRUD with binary-search-driven page placement, a `WHERE`-clause query evaluator that respects `AND` / `OR` / `XOR` precedence, and a **plain-text SQL parser** (built on JSQLParser 4.0) that turns `CREATE TABLE`, `CREATE INDEX`, `INSERT`, `UPDATE`, `DELETE`, and `SELECT` statements into engine API calls.

## Features

- **Page-based table storage.** Each table is a sequence of pages (default 250 rows per page, configurable via `DBApp.config`). Pages and table metadata are persisted with Java's built-in serialization (`ObjectOutputStream`) under `src/main/resources/data/`.
- **Clustering key + per-column min/max constraints.** Every table declares a clustering (primary) key and per-column type + min/max bounds at creation time; inserts that violate type or range bounds are rejected.
- **Sorted inserts via binary search over pages.** `BinarySearchForPage` + `BinarySearchForRow[Insert]` locate the correct page and slot in O(log n) over per-page min/max metadata; full pages cascade an overflow into the next page (or a new last page).
- **Multi-dimensional grid index** (`GridIndex` + `CArray` + `GridCell` + `Bucket`). Every indexed column is partitioned into 10 ranges (works for `Integer`, `Double`, `String`, and `java.util.Date` — strings use a lexicographic-range counter in `stringRanges.java`); the cartesian product of ranges forms a sparse N-dimensional grid where each occupied cell holds a chain of overflow buckets, and each bucket holds `<pkValue, pageNumber>` addresses up to a configurable maximum.
- **Index-aware query planner.** `selectFromTable` first asks the table for the smallest matching index; `AND` / `OR` / `XOR` are evaluated as set operations directly over grid cells when fully indexed, otherwise the engine falls back to a per-page **linear scan** for the unindexed terms.
- **Full CRUD with index maintenance.** `insertIntoTable`, `updateTable`, `deleteFromTable` keep every matching grid index in sync via add/remove/update hooks plumbed through `PageActionListener` → `RowsVector` → `Page` → `Table.indiciesUpdateRow`.
- **Plain-text SQL.** `parseSQL(StringBuffer)` accepts standard SQL strings (parsed by JSQLParser) and dispatches to the matching engine method, including arbitrarily nested `WHERE` expressions with AND/OR.
- **Pretty-printed table output** via the [prettytable](https://github.com/skebir/prettytable) library — `Table.toString()` renders every page as an aligned ASCII table for debugging.
- **Persistent metadata.** `src/main/resources/metadata.csv` is appended on every `CREATE TABLE` / `CREATE INDEX` and re-read on every mutating call (`updateConstraintsUsingMetadata`) so external edits to the metadata file flow back into in-memory schema.

## SQL / API surface

### Programmatic API (`DBAppInterface`)

```java
void init();                                                            // create data dir + metadata.csv
void createTable(String tableName, String clusteringKey,
                 Hashtable<String,String> colNameType,
                 Hashtable<String,String> colNameMin,
                 Hashtable<String,String> colNameMax);
void createIndex(String tableName, String[] columnNames);               // builds a grid index over the listed columns
void insertIntoTable(String tableName, Hashtable<String,Object> colNameValue);
void updateTable(String tableName, String clusteringKeyValue,
                 Hashtable<String,Object> columnNameValue);             // matched by clustering key
void deleteFromTable(String tableName, Hashtable<String,Object> columnNameValue);
Iterator selectFromTable(SQLTerm[] sqlTerms, String[] arrayOperators);  // returns Iterator<Row>
```

`SQLTerm` is the `<table, column, op, value>` quadruple consumed by `selectFromTable`. The supported comparison operators are `=`, `!=`, `>`, `<`, `>=`, `<=`. The `arrayOperators` array carries the in-order logical connectives between successive terms — accepted values are `"AND"`, `"OR"`, `"XOR"` (length must be `sqlTerms.length - 1`).

Supported column types are `java.lang.Integer`, `java.lang.Double`, `java.lang.String`, and `java.util.Date` (date strings parsed as `yyyy-MM-dd`).

### Plain-text SQL (`DBApp.parseSQL(StringBuffer)`)

Accepts any of the following statement shapes (parsed by JSQLParser, then dispatched to the engine API):

```sql
CREATE TABLE sami (
    gpa decimal(5,2),
    student_id varchar(25),
    PRIMARY KEY (gpa)
);

CREATE INDEX idx ON transcripts (gpa, student_id);

INSERT INTO transcripts(gpa, student_id) VALUES (4.5, '63-4499');

SELECT * FROM transcripts WHERE gpa > 4.0 AND student_id > '45-0000';

UPDATE transcripts SET student_id = '99-9999' WHERE gpa = 3.1964;

DELETE FROM transcripts WHERE gpa = 3.1964 AND course_name = 'iQrcnv';
```

`CREATE TABLE` infers per-column min/max defaults by SQL type (`varchar` → `AAAAAA`/`zzzzzz`, `int` → `0`/`1000000`, `double|float|decimal` → `0.0`/`1000000.0`, `date|datetime` → `1990-01-01`/`2030-01-01`). For tighter bounds, call `createTable(...)` directly with explicit min/max hashtables. `DELETE` only accepts AND-connected `=` predicates (mirroring the engine API). `SELECT` respects `AND` / `OR` precedence parsed into a left-deep expression tree, and operator precedence is honored when evaluating mixed clauses.

## Architecture

```
DBApp  ──────────────────────────────────────────────────────────────
  │   parseSQL(StringBuffer)                          ── JSQLParser
  │   selectFromTable / insert / update / delete      ── public API
  ▼
Table  ── columns, pages dictionary, grid indices, metadata I/O
  │
  ├── Page (250 rows max)  ── min/max pkey cache, binary search slot
  │     └── RowsVector  ── add/remove/clear hooks → PageActionListener
  │           └── Row (Hashtable<String,Object>)
  │
  └── GridIndex (1+ columns)
        ├── ranges per dimension (10 buckets per column, type-aware)
        ├── CArray (sparse N-dim Vector)
        │     └── GridCell (one per occupied range tuple)
        │           └── Bucket (chain on overflow)
        │                 └── Address <table, pkValue, pageNumber>
        └── queryIndex(Query) → AND/OR/XOR over GridCell sets

helper.java  ── readConfig, serialize/deserialize, parseStringButRespectType,
                compareObjects (Number-safe), CSV metadata I/O
stringRanges ── lexicographic range counter for String-typed grid columns
```

**Insert flow.** `DBApp.insertIntoTable` first asks every matching grid index for a candidate page (one round-trip through `evaluateIndexQuery` → bucket scan), falling back to `BinarySearchForPageInsert` on the table's pages. If the target page is full, the last row spills into the next page; if no next page exists, a new tail page is appended. Every page-level mutation triggers `Table.indiciesAddRow` / `indiciesRemoveRow` / `indiciesUpdateRow`, which walks `Table.indices` and patches every affected grid cell + bucket.

**Select flow.** `selectFromTable` builds a `Query` from the input terms, picks the index with the **lowest count of unindexed columns** (`Table.searchingIndex`), then either delegates the full query to `GridIndex.queryIndex` (when every column is covered) or recursively splits the expression on the highest-precedence operator (`evaluateEachSegmentOf`). Single-term subqueries hit the grid (if indexed and op ≠ `!=`) or fall through to `linearQuery` — a per-page sequential scan that evaluates the term row-by-row.

## Tech Stack

- **Language:** Java 11 (`maven.compiler.source`/`target = 11`)
- **Build tool:** Apache Maven (`pom.xml`, group `edu.guc`, artifact `DB2Project`)
- **Persistence:** Java built-in serialization (`ObjectOutputStream` / `ObjectInputStream`) — no external storage engine
- **SQL parsing:** [JSQLParser 4.0](https://github.com/JSQLParser/JSqlParser) (via JitPack)
- **Pretty-printing:** [prettytable 1.0](https://github.com/skebir/prettytable) (via JitPack)
- **Testing:** JUnit Jupiter 5.5.2 + JUnit Platform Runner 1.5.2

## Project Structure

```
src/main/java/
  DBApp.java              — engine entry point + parseSQL + index-aware query planner (1343 LOC)
  DBAppInterface.java     — public API contract
  DBAppException.java     — checked exception type
  Table.java              — table I/O, pages dictionary, binary-search insert/lookup, index registry
  Page.java               — per-page row container, min/max cache, validation hooks
  Row.java                — Hashtable<String,Object> with action listener
  RowsVector.java         — Vector<Row> with add/remove hooks → page listener
  Column.java             — column metadata (name, type, isPK, indexed, min, max)
  PageActionListener.java — page-level row-mutation callbacks
  GridIndex.java          — N-dimensional grid index (range partitioning, AND/OR/XOR over cells)
  CArray.java             — sparse N-dim Vector backing the grid
  GridCell.java           — one cell per occupied range tuple, owns a bucket chain
  Bucket.java             — Vector<Address> with overflow-on-add / underflow-on-remove
  Address.java            — <table, pkValue, pageNumber> entry stored in buckets
  Query.java              — terms + operators container with AND/OR/XOR sub-query helpers
  SQLTerm.java            — public <table, column, op, value> quadruple
  helper.java             — config, serialization, type-safe compare, metadata I/O
  stringRanges.java       — lexicographic-range counter for String-indexed columns
  test.java               — scratch driver (not a unit test)
src/main/resources/
  DBApp.config            — MaximumRowsCountinPage = 250, MaximumKeysCountinIndexBucket = 100
  metadata.csv            — per-table column schema (auto-maintained)
  {courses,pcs,students,transcripts}_table.csv — sample seed data used by tests
src/test/java/
  Milestone1Tests.java         — public M1 tests (table creation + CRUD + page mechanics)
  Milestone1PrivateTests.java  — extended M1 tests
  Milestone2Tests.java         — public M2 tests (index creation + query routing)
  Milestone2PrivateTests.java  — extended M2 tests
pom.xml                   — Maven build (Java 11, JitPack repo, JSQLParser + prettytable + JUnit 5)
```

## How to Build & Run

The project is a standard Maven module — clone, then:

```bash
# Compile
mvn compile

# Run all unit tests (Milestone 1 + Milestone 2)
mvn test

# Run a specific test class
mvn -Dtest=Milestone1Tests test
mvn -Dtest=Milestone2Tests test

# Package
mvn package
```

The `DBApp.main(...)` method is wired up as a scratch driver — it loads sample tables and exercises queries; uncomment the relevant block before running. Run it from your IDE (Eclipse / IntelliJ — both `.classpath` and `.idea/` are checked in) or directly via the Maven exec plugin.

**Configuration knobs** live in `src/main/resources/DBApp.config`:

```properties
MaximumRowsCountinPage = 250
MaximumKeysCountinIndexBucket = 100
```

`MaximumRowsCountinPage` sets the per-page row cap for both tables AND grid index buckets (the `Bucket` constructor reads the same key). `MaximumKeysCountinIndexBucket` is declared in the config but not currently consumed by the code.

**Sample tables** (`createStudentTable`, `createCoursesTable`, `createTranscriptsTable`, `createPCsTable`) and CSV seed loaders live in the test classes and in `DBApp.createTranscriptsTable` / `insertTranscriptsRecords` — handy starting points for ad-hoc experimentation.

## Coursework Context

Built for **CSEN 604 — Databases II**, **Faculty of Media Engineering and Technology**, **German University in Cairo (GUC)**, taught in the B.Sc. Computer Science & Engineering programme. The repository implements both course milestones:

- **Milestone 1** — table creation, page-based storage, sorted insert with overflow, CRUD, linear-scan select.
- **Milestone 2** — multi-column grid index, index-aware query planner with AND/OR/XOR, plain-text SQL parser (covered as a bonus on top of M2).

The original final commit landed in **January 2022**.

## Authors

Anas ElNemr  ·  Ahmed Eltawel
