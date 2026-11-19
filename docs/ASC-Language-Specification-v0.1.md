# ASC Language Specification v0.1

**Status:** Draft historical reconstruction with confirmed modernizations  
**Language:** ASC (Associative Computing)  
**Version:** 0.1  
**Date:** 2026-11-19

## 1. Purpose and scope

This document specifies version 0.1 of a reconstructed ASC programming language. Its purpose is to preserve the original, single-instruction-stream ASC model of computation while defining a small set of intentional surface-syntax and toolchain modernizations.

The primary historical sources are:

1. *ASC Primer*, Department of Mathematics and Computer Science, Kent State University (undated handbook; its text refers to the language and emulator as they existed around 1990-1992).
2. Jerry Potter, Johnnie Baker, Stephen Scott, Arvind Bansal, Chokchai Leangsuksun, and Chandra Asthagiri, "ASC: An Associative-Computing Paradigm," *Computer*, November 1994, pp. 19-25.

The Primer is the primary source for language constructs. The 1994 paper is the primary source for the abstract computation model and its intended portability across computing engines.

The surviving sources contain typographical inconsistencies, OCR-sensitive punctuation, examples from more than one compiler target, and features whose complete grammar or edge semantics are not stated. This specification therefore uses three confidence labels:

- **Required**: supported directly by the sources or by a confirmed modernization decision, and normative in v0.1.
- **Source-supported, grammar pending**: clearly present in historical ASC, but not yet specified precisely enough for portable implementation.
- **Unresolved**: the available sources do not establish a single safe rule. An implementation must diagnose the construct or document an extension; it must not silently invent historical behavior.

Examples in this document are illustrative unless a production is explicitly labeled normative.

## 2. Design principles

### 2.1 Historical semantic core

ASC is an associative, data-parallel language. The v0.1 semantic target is the simplest historical ASC model: one instruction stream controlling a collection of processing elements (PEs), each with local storage. Related data items form a record, normally one record per PE.

The instruction stream broadcasts an operation to all PEs in the current active set. Each active PE applies that operation to its own local data. Inactive PEs observe the instruction stream but do not perform its data operation.

ASC computation is organized around a nestable cycle:

1. **Search**: evaluate a parallel condition across the current active set and identify responders.
2. **Process**: execute one or more broadcast operations on the responders.
3. **Retrieve**: select or reduce responder data into scalar control or data.

A nested search restricts the current active set. Leaving the nesting construct restores the prior active set.

### 2.2 One language, multiple backends

Execution placement is not part of ASC source syntax. The compiler selects a backend:

```text
asc --target=cpu program.asc
asc --target=cpu-simd program.asc
asc --target=gpu program.asc
```

The three target names and this command-line form are modern compiler requirements, not claims about the exact historical switches.

- `cpu` is the sequential reference implementation of ASC semantics.
- `cpu-simd` may implement PE operations with host vector instructions.
- `gpu` may implement PE operations with GPU work items and masks.

All conforming targets must preserve the same observable ASC behavior. Target selection may change performance, storage layout, and lowering strategy, but must not change source-level meaning. ASC v0.1 has no source-level CPU, SIMD, GPU, device, kernel, thread, or placement statement.

This separation is historically grounded. The Primer describes a common first compiler pass that produced intermediate code and target-specific second passes for an emulator, the Connection Machine, and the Wavetracer. The 1994 paper describes ASC as applicable from conventional sequential computers through SIMD and associative machines.

### 2.3 Confirmed syntax modernizations

The following departures from historical ASC are Required:

1. Keywords are lowercase and case-sensitive.
2. Identifiers are case-sensitive.
3. Comparisons use `==`, `!=`, `<`, `<=`, `>`, and `>=`.
4. Boolean operations use `&&`, `||`, and `!`.
5. Parentheses may group expressions, including nested expressions.
6. Conventional C-style expression precedence is used as specified in Section 7.
7. Declaration modes precede the data type as qualifiers: `parallel int field[$];` and `scalar int value;`.

Historical uppercase keywords and dot-delimited operators are not part of the v0.1 source grammar. A separate conversion tool may support legacy source, but conversion is outside this language specification.

## 3. Abstract execution model

### 3.1 Processing elements and records

