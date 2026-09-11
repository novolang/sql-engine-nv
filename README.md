# sql-engine-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package
works; calling it panics with `not implemented`.

## What this is

A SQL engine that never reads a file.  It lexes, parses, plans and
executes; when it needs a page it says so and waits for the caller to
hand one over.  Four modules, in the order the text moves through
them: `sqltok` (text to tokens), `sqlast` (tokens to a tree),
`sqlplan` (a tree to an operator plan) and `sqlengine` (a plan to
rows).

For an end user that means the SQL engine, the storage format and the
file are three separable things.  The same engine answers a query over
a file on disk, over a database mapped into a browser's memory, and
over a fixture a test built by hand — because the only thing that
differs between those is whoever answers `PrNeedPage`.

## The one example that will work

```novo
use std.list
use btree
use btcell
use sqlengine

// Run one statement to completion against a page store the caller
// already has.  `read_page` is the host's; nothing else here is.
fn query(schema: sqlengine.Schema, sql: Str) -> [[btcell.Cell]] [fs]
    var e = sqlengine.open(schema)
    var rows: [[btcell.Cell]] = []
    match sqlengine.prepare(e, sql)
        Err(x) => rows
        Ok(start) =>
            var st = start
            var going = true
            while going
                let p = sqlengine.step(e, st)
                e = p.engine
                st = p.statement
                match p.result
                    SrRow(r)   => rows = list.push(rows, r)
                    SrDone(n)  => going = false
                    SrError(x) => going = false
                    SrPage(q)  =>
                        match q
                            PrNeedPage(id)    => e = sqlengine.feed_page(e, id, read_page(id))
                            PrWritePage(i, b) => write_page(i, b)
                            PrAllocPage(k)    => e = sqlengine.feed_alloc(e, alloc_page(k))
                            PrFreePage(i)     => free_page(i)
            rows
```

The `[fs]` on that function is the caller's, from its own `read_page`.
Nothing in sql-engine-nv contributed to it.

## The layer, and why

`core`.  The budget is `[]`, and there are two places that costs
something visible in the surface — both worth reading before adding a
body.

**The catalog arrives as an argument.**  `open` takes a `Schema`
rather than reading one, because reading a catalog is a page walk and
a page walk is the host's.  `pager-nv` reconstructs it from the file
and hands it over.

**The clock arrives as an argument.**  novodb rewrites `DATE('now')`
and its family into literals, and reaches for `time.now()` to do it
inside an `unsafe_discharge [time]` block whose own comment says the
discharge is what keeps its executor effect-free.  A `core` package
may not: `no-discharge-in-core` is a shard row, and a discharge is a
lie about a budget rather than a way to meet one.  So the substitution
stayed and the clock left — `substitute_now(toks, snapshot)` takes the
three formatted strings, and `pager-nv.now_snapshot` is the `[time]`
function that makes one.

## The load-bearing interface

```novo
pub enum StepResult
    SrRow(row: [btcell.Cell])
    SrDone(rows_affected: Int)
    SrPage(request: btree.PageRequest)
    SrError(err: sqlast.SqlError)
```

Five calls are the whole package from a host's side — `open`,
`prepare`, `step`, `feed_page`, `feed_alloc` — and `StepResult` is
what `step` answers.

**`SrPage` carries btree-nv's `PageRequest`, and this package does not
declare one of its own.**  Both packages are `core`, so the layer rule
permitted either direction, and the choice was made on what the type
means.  It means *the tree ran out of pages*: the engine has no page
of its own to want, and every request that leaves `step` came out of a
`btree.Cursor` the engine is holding.  A type that flows downward
through a relay belongs at the bottom of the relay.  btree-nv's README
carries the other half of this argument, and the consequence both
state: the must-have plan has this package at **P0** and btree-nv at
**P1**, and the dependency inverts that for the implementation order.

