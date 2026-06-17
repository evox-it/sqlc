# Proposal: `sqlc.optional` — runtime-omittable WHERE predicates (one function, planner-friendly)

Status: **draft** (evox-it fork)
Sibling primitive to [`sqlc.switch`](./sqlc-switch.md). Built on the `feat/sqlc-switch` line.

## Problem

`sqlc.switch` solved *bounded single-dimension* structural variation (sort order,
one filter shape) by compile-time expansion into one function per branch. It does
**not** solve the much more common reporting/listing shape:

> A query with **N mutually independent optional filters**, any subset of which may
> be supplied at runtime.

Concrete motivating case (afc-web, `report.fund_service.list_detail`): a mandatory
issuer + open-date range, plus **6 independent optional filters** (close-date `>=`,
close-date `<=`, operator ids, payment-mode types, transaction types, ticket-product
ids). The two existing options both fail:

1. **Static guard predicates** — the documented sqlc optional-filter idiom:
   ```sql
   AND (sqlc.narg('close_start')::timestamptz IS NULL OR fs.close_date >= sqlc.narg('close_start'))
   AND (cardinality(sqlc.arg('payment_types')::text[]) = 0 OR fpm.type = ANY(sqlc.arg('payment_types')))
   ```
   These compile to one static query string with **every guard always present**.
   Postgres cannot see selectivity through `param IS NULL OR …` / `cardinality(…)=0 OR …`,
   so estimates degrade as filters are added and the planner flips to nested loops
   over join fan-out. Observed in production: **adding filters made the query slower
   and eventually timed out** — the opposite of the intent. (This is what forced a
   raw hand-built dynamic-WHERE in Go, abandoning sqlc for that handler.)

2. **`sqlc.switch` per filter** — one switch per optional filter, on/off branches,
   expands to the **Cartesian product = 2^N functions** (here 2^6 = 64), and there is
   no dispatcher for the cross-product (the v1/v2 dispatcher keys off a single
   selector). Unusable.

There is no way today to express "include this predicate only when its argument is
present" such that the *omitted* predicate physically disappears from the SQL the
planner sees — which is the only thing that actually fixes the plan.

## Design goals (inherited from `sqlc.switch`, plus one)

- **No SQL injection, ever.** User input never reaches the query string. Only
  author-authored constant fragments do; runtime varies *which* fragments are
  included, never their text.
- **Schema-validated at compile time.** Every fragment parses as real SQL and
  references real columns/args. Bad column = compile error.
- **Planner/index friendly — and this is the whole point.** An omitted filter must
  be *physically absent* from the emitted SQL, not hidden behind an `OR`/`CASE`
  guard. A present filter must emit a clean, sargable predicate.
- **One function, not 2^N.** A single generated method whose body assembles the
  WHERE clause at runtime from a fixed set of pre-validated fragments.
- **Modeled on existing precedent** — both `sqlc.switch` (fragment-as-string-literal,
  AST recognition) and `sqlc.slice` (runtime query-string assembly + dynamic args).

## Syntax

```sql
-- name: ListFundServiceDetail :many
SELECT fs.fund_service_id, ft.amount, ...
FROM afc.fund_service fs
INNER JOIN afc.fund_transaction ft ON ft.fund_service_id = fs.fund_service_id
LEFT JOIN afc.fund_payment_mode fpm ON fpm.fund_payment_mode_id = ft.fund_payment_mode_id
WHERE fs.issuer_code = sqlc.arg('issuer_code')
  AND fs.open_date BETWEEN sqlc.arg('start_date') AND sqlc.arg('end_date')
  AND sqlc.optional('fs.close_date >= sqlc.narg(close_start)')
  AND sqlc.optional('fs.close_date <= sqlc.narg(close_end)')
  AND sqlc.optional('fpm.type = ANY(sqlc.slice(payment_types))')
  AND sqlc.optional('fs.system_user_id = ANY(sqlc.slice(operator_user_ids))')
ORDER BY fs.open_date DESC, fs.fund_service_id DESC;
```

- `sqlc.optional('<fragment>')` sits where a boolean predicate is grammatically legal
  (`WHERE`/`HAVING` conjunct). Like `sqlc.switch`, the argument is a **string literal**
  — a compile-time constant authored in the `.sql` file.
- The fragment contains **exactly one** parameter reference (`sqlc.narg(name)` or
  `sqlc.slice(name)`; bare `sqlc.arg` is rejected — see *Presence*). That parameter
  becomes a field of the generated `Params` struct.