A PE contains local data for at most one active ASC record at a time. A parallel field denotes one value per applicable PE; fields associated together form the columns of a logical table, while each PE's local values form one row.

The abstract model assumes enough PEs to hold the program's live records. A concrete backend may virtualize PEs and execute them sequentially or in batches.

### 3.2 Instruction stream and active set

There is one ASC instruction stream in v0.1. At every point in execution it has a current active set of PEs.

A parallel statement is conceptually broadcast once and is performed independently by every PE in the active set. A scalar statement is performed by the scalar control context.

The following are semantic requirements, independent of implementation:

- A backend must represent the current active set without exposing its representation to the program.
- Nested responder scopes must restore the enclosing active set on exit.
- A parallel write affects only PEs active for that statement.
- The result must not depend on the physical order in which a backend batches ordinary data-parallel operations.

### 3.3 Responders and nonresponders

Evaluating a logical parallel expression over active PEs partitions them into:

- **responders**, for which the expression is true; and
- **nonresponders**, for which the expression is false.

PEs outside the incoming active set belong to neither partition for that evaluation.

Responder selection constructs may use one partition, both partitions in sequence, or a scalar any/none test. The precise rules for each confirmed construct appear below.

### 3.4 Selection and broadcast retrieval

Historical ASC can select a responder, retrieve one of its values, and broadcast that value to the instruction stream or other PEs. The Primer's `get`, `next`, index variables, and index-returning reductions expose forms of this behavior.

Where the historical source says "first" or "topmost," v0.1 treats the result as dependent on the association's defined logical row order, not on a backend's scheduling order. Where a source merely requires an arbitrary responder, a portable program must not depend on which tied responder is selected.

### 3.5 Abstract cost versus concrete cost

The historical model treats associative search, global Boolean operations, responder selection, and minimum/maximum identification as constant-time primitives under its machine assumptions. This is an algorithmic property of the ASC model, not a performance guarantee for `cpu`, `cpu-simd`, or `gpu`. A conforming backend may lower one abstract operation to multiple physical instructions.

## 4. Lexical structure

### 4.1 Source files and statement termination

ASC source files conventionally use the `.asc` suffix. Statements shown by the sources terminate with `;`. Block introducers do not consistently carry semicolons in the Primer; the normative productions in this document show where v0.1 requires them.

Whether a trailing semicolon is permitted after every block terminator is **Unresolved** except where explicitly shown.

### 4.2 Keywords and case

Keywords are lowercase only. `if` is a keyword; `IF` and `If` are identifiers if otherwise lexically valid. Identifier lookup is case-sensitive, so `value`, `Value`, and `VALUE` are distinct.

The complete v0.1 reserved-word set is **Source-supported, grammar pending**. At minimum, an implementation must reserve the keywords used by the Required constructs it implements. The historical Appendix D list is evidence for eventual coverage, but it contains scan and print ambiguities and must not be copied blindly into a lexer.

### 4.3 Identifiers

Historical ASC identifiers begin with a letter and are limited to 32 characters. Whether v0.1 retains the 32-character limit and the exact permitted continuation characters is **Unresolved**. Implementations should accept ASCII letters followed by ASCII letters, decimal digits, or `_`, and should not impose a limit below 32 characters. This recommendation is non-normative until the grammar is finalized.

### 4.4 Comments

The Primer specifies C-style comments:

```text
/* comment */
```

It also states that comments may nest. Therefore nested `/* ... */` comments are Required. An unterminated comment is a compile-time error.

### 4.5 Literals

The sources show decimal integer constants and historical hexadecimal, octal, and binary forms in `define` declarations. The exact legacy spelling and width/overflow behavior are **Source-supported, grammar pending**. Character and string literals are used by examples and I/O constructs, but their full escaping rules are **Unresolved**.

## 5. Program structure and declarations

### 5.1 Compilation units

Historical ASC defines main programs and subroutines:

```text
main program_name
    declarations
    statements
end;
```

```text
subroutine subroutine_name
    declarations
    statements
end;
```

The order given by the Primer is constants, equivalences or storage definitions, variable declarations, association declarations, and program body. Main/subroutine structure is **Required**, while the exact declaration-order constraints are **Source-supported, grammar pending**.

### 5.2 Scalar and parallel modes

Every declared data object has a mode:

- `scalar`: stored once in the scalar control context.
- `parallel`: one value per applicable PE.

Historical ASC placed the data type before the mode:

```text
data_type scalar name;
data_type parallel field[$];
```

ASC v0.1 intentionally reverses that order. The mode is a leading declaration qualifier, analogous to a C declaration qualifier:

```text
scalar data_type name;
parallel data_type field[$];
```

For example:

```text
scalar int count;
parallel int value[$];
parallel logical selected[$];
parallel index idx[$];
```

The ordering `data_type scalar` or `data_type parallel` is historical syntax and is not accepted as standard v0.1 syntax. `$` designates the parallel row dimension. Scalar arrays and parallel fields with up to three dimensions appear in the Primer. Their exact v0.1 array and indexing grammar is **Source-supported, grammar pending**.

### 5.3 Historical data types

The Primer lists `int`, `hex`, `oct`, `bin`, `card`, `char`, `logical`, and `index`, and elsewhere mentions `real`. It permits explicit field widths for some parallel values.

The scalar/parallel distinction, `parallel logical`, and `parallel index` are Required semantic categories. The complete type system, conversion rules, widths, signedness, overflow, availability of `real`, and target-independent representation are **Unresolved**. A compiler must not infer these details solely from a host C type.

### 5.4 Index variables

An index variable identifies one responder row so that an associated parallel value can be retrieved or updated using syntax such as `value[idx]`. Index variables connect search, process, and retrieve operations.

Historical descriptions sometimes call an index variable a one-bit parallel field, reflecting a one-hot representation. That representation is not source-observable and is not required of modern backends. The semantic requirement is that an index designates either no row or exactly one row within its valid scope.

### 5.5 Constants

Historical ASC provides `define` for scalar constants and `deflog` for logical constants:

```text
define(name, value);
deflog(name, value);
```

These constructs are **Source-supported, grammar pending** because literal grammar, name scope, and type rules are incomplete in the sources.

### 5.6 Associations

An association groups parallel fields into a record/table under a logical parallel field:

```text
associate tail[$], head[$], weight[$] with graph[$];
```

The associated fields and association field must already be declared. Association membership determines which values belong to the same logical record. Association declaration is Required. Aliasing, overlapping associations, and whether a field may belong to multiple associations are **Unresolved**.

### 5.7 Storage equivalence (`defvar`)

The Primer uses `defvar` to overlap fields or place one at an absolute address. This is strongly tied to historical memory layouts and native-code interoperability. It is **Source-supported, grammar pending** and is not required for portable v0.1 programs. A compiler may reject it with a specific unsupported-feature diagnostic until a target-independent rule is adopted.

## 6. Statements and control constructs

### 6.1 Assignment

The assignment operator is `=`.

Historical mode rules are:

- a scalar target receives a scalar expression;
- a parallel target may receive a scalar expression (broadcast) or a parallel expression;
- a logical parallel target receives a logical parallel expression.

For a parallel assignment, only the current active set is modified.

```text
total = total + 1;
weight[$] = weight[$] + 5;
selected[$] = (weight[$] >= 10) && (weight[$] < 20);
```

Retrieving a parallel value through a single index produces a scalar value. Other implicit scalar/parallel conversions are not established and are **Unresolved**.

### 6.2 Scalar `if`

When the condition is scalar, `if` is an ordinary branch. Exactly one selected arm executes.

```text
if (count == 0) then
    result = 1;
else
    result = 2;
endif;
```

### 6.3 Parallel `if`

When the condition is a logical parallel expression, `if` is responder-based masking, not an ordinary scalar branch.

Let `A` be the incoming active set, `T` the responders in `A`, and `F = A - T`.

```text
if (parallel_condition) then
    then_statements
else
    else_statements
endif;
```

The `then` statements execute with active set `T`. If present, the `else` statements execute with active set `F`. On exit, active set `A` is restored. Historical hardware broadcasts both arms under complementary masks; a backend may skip an empty arm so long as observable semantics are unchanged.

Whether scalar side effects inside both arms are permitted, and their ordering if permitted, is **Unresolved**. Portable v0.1 programs should not place scalar side effects in both arms of a parallel `if`.

### 6.4 `elsenany`