The second reason is arithmetic.  A read is not the only thing a
statement needs: an INSERT that splits a leaf needs a page ALLOCATED
and a DELETE that empties one needs it FREED.  A `NeedPage`/`WritePage`
pair inline in `StepResult` would have to grow two more arms, and they
would be btree-nv's arms spelled a second time.

## The reference implementation

SQLite's parser, planner and VM as the design, and **novodb's `lex`,
`parse`, `plan` and `exec` as the code** — 28,000 of its 46,000 lines.
The dialect is what novodb accepts: `Stmt` in `sqlast` is the
authoritative statement list and `Expr` the expression list, and where
SQLite and the SQL standard disagree (bitwise-versus-comparison
precedence, INTERSECT precedence) this follows SQLite.

Four things did not come across unchanged.

| novodb | here | why |
| --- | --- | --- |
| `lex.Tok` is ~130 nullary variants with an `is_tk_<word>` predicate each | `Token { kind, text, offset }` and `keyword()` | novodb's own parser never matches the enum and says so at the top of the file; publishing it would make every new keyword a breaking change for anyone who did |
| a view body, a CTE body, a scalar subquery and a rich `IN (SELECT …)` are RAW SQL TEXT, re-parsed on every read | `Select` holds `?Select` and `[Select]` | novodb's reason is a declaration-order limitation in its parser file, not a design choice — its comments say so |
| `parse.Value` and `btree.Cell` are the same five storage classes twice, converted at every INSERT | one `btcell.Cell`, btree-nv's | across a package boundary the duplication would put the bridge on the public surface |
| `lowering_supported` gates the plan path; everything it refuses falls back to a direct AST walk | every `Select` lowers | two executors with different behaviour and no way for a caller to know which ran is not a thing to publish |

The last one is the largest piece of work the implementation lane
inherits, and it is a decision made here rather than discovered there.
The forms novodb falls back on are `FROM (subquery)`, set operations,
CTEs, window functions, INSTEAD OF triggers, scalar subqueries in
expressions, and views in FROM; `tests/sqlplan_tests.nv` asserts each
of them lowers.

**Two gaps in the dialect are inherited on purpose and stated rather
than left to be found.**  SQL comments are not lexed — neither `-- …`
nor `/* … */`, and a `-` followed by a `-` reads as two minus
operators — and double-quoted or backtick-quoted identifiers are not
lexed either, so a column named `"select"` cannot be spelled.  Both
are asserted in `tests/sqltok_tests.nv`, and when the bodies close
them those are the tests that change.

## Building it, and checking it

```bash
novo pkg add sql-engine-nv    # add it to a package
novo pkg build                # type-check and effect-check every module
novo test tests/sqltok_tests.nv
```

**`novo test` is red on every suite, and that is the published state.**
Each test calls a function whose body is `todo()`, so the first
assertion in each file panics:

```
$ novo test tests/sqlast_tests.nv
  ✗ test_a_select_parses_to_a_select_statement
      not implemented: sql-engine-nv.sqlast.parse
  0 passed, 1 failed
```

The tests are the design under review, not a regression net.  When the
bodies land they become the first real assertions, unchanged.

## Status

| function | implemented |
| --- | --- |
| `sqltok.lex`, `keyword`, `is_keyword`, `keywords`, `position`, `show` | no |
| `sqlast.parse`, `parse_tokens`, `affinity_of`, `builtins` | no |
| `sqlast.show_stmt`, `show_expr`, `show_select`, `columns_of` | no |
| `sqlplan.lower`, `optimise`, `optimise_rules`, `reorder_joins` | no |
| `sqlplan.explain`, `estimate_rows`, `tables_of`, `output_columns` | no |
| `sqlplan.fold_constants`, `const_value` | no |
| `sqlengine.open`, `prepare`, `step`, `feed_page`, `feed_alloc` | no |
| `sqlengine.schema_new`, `schema_with_table`, `schema_table`, `schema_of` | no |
| `sqlengine.columns`, `awaiting`, `reset`, `affected`, `plan_of` | no |
| `sqlengine.substitute_now`, `evaluate`, `compare`, `apply_affinity` | no |