- Semantics: the predicate is included (AND-ed into WHERE) **iff its argument is
  "present"** at runtime; otherwise it is omitted entirely and its bind value is not
  sent.

### Presence (the per-type "is it set?" rule)

`sqlc.optional` requires a *nullable/absence-capable* parameter so "unset" is
representable. Presence is decided in generated Go by the parameter's mapped type:

| Param kind | Go type (pgx) | present when |
|---|---|---|
| `sqlc.narg(x)` scalar | `pgtype.*` / `*T` | non-NULL / non-nil |
| `sqlc.slice(x)` | `[]T` | `len(x) > 0` |

Bare `sqlc.arg(x)` (non-null `T`) is a **compile error** inside `sqlc.optional`: a
plain scalar has no "absent" state, so the author must use `narg`/`slice` and the
intent stays explicit. This mirrors the AGENTS.md rule that optional bool filters
must use `narg`, never `arg`.

## Why this does NOT require checking 2^N branches (the key de-risking insight)

The stated worry was: *"it would require the compiler to check all optional branches,
which is complicated."* It does not — because the optional predicates are
**independent conjuncts** (`AND`-ed), not alternatives.

- Validity is monotonic under conjunction removal: if `WHERE A AND B AND C AND D`
  parses and every column/arg resolves, then **any subset** (`WHERE A AND C`, …, even
  `WHERE A`) also parses and resolves. Dropping `AND`-conjuncts never invalidates the
  remainder.
- Therefore the compiler validates **exactly one** form — the *all-present*
  expansion — by splicing every fragment in (reusing `sqlc.switch`'s splice-and-
  reparse machinery, `expand_switch.go:matchParen`/text-splice). One parse, one
  analyze pass. If all-present type-checks, all 2^N runtime combinations are sound.
- Column/arg resolution needs the full FROM/JOIN context (e.g. `fpm.type` needs the
  `fund_payment_mode` join). The all-present splice already carries it. So validation
  cost is **O(number of fragments)** for span-finding and **O(1)** parse/analyze
  passes — not O(2^N).

The genuinely new work is **codegen**, not validation: emit one function that
assembles the WHERE and renumbers placeholders at runtime.

## Codegen strategy

Unlike `sqlc.switch` (expand-before-parse → N ordinary queries), `sqlc.optional`
must keep the fragments through analysis (so params/columns type-check against the
all-present form) and emit **one** function whose body builds the SQL at runtime.
The infrastructure already exists for `sqlc.slice`:

> `internal/codegen/golang/templates/stdlib/queryCode.tmpl:127-165` already emits
> `query := <const>; var queryParams []interface{}; …conditional strings.Replace…;
> q.db.Query(ctx, query, queryParams...)`. That is precisely runtime query-string
> assembly + a dynamically-built args slice.

`sqlc.optional` generalizes that pattern.

### Compile-time emission (sentinel in the const)

In `rewrite.NamedParameters` (`internal/sql/rewrite/parameters.go`) recognize
`sqlc.optional` (a `FuncCall` with schema `sqlc`, exactly as `slice`/`switch` are
recognized via `astutils.Search`). For each optional, replace its text span with a
uniquely-keyed sentinel comment and record the fragment + its single param + presence
kind on the codegen `Query` (a new `OptionalPredicates []OptionalPred` field on
`internal/codegen/golang/query.go:Query`, populated in `result.go`). The emitted
constant keeps the mandatory predicates with their normal compile-time `$N`, and
carries one sentinel per optional, e.g.:

```sql
const listFundServiceDetail = `SELECT ... WHERE fs.issuer_code = $1
  AND fs.open_date BETWEEN $2 AND $3
  /*OPTIONAL:close_start*/ /*OPTIONAL:close_end*/ /*OPTIONAL:payment_types*/ ...
ORDER BY fs.open_date DESC, fs.fund_service_id DESC`
```

### Runtime emission (one method, conditional assembly + `$N` renumbering)

The generated method body (new template block, gated on `.Arg.HasOptionalPredicates`):