The Primer uses `elsenany` ("else none any") with several responder constructs. For a parallel `if`, it means: execute the first body with the responder set if at least one responder exists; otherwise execute the `elsenany` body using the original incoming active set.

```text
if (parallel_condition) then
    responder_statements
elsenany
    no_responder_statements
endif;
```

Only one body is selected. The exact restrictions on mixing `else` and `elsenany` are **Unresolved**; a construct containing both must be rejected in v0.1.

### 6.5 `setscope`

`setscope` explicitly restricts the active set:

```text
setscope selected[$]
    statements
endsetscope;
```

Its condition is evaluated over the incoming active set. The body executes on responders, and the incoming active set is restored on exit. `setscope` is Required.

### 6.6 `any`

`any` performs a global existence test on a logical parallel expression. If one or more responders exist, its first body executes using the original incoming active set, not merely the responder subset. If none exist, an optional `elsenany` body executes, also using the incoming active set.

```text
any (parallel_condition)
    statements
elsenany
    no_responder_statements
endany;
```

This distinction from parallel `if` is Required and comes directly from the Primer.

### 6.7 `first` / `loop` / `until`

Historical ASC provides a post-test loop in which one or more `until` tests may appear within the body:

```text
first
    initialization
loop
    statements
    until (scalar_condition)
    more_statements
endloop;
```

The Primer also supports a logical parallel condition and `until nany parallel_condition`. A scalar condition exits when true; a parallel condition without `nany` exits when responders exist; with `nany`, it exits when no responders exist. This construct is **Source-supported, grammar pending**, especially regarding multiple `until` statements, omitted `first`, and semicolon placement.

### 6.8 Responder iteration: `for`

`for` evaluates a logical parallel expression once, records its responders, and processes them one at a time using an index variable:

```text
for idx in parallel_condition
    statements_using(idx)
endfor idx;
```

The condition is not reevaluated after each iteration. Each initial responder is selected once. The historical Primer describes top-to-bottom order, but the 1994 model permits arbitrary responder selection. Until logical row-order requirements are finalized, selection order is **Unresolved**, and portable programs must not depend on it.

The optional `elsenany` form is source-supported but its exact timing and active-set semantics are **Source-supported, grammar pending**.

### 6.9 Responder iteration: `while`

`while` resembles `for`, except that its logical parallel expression is reevaluated before every iteration:

```text
while idx in parallel_condition
    statements_using(idx)
endwhile idx;
```

Each iteration selects one current responder. The body must eventually change relevant state so that the expression ceases to produce responders, unless nontermination is intended. Selection order has the same unresolved status as `for`.

### 6.10 `get`

`get` evaluates a logical parallel expression, selects one responder, binds the index variable to it, and executes the body once. If no responder exists, the first body is skipped and an optional `elsenany` body executes.

```text
get idx in parallel_condition
    statements_using(idx)
elsenany
    no_responder_statements
endget idx;
```

`get` does not remove or alter the selected responder merely by selecting it. A repeated `get` may therefore select the same row unless the program changes the condition. The source calls the selected responder the first/topmost; the tie rule is **Unresolved** pending a normative row-order definition.

### 6.11 `next`

`next` operates on a logical parallel variable, selects its first/topmost set entry, binds an index to that row, clears that entry in the logical variable, and executes the body once:

```text
next idx in selected[$]
    statements_using(idx)
elsenany
    no_responder_statements
endnext idx;
```

Unlike `get`, the historical form requires a logical parallel variable rather than an arbitrary expression. The clearing behavior is Required; selection order remains **Unresolved**.

## 7. Expressions and operators

### 7.1 Expression modes

An expression is scalar or parallel.

- A scalar expression is evaluated once by the scalar control context.
- A parallel expression is evaluated independently at every PE in the current active set.
- Combining a parallel operand with a scalar operand broadcasts the scalar operand, where the operand types permit the operation.
- Parentheses change grouping only; they do not convert modes or alter active-set semantics.

### 7.2 Required operators

| Category | Operators |
|---|---|
| Grouping | `(` `)` |
| Unary | `!`, unary `+`, unary `-` |
| Multiplicative | `*`, `/` |
| Additive | `+`, `-` |
| Relational | `<`, `<=`, `>`, `>=` |
| Equality | `==`, `!=` |
| Boolean AND | `&&` |
| Boolean OR | `||` |
| Assignment | `=` |

