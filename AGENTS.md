# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, ...) when working with code in this repository.

## Project Overview

Linq.J is a zero-dependency Java library that brings LINQ (Language Integrated Query) capabilities to Java, enabling SQL-like querying of in-memory collections via a fluent, chainable API with lambda expressions.

- **Maven coordinates:** `xyz.erupt:linq.j` (version in `pom.xml`, currently `1.0.0`)
- **Requirements:** JDK 8+, zero production dependencies. The compiler plugin sets `source`/`target` 8 (not `release`), so a newer JDK accepts APIs that do not exist on Java 8 — keep main code to Java 8 syntax and APIs (no `var`, although the README examples use it).
- **Test dependencies only:** JUnit 4.13.2, Gson 2.9.1. The test models in `src/test/java/xyz/erupt/linq/data/` write their getters by hand (no Lombok).

## Build & Test Commands

Single-module Maven project, no wrapper — use the system `mvn`.

```bash
mvn clean compile                              # Compile
mvn clean test                                 # Run all tests
mvn clean test -Dtest=LinqTest                 # Run specific test class
mvn clean test -Dtest=LinqTest#selectId        # Run specific test method
mvn clean package                              # Build JAR (also attaches the sources and javadoc jars)
mvn clean install -Dgpg.skip                   # Install locally without signing
mvn clean deploy                               # Publish to Maven Central (needs a GPG key and the `central` server in settings.xml)
```

- `maven-gpg-plugin` signs in the `verify` phase with no profile around it, so a plain `mvn install` / `mvn verify` requires a GPG key.
- `BenchTest` is a benchmark, not a correctness suite: it builds 1,000,000 rows and prints timings, and it runs as part of `mvn test`. Run it alone with a real heap: `mvn test -Dtest=BenchTest -DargLine="-Xmx6g -XX:+UseParallelGC"`.

## Architecture

**Data flow:** `Linq` (entry point) → builds a `Dql` → a terminal method in `Write` hands it to the `Engine` → results.

### Core layers

All under `src/main/java/xyz/erupt/linq/`.

| Package | Role |
|---------|------|
| `Linq` | Entry point (`Linq.from(List)` / `Linq.from(T...)`) and the only implementation of the grammar interfaces |
| `grammar/` | Six SQL-like interfaces: `Select`, `Join`, `Where`, `GroupBy`, `OrderBy`, `Write` (the terminals: `toList`, `one`, `first`, `toMaps`, `toMap`, `count`, `exists`) |
| `schema/` | DQL data structures: `Dql` (full query), `Row`, `Column`, `JoinSchema`, `WhereSchema`, `OrderBySchema` |
| `engine/` | `Engine` (abstract), `EruptEngine` (default), `ParallelEruptEngine` (opt-in) |
| `lambda/` | Lambda parsing: `SFunction` (serializable function), `LambdaSee` (method reference → `LambdaInfo`), `It` (self reference for simple-typed sources) |
| `util/` | `Columns` (column factory and aggregates: sum/avg/min/max/count/countDistinct/groupArray/aggregate), `RowUtil` (object ↔ `Row`), `Accessors` (fast getters/setters), `ReflectField` (cached field lists), `CompareUtil`, `VirtualColumn` |
| `consts/` | Enums: `JoinType`, `OrderByDirection`, `CompareSymbol`, `JoinStrategy` |
| `exception/` | `LinqException` |

### Building the query

- `Linq` implements all six grammar interfaces; every method mutates its `Dql` and returns `this`. Most of the API is `default` methods on the interfaces (`eq`, `gt`, `like`, `leftJoin`, `orderByDesc`, ...) that funnel into a few abstract ones implemented in `Linq`, so a new operator usually belongs in the interface.
- `Dql.getColumns()` is a list whose `add` first removes an equal column, so selecting the same column twice keeps the last one. Before execution the engine also keeps only the last column per alias.
- A typed `where(column, predicate)` stores two forms in `WhereSchema`: a row-level `condition` and a `valueCondition` on the raw column value. `where(Predicate<Row>)` has only the first, which rules out pushdown and the direct path (below).

### Resolving method references to columns

- `LambdaSee.info(SFunction)` invokes the lambda's synthetic `writeReplace` to get its `SerializedLambda`. The class comes from the instantiated method type, the field name from the implementation method name with a leading `get` / `is` stripped and decapitalized. No bytecode is read and no field lookup is done, so the getter must follow that naming convention.
- Results are cached by `SFunction` instance. Only method references (`User::getName`) are supported; an arbitrary lambda body resolves to its synthetic method name, not to a field.
- A `Column` is identified by table class + field + alias. Rows are keyed by the runtime class of the first non-null source element, with one column per declared field including superclass fields (`ReflectField`).
- Simple-typed sources (`RowUtil.SIMPLE_CLASS`: `CharSequence`, `Number`, `Date`, `Temporal`, `Boolean`, `Character` and their arrays) produce a single column addressed by `It::self`. `VirtualColumn` is the placeholder table for columns with no source field (`Columns.count(alias)`, `selectExpr`).

### Execution

`Write.toList` / `toMaps` call `Engine.preprocessor` (no `select` → all fields of the source element; joining the same class twice is rejected), then try the direct path and fall back to the row pipeline.

- **Row pipeline** (`EruptEngine.query`), in this order: where pushdown or `toTable` → join → where → group by or select → having → order by → offset / limit → distinct. Distinct runs after limit.
- **Where pushdown** (`filterAtSource`): with no joins and only typed wheres, predicates run on the source objects before any `Row` is built.
- **Direct path** (`queryDirect` / `queryDirectMap`): a single-table query with no join, group by, having, order by, distinct, aggregate or row converter maps source objects straight to the target type and never builds a `Row`. Returning `null` means "fall back to `query`". Its results must stay identical to the row pipeline; `DirectPathTest` covers this.
- **Join:** hash join only (`JoinStrategy.HASH`). Argument order is `join(target, targetOn, sourceOn)`.
- **Group by:** a selected column without an aggregate function takes its value from the first row of the group. Aggregates are `Column.groupByFun`; an aggregate column without `groupBy` collapses the result to one row.
- **Numbers:** aggregates produce `BigDecimal`, which `RowUtil.bigDecimalConvert` narrows to the target field type on output.
- **Nulls:** null source elements are skipped and null field values are not stored in the `Row`. In `orderBy` a null on either side compares as equal.
- **`Row`** is an array-backed `AbstractMap<Column, Object>` with linear lookup; `putDirect` skips the duplicate check and is only safe when the column is known to be absent.
- **Object access:** `Accessors` builds `LambdaMetafactory` getters/setters for public accessors on public classes (`getX`, `isX`, record-style `x()`, `setX`), cached per class; everything else falls back to field reflection. Target classes need a no-arg constructor; columns map to target fields by alias.

### Engines

- `Linq.engine()` returns a new `EruptEngine` unless one was set with `setEngine(Engine)` or `parallel()` / `parallel(threshold)`.
- A custom engine extends `Engine` and implements `query(Dql)`; `queryDirect` / `queryDirectMap` are optional and default to `null`.
- `ParallelEruptEngine` overrides only the `toTable` and `mapSource` hooks of `EruptEngine`, so only source materialization runs in parallel (common ForkJoinPool, 50,000-element default threshold, encounter order preserved). Join, group and sort stay sequential. It is never the default.

### Planned features (not yet implemented)

- UNION ALL / UNION / INTERSECT / EXCEPT / UNION BY NAME
- Window functions

## Documentation

`README.md` (English) and `README-zh.md` (Chinese) describe the public API. `docs/` is the static project website.
