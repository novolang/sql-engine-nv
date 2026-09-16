# sql-engine-nv

SQL is the query language relational databases are asked questions in.
This package is a SQL engine: it turns the text of a statement into
tokens, then into a syntax tree, then into a plan, and then runs the
plan to produce rows. It never opens a file. When it needs a page of a
table it says so and waits for the caller to hand one over. The dialect
is [SQL as understood by SQLite](https://www.sqlite.org/lang.html), and
the tree it reads its rows out of is
[btree-nv](https://novo-lang.org/packages/btree-nv)'s.
[pager-nv](https://novo-lang.org/packages/pager-nv) is what answers the
requests it makes.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A SQL statement becomes rows in four passes, and each pass is one
module here.

**Lexing** turns the text into **tokens**: a keyword, an identifier, a
number, a string, a blob, or a piece of punctuation. Each token carries
the text it covered and the byte offset it started at, so an error a
user sees can name the position in the SQL they typed.

**Parsing** turns the tokens into a **syntax tree**. The tree's shape is
the dialect: `Stmt` is the list of statement forms this engine accepts
and `Expr` the list of expression forms, and anything outside those two
lists is refused. The parser **recovers**, which means it returns a
statement and a list of everything that was wrong with it, rather than
stopping at the first mistake.

**Planning** turns a SELECT into a **plan**, an operator tree that says
how the answer is computed rather than what it is. The operators are
applied in SQL's own evaluation order.

| Order | Operator |
| --- | --- |
| 1 | Scan a table, or a materialised sub-select |
| 2 | Join |
| 3 | Filter — the WHERE clause |
| 4 | Aggregate — GROUP BY and HAVING |
| 5 | Sort — ORDER BY |
| 6 | Distinct |
| 7 | Limit and offset |
| 8 | Project — the select list |

Project sits at the top so that ORDER BY and DISTINCT see the columns
as they were before the select list narrowed them. That is what SQL
means by ordering on a column the select list did not ask for.

**Execution** runs the plan. It is a **cursor**: the caller advances a
prepared statement one step at a time, and a step answers a row, or
says the statement is finished, or says it failed, or asks for a page
and waits. The caller performs the read and hands the bytes back with
`feed_page`.

**This package performs no input or output**, and the compiler checks
that on every build. Two things follow from it that a caller sees in
the signatures.

**The schema arrives as an argument.** `open` takes a `Schema` — the
tables, their columns, their constraints and the page each table's tree
is rooted at — rather than reading one. Reading a catalogue is a walk
over pages, and walking pages is the caller's.

**The clock arrives as an argument.** SQL's `DATE('now')` and its
family are rewritten into literals before the statement is parsed, and
reading a clock is something a package with no effects may not do. The
caller reads its own clock, formats three strings into a `NowSnapshot`,
and passes it to `substitute_now`. `pager-nv`'s `now_snapshot` is a
function that makes one. A snapshot whose strings are empty gives NULL
from every `now` call, which is a defined answer and not a crash.

The consequence for an end user is that the engine, the storage format
and the file are three separable things. The same engine answers a
query over a file on disk, over a database held in memory, and over a
fixture a test built by hand. The only difference between those three
is whoever answers the page requests.

A value in a row is a **cell**, and a cell is one of five things: an
integer, a float, text, a blob, or NULL. Those are the five storage
classes SQLite names, they are btree-nv's `Cell`, and there is no
second value type for a literal.

## Install

```
novo pkg add sql-engine-nv
```

## Example

```novo
use btree
use nodefmt
use sqlengine

// Where the pages come from. A real program reads them out of a
// database file; this one hands back a single empty leaf page.
fn read_page(page_id: Int) -> [Int]
    nodefmt.encode_header(nodefmt.KIND_LEAF, page_id)

fn main() [io]
    // An engine over an empty schema. A pager reconstructs the real
    // one from a file's catalogue and passes it here.
    var e = sqlengine.open(sqlengine.schema_new())

    match sqlengine.prepare(e, "SELECT n FROM t")
        // Syntax, an unknown table, an unknown column: everything that
        // can be decided without reading a row.
        Err(x) => println(x.message())
        Ok(first) =>
            var st = first
            var rows = 0
            var going = true
            while going
                // One advance. The engine and the statement come back
                // beside the step they just took.
                let a = sqlengine.step(e, st)
                e = a.engine
                st = a.statement
                match a.result
                    SrRow(row) => rows = rows + 1
                    SrDone(n)  => going = false
                    SrError(x) => going = false
                    SrPage(q)  =>
                        match q
                            // The engine wants a page. Read it and
                            // hand it back.
                            PrNeedPage(id)    => e = sqlengine.feed_page(e, id, read_page(id))
                            // A SELECT never asks for these three.
                            PrWritePage(i, b) => going = false
                            PrAllocPage(k)    => going = false
                            PrFreePage(i)     => going = false
            println("${rows} rows")
```

That program declares `[io]` for its own `println` and nothing else.
Reading a page from a real file is where an `[fs]` would come from, and
it would be the caller's, not this package's.

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: sql-engine-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `sqltok` | Text to tokens: the token kinds, the reserved-word list this dialect recognises, and the line and column of a byte offset. |
| `sqlast` | The dialect as types: the statement forms, the expression forms, the recovering parser, the printer that renders a tree back to SQL, and the error type all three later passes raise. |
| `sqlplan` | The operator tree: the plan nodes, the lowering from a SELECT, the rewrites, the row-count estimate, and the rendering `EXPLAIN` returns. |
| `sqlengine` | The engine: the schema types, prepare, step, the two functions that feed an answer back, and the expression evaluator. |

## How to choose an entry point

**`sqlengine` is the whole package from a host's side**, and it is five
calls: `open`, `prepare`, `step`, `feed_page` and `feed_alloc`. Use it
to run SQL.

**`sqltok` and `sqlast` on their own read SQL without running it.** A
formatter, a linter, a migration checker or an editor's completion list
needs the tokens and the tree and no pages at all. `sqltok.keywords`
and `sqlast.builtins` are the dialect's reserved words and its scalar
functions, published for exactly that.

**`sqlplan` on its own shows a user why a query is slow.**
`sqlplan.lower` and `sqlplan.optimise` produce the plan,
`sqlplan.explain` renders it, and `sqlplan.estimate_rows` costs it
against statistics the caller supplies.

**`sqlengine.evaluate` runs one expression over one row the caller
already holds.** It is what a CHECK constraint, a DEFAULT, a trigger's
WHEN clause and a partial index all need, without building a plan.
`sqlplan.fold_constants` and `sqlplan.const_value` are the same need
for an expression that names no column.

## The rules a user needs

1. **The dialect is SQLite's, and `Stmt` is the whole of it.** A form
   that is not an arm of `Stmt` is refused with `SeUnsupported`, which
   is a different answer from `SeSyntax`: the first says this engine
   has not built that yet, the second says the text is wrong.
2. **Where SQLite and the SQL standard disagree, this follows
   SQLite.** The four bitwise operators `&`, `|`, `<<` and `>>` bind
   tighter than the comparison operators and looser than `+` and `-`.
   `UNION`, `INTERSECT` and `EXCEPT` are evaluated flat and left to
   right, where the standard gives `INTERSECT` the higher precedence.
   The binding order, loosest first, is: `OR`, `AND`, `NOT`,
   comparison and `IS` and `LIKE` and `GLOB` and `BETWEEN` and `IN`,
   bitwise, `+` and `-`, `*` and `/` and `%`, unary `-` and `~`, then
   a primary expression.
3. **SQL comments are not recognised.** Neither `-- …` nor `/* … */`.
   A `-` followed by a `-` lexes as two minus operators. A dump that
   carries a comment is the first file most people try, so this is
   stated rather than left to be found.
4. **Double-quoted and backtick-quoted identifiers are not
   recognised.** An identifier is a run of ASCII letters, digits and
   underscores whose first character is not a digit, so a column named
   `"select"` cannot be spelled.
5. **`prepare` raises what needs no data and `step` raises the rest.**
   A syntax error, an unknown table or column, an arity mismatch and an
   unsupported form come out of `prepare`. A constraint violation, a
   type error and a division by zero need a row and come out of `step`.
6. **`step` answers one of four things.** `SrRow` is a row. `SrDone`
   carries the rows produced by a SELECT or changed by anything else.
   `SrError` is a failure, and a statement that has failed repeats that
   failure on every later step. `SrPage` is the engine asking, and the
   caller answers it with `feed_page` or `feed_alloc` and steps again.
7. **`feed_page` takes a whole page, header included.** A caller may
   feed a page the engine has not asked for — a prefetch — and the
   engine uses it when it gets there. A page it never wants is dropped.
8. **A byte offset becomes a line and a column with
   `sqltok.position`.** `SeSyntax` and `SeUnsupported` carry the
   offset; turning it into a position needs the source text, which the
   error does not hold.
9. **Every `Select` lowers to a plan.** There is no gate and no
   fallback to a different executor, so `FROM (SELECT …)`, set
   operations, common table expressions including recursive ones,
   window functions, scalar subqueries in expression position and
   views in FROM all produce a `Plan` like everything else.
10. **Indexes are recorded and never consulted.** `CREATE INDEX` is
    accepted so that a SQLite dump loads, and the definition is kept in
    the schema, but every table access is a full scan of that table's
    tree.
11. **`PRAGMA` parses and does nothing.** A dump almost always opens
    with a few, and accepting them is what lets the rest of it load.
12. **`BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `RELEASE` and
    `ROLLBACK TO` parse and step to `SrDone(0)`.** What they mean is
    durability, and durability belongs to whatever owns the file.
13. **A foreign key's `ON DELETE` and `ON UPDATE` actions are recorded
    and not performed.** Every action behaves as `NO ACTION` in this
    version.
14. **Affinity is applied on the way in.** `INSERT INTO t(n) VALUES
    ('7')` into an INTEGER column stores the integer 7. A value that
    cannot be coerced is stored as it is, which is also SQLite's rule.
15. **Comparison follows SQLite's storage-class order.** NULL sorts
    before everything, then integers and floats by value, then text,
    then blobs. That order is `btcell.tag`, and `sqlengine.compare`
    answers -1, 0 or 1.
16. **`AUTOINCREMENT` keeps a high-water mark separate from the next
    rowid.** The two differ after a DELETE, and the difference is
    observable: a deleted id is not handed out again.
17. **`sqlplan.explain`'s rendering is a published format.** It is what
    an `EXPLAIN` statement returns, as a single-column result, so a
    tool that parses that output is parsing this.
18. **Nothing here is mutated.** `step` returns an engine and a
    statement beside the result rather than changing either in place,
    so a caller may hold two engines over one database and know they
    cannot alias (SPEC section 14).

## What is not included

- **Any input or output.** No file is opened and no page is read. The
  package's effect row is empty and the compiler enforces it.
- **A page store.** `SrPage` is how the engine asks.
  [pager-nv](https://novo-lang.org/packages/pager-nv) is what answers.
- **Index-based access paths.** See rule 10.
- **Transactions, savepoints and durability.** The engine is not a
  connection and holds no transaction. See rule 12.
- **Foreign-key enforcement.** See rule 13.
- **A clock.** `substitute_now` takes the three strings a host
  formatted. See *What it is*.
- **Reading a SQLite database file.** The pages this engine asks for
  and decodes are btree-nv's page format, not SQLite's on-disk format.
  [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) is the package
  that reads a file another program wrote.
- **SQL comments and quoted identifiers.** See rules 3 and 4. Both are
  meant to close before 0.1.0, and the two tests that assert their
  absence are what will change.

## Related packages

- [btree-nv](https://novo-lang.org/packages/btree-nv) is the ordered
  map the rows are read out of. It publishes `Cell`, the value a row
  holds, and `PageRequest`, which `SrPage` carries outward unchanged —
  every request that leaves `step` came out of a `btree.Cursor` this
  engine is holding.
- [pager-nv](https://novo-lang.org/packages/pager-nv) is the file, its
  write-ahead log and the checksums over it. `driver.execute` there is
  the request loop of the example above, written once.
- [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) reads a real
  SQLite database file, in SQLite's own format, with its own cursors.
  Take that package to open a file some other program wrote. Take this
  one for SQL over a database in novo-lang's format.
- `std.sql` in the standard library declares the `Database` trait a
  program holds when it does not want to name an engine at all.

## Tests

```bash
novo test tests/sqltok_tests.nv     # 12 tests: the tokens and the dialect's words
novo test tests/sqlast_tests.nv     # 14 tests: the tree, the recovery, the round trip
novo test tests/sqlplan_tests.nv    # 12 tests: the lowering and the rewrites
novo test tests/sqlengine_tests.nv  # 19 tests: the step protocol, driven by hand
```

The reference is SQLite: its parser, its planner and its virtual
machine are the design, and where the dialect is a choice the test
names SQLite as the source of the answer. `sqltok_tests.nv` asserts the
two gaps in rules 3 and 4 as they stand, so closing them is a test that
changes rather than a behaviour that drifts. `sqlplan_tests.nv` asserts
that every form lowers and that there is no fallback path.
`sqlengine_tests.nv` is written the way a host writes: a loop that
steps, answers what the engine asks for, and steps again. It checks
that a schema is handed in and never read, that a SELECT asks for its
root page first and writes nothing, that a feed the engine did not ask
for is kept rather than refused, and that an empty clock snapshot gives
NULL rather than a panic.

The tests compile today and fail at run, each on the
`not implemented: sql-engine-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `sqltok.lex`, `.keyword`, `.is_keyword`, `.keywords`, `.position`, `.show` | no |
| `sqlast.parse`, `.parse_tokens`, `.affinity_of`, `.builtins` | no |
| `sqlast.show_stmt`, `.show_expr`, `.show_select`, `.columns_of` | no |
| `sqlast.SqlError.message` | no |
| `sqlplan.lower`, `.optimise`, `.optimise_rules`, `.reorder_joins` | no |
| `sqlplan.explain`, `.estimate_rows`, `.tables_of`, `.output_columns` | no |
| `sqlplan.fold_constants`, `.const_value` | no |
| `sqlengine.open`, `.prepare`, `.step`, `.feed_page`, `.feed_alloc` | no |
| `sqlengine.schema_new`, `.schema_with_table`, `.schema_table`, `.schema_of` | no |
| `sqlengine.columns`, `.awaiting`, `.reset`, `.affected`, `.plan_of` | no |
| `sqlengine.substitute_now`, `.evaluate`, `.compare`, `.apply_affinity` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