Historical exclusive-or exists, but its v0.1 spelling and typing are **Unresolved**.

### 7.3 Precedence and associativity

From highest to lowest:

1. Parenthesized expressions
2. Unary `!`, `+`, `-` (right associative)
3. `*`, `/` (left associative)
4. `+`, `-` (left associative)
5. `<`, `<=`, `>`, `>=` (left associative syntactically; chained comparisons are not defined)
6. `==`, `!=` (left associative syntactically; chained equality is discouraged)
7. `&&` (left associative)
8. `||` (left associative)
9. `=` (right associative if assignment expressions are admitted)

Whether assignment is an expression or only a statement is **Unresolved**. Until finalized, a conforming v0.1 compiler must accept simple assignment statements and may reject nested assignment.

The sources do not establish short-circuit evaluation for `&&` or `||`. They are Boolean combinators over scalar or parallel logical values. Programs must not rely on operand side effects or short-circuiting.

## 8. Reductions and responder queries

The following historical functions are source-supported:

- `maxval(field[$])`: maximum value among active responders.
- `minval(field[$])`: minimum value among active responders.
- `maxdex(field[$])`: an index of an active responder holding the maximum.
- `mindex(field[$])`: an index of an active responder holding the minimum.
- `count(...)`: number of active responders.
- `nthval(field[$], n)`: nth value in ascending order.
- `nthdex(field[$], n)`: index of the nth value in ascending order.

`maxval`, `minval`, `maxdex`, `mindex`, and responder count are Required semantic operations. Exact call typing, empty-set behavior, tie selection, `n` origin, out-of-range behavior, and the Primer's precise `count(them[$])` grammar are **Unresolved**.

Until those rules are finalized:

- applying a value or index reduction to an empty responder set must produce a run-time diagnostic or a compile-time proof error, not an invented sentinel;
- programs must not depend on which tied row `maxdex` or `mindex` returns; and
- `count` of an empty responder set is zero.

## 9. Input/output, allocation, and modularity

### 9.1 Parallel I/O

The Primer defines:

- `read` to load associated parallel fields, allocating one record per input row;
- `print` to output associated parallel fields; and
- `msg` to output text and scalar values, with a debugging form that can dump parallel fields.

These features are **Source-supported, grammar pending**. File handles, input termination, formatting, error handling, and interaction with inactive records are not normatively specified in v0.1.

### 9.2 Dynamic record allocation

Historical ASC provides:

```text
allocate idx in association[$]
    statements_using(idx)
endallocate idx;

release parallel_condition from association[$];
```

`allocate` obtains a record from an idle pool and scopes an index to its body. `release` returns all matching records to the idle pool in parallel. These are **Source-supported, grammar pending**. Exhaustion behavior, initialization, dangling indexes, and association consistency require further specification.

### 9.3 Subroutines and `call`

Historical `call` explicitly maps caller and callee fields using input and output redirection-like notation. The Primer also states that declarations could be shared with `#include`, with target-dependent rules for separate files.

Subroutines and explicit field mapping are **Source-supported, grammar pending**. Parameter direction, aliasing, local variables, separate compilation, include processing, recursion, and scalar argument mapping are **Unresolved**.

### 9.4 Advanced historical constructs

The following are documented by the Primer but are not sufficiently defined for normative v0.1 implementation:

- associative pronouns and articles: `them`, `their`, `its`, `each`, `the`, and `a`;
- inter-PE shifts through expressions such as `$+1`;
- structure-code functions;
- `stackwhile` / `recurse` restricted recursion;
- `andif` / `andfor` complex pattern search;
- embedded target assembly (`asmcode` / `endasm`);
- the performance monitor (`perform`, `sc_perform`, `pa_perform`);
- absolute storage placement and historical native-language interoperability.

They are **Source-supported, grammar pending** and reserved for later specification. A v0.1 implementation should recognize them well enough to issue an explicit unsupported-feature diagnostic rather than misparse them as ordinary identifiers.

Multiple instruction-stream control parallelism described in the 1994 paper is outside the v0.1 language. It may be used internally by a backend only if it preserves single-instruction-stream observable semantics.

## 10. Conformance

### 10.1 Compiler conformance