```go
func (q *Queries) ListFundServiceDetail(ctx context.Context, arg ListFundServiceDetailParams) ([]ListFundServiceDetailRow, error) {
    query := listFundServiceDetail
    args := []interface{}{arg.IssuerCode, arg.StartDate, arg.EndDate} // fixed prefix: $1..$3
    n := 3                                                            // next placeholder

    if arg.CloseStart != nil {                                        // presence: narg → non-nil
        n++
        args = append(args, arg.CloseStart)
        query = strings.Replace(query, "/*OPTIONAL:close_start*/",
            fmt.Sprintf("AND fs.close_date >= $%d", n), 1)
    } else {
        query = strings.Replace(query, "/*OPTIONAL:close_start*/", "", 1)
    }
    if len(arg.PaymentTypes) > 0 {                                    // presence: slice → len>0
        n++
        args = append(args, arg.PaymentTypes)
        query = strings.Replace(query, "/*OPTIONAL:payment_types*/",
            fmt.Sprintf("AND fpm.type = ANY($%d)", n), 1)
    } else {
        query = strings.Replace(query, "/*OPTIONAL:payment_types*/", "", 1)
    }
    // … one block per optional, in declaration order …

    rows, err := q.db.Query(ctx, query, args...)
    // identical scan loop to the stock :many template
}
```

The fragment's own `sqlc.narg/slice(name)` reference is rewritten to a `$%d`
placeholder **template** at compile time (not a fixed number); `n` is incremented in
declaration order so the contiguous-args requirement pgx imposes is always satisfied.
Mandatory predicates keep their leading fixed `$1..$k`; optionals consume `$k+1…`
in source order. This is the same renumbering `sqlc.slice` already does for `?`
expansion, lifted to `$N` and made conditional.

### Driver coverage

- **pgx / stdlib (postgres)**: `$N` renumbering as above. NB pgx currently has **no**
  runtime-assembly path at all (`templates/pgx/queryCode.tmpl` has no `SLICE`
  handling because pgx passes arrays natively) — this proposal introduces the first
  one for pgx. The stdlib slice path is the working reference.
- **MySQL / SQLite**: `?` placeholders are positional-by-occurrence, so no numbering
  is needed — just include/omit the fragment and append/skip the arg, exactly like
  the existing slice sentinel.

### What stays static / safe

- Fragments are file constants, re-parsed and column-checked at compile time (all-
  present form). Unknown column → `sqlc generate` fails, identical guarantee to
  `sqlc.switch`.
- Runtime varies only: (a) whether a fragment's text is spliced in, and (b) whether
  its already-typed bind value is appended. User input never becomes SQL text.
  Injection is structurally impossible.

## Scope (v1)

- Recognized **only in `WHERE`/`HAVING` conjuncts** (boolean position). Rejected in
  SELECT projection (could change result shape) and in `ORDER BY` (that is
  `sqlc.switch`'s job).
- Each `sqlc.optional` fragment references **exactly one** `narg`/`slice` param.
  (Multi-arg fragments — e.g. a BETWEEN needing two — are expressed as two optionals,
  or deferred to v2.)
- Fragments must be a single self-contained predicate; the generator always joins
  them with `AND`. Leading `AND` is emitted by the generator, so fragments are
  written bare (`fs.close_date >= sqlc.narg(close_start)`).
- One `Params` struct + one `Row` struct (no per-branch duplication — that is the
  advantage over `sqlc.switch`).

## Open questions

1. **`OR`-grouping.** v1 only AND-joins optionals. Do we need optional groups that
   OR together? Probably a v2 `sqlc.optionalGroup(...)`.
2. **Presence override.** Allow an explicit presence expression
   (`sqlc.optional(frag, when => '...')`) for cases where "non-nil/len>0" is wrong
   (e.g. an empty string that *should* filter)? Default rule covers the 95% case.
3. **Empty WHERE.** If every predicate including mandatory ones were optional and all
   omitted, the generator must drop the `WHERE` keyword itself. v1 sidesteps this by
   requiring ≥1 mandatory predicate (the issuer/RLS predicate always qualifies in
   this codebase).
4. **Prepared-statement cache.** Each distinct present/absent combination yields a
   distinct query string → a distinct pgx prepared statement (bounded by 2^N, but in
   practice only the handful of combinations actually used). This is *desirable* — it
   is exactly why the plan is good (clean per-combination predicates) — but worth
   documenting so users understand the statement-cache footprint.

## Relationship to `sqlc.switch`

| | `sqlc.switch` | `sqlc.optional` |
|---|---|---|
| Varies | one bounded dimension (sort / one filter shape) | N independent optional filters |
| Expansion | compile-time, **N functions** (one per key) | runtime, **1 function** |
| Mechanism | splice-before-parse (`expand_switch.go`) | sentinel-in-const + runtime assembly (`sqlc.slice` model) |
| Result structs | shared via `SwitchGroup` | single, by construction |
| Combinatorial cost | linear in keys | none (1 fn regardless of N) |
| Planner benefit | static clean `ORDER BY col` | omitted filters physically absent |

The two are complementary: a report can use `sqlc.switch` for `ORDER BY @sort` and
`sqlc.optional` for its filter set in the same query.