A conforming v0.1 compiler must:

1. implement the single-instruction-stream active-set and responder semantics in this document;
2. accept lowercase, case-sensitive keywords and case-sensitive identifiers;
3. implement the Required comparison, Boolean, arithmetic, grouping, and assignment syntax;
4. keep execution target selection outside ASC source;
5. accept `--target=cpu`, `--target=cpu-simd`, and `--target=gpu`, or clearly report that a recognized backend is not yet available;
6. preserve observable behavior across implemented targets;
7. diagnose uses of unresolved or unsupported historical constructs rather than assign them invented semantics; and
8. document implementation limits, including PE capacity, integer widths, and unavailable backends.

### 10.2 Reference semantics

The `cpu` target is the reference for semantic testing. It may implement parallel fields as arrays and active/responder sets as masks, but those representations are not language-visible.

The same conformance suite should run against every backend. Tests must cover at least:

- nested active-set save and restore;
- parallel `if` true and false partitions;
- `elsenany` with empty and nonempty responder sets;
- `any` executing under the incoming active set;
- scalar broadcast and masked parallel assignment;
- `get` versus destructive `next` behavior;
- `for` snapshot versus `while` reevaluation;
- reductions with unique extrema, ties, and empty sets; and
- case sensitivity and parenthesized operator grouping.

### 10.3 Extensions

An implementation extension must be disabled by default or must use syntax that cannot be mistaken for specified ASC. Extensions must not redefine a Required construct. Diagnostics and documentation must identify extension behavior as nonstandard.

## 11. Known open issues for v0.2

The following require source comparison, recovered compiler evidence, or explicit design decisions:

1. Complete lexical grammar and reserved-word set.
2. Normative declaration grammar and declaration ordering.
3. Exact scalar and parallel type system, widths, conversions, and overflow.
4. Logical row ordering and responder-selection rules.
5. Scalar effects inside responder-masked constructs.
6. Empty-set and tie behavior for reductions.
7. Full grammar for `for`, `while`, `get`, `next`, `first`/`loop`/`until`, and `elsenany`.
8. Association membership, aliasing, allocation, and release invariants.
9. Subroutine mapping, scope, separate compilation, and include behavior.
10. Parallel I/O formats and error semantics.
11. Pronouns/articles, structure codes, shifts, restricted recursion, and complex searches.
12. Whether and how to offer a separately enabled legacy-source compatibility mode.

## Appendix A. Status summary

| Area | v0.1 status |
|---|---|
| Single instruction stream, active sets, responders | Required |
| Search-process-retrieve model and nested restoration | Required |
| Scalar and parallel modes | Required semantic distinction |
| Leading mode qualifier (`parallel int`, `scalar int`) | Required modernization |
| Lowercase case-sensitive keywords and identifiers | Required modernization |
| `== != < <= > >=`, `&& || !` | Required modernization |
| Parentheses and precedence | Required modernization |
| Compiler `--target=cpu|cpu-simd|gpu` | Required modernization |
| Source-level placement directives | Prohibited |
| Assignment and scalar/parallel `if` | Required core |
| `setscope`, `any` | Required core |
| `for`, `while`, `get`, `next` | Core semantics required; grammar/order details pending |
| Associations and index retrieval | Required core; edge rules pending |
| Minimum, maximum, index, and count reductions | Required operations; edge rules pending |
| I/O, allocation, subroutines | Source-supported, grammar pending |
| Pronouns, structure codes, restricted recursion, complex search | Source-supported, grammar pending |
| Multiple instruction-stream source semantics | Out of scope |

## Appendix B. Illustrative modernized fragment

This fragment demonstrates confirmed surface syntax and core responder semantics. It is not a claim that every declaration detail is final.

```text
main classify
    scalar int selected_count;
    parallel int value[$], result[$];
    parallel logical rows[$];

    associate value[$], result[$] with rows[$];

    setscope rows[$]
        if ((value[$] >= 10) && (value[$] < 20)) then
            result[$] = value[$] * 2;
        else
            result[$] = 0;
        endif;

        selected_count = count(them[$]);
    endsetscope;
end;
```

The use of `them` in the final statement is historical but its exact v0.1 grammar remains pending. A conformance test should use the compiler's finalized responder-count form once Section 8 is resolved.
