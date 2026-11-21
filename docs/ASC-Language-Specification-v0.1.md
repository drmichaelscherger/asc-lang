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
4. Boolean operations use `!`, `&&`, `||`, and `^^` (exclusive OR).
5. Arithmetic includes the remainder operator `%` in addition to `-`, `+`, `*`, and `/`.
6. Bitwise operations use `~`, `&`, `^`, and `|`.
7. Parentheses may group expressions, including nested expressions.
8. Conventional C-style expression precedence is used as specified in Section 7.
9. Every variable declaration begins with its storage mode as a qualifier, followed by its type: `parallel int field[$];` and `scalar int value;`.
10. The primitive value types are the predefined identifiers `int`, `real`, `logical`, and `string`. There is no separate character type. `index` is a reserved special type used only for parallel association-entry selectors.
11. Source-level bit-width suffixes are not permitted. `int` is always signed 64-bit two's-complement, and `real` is always IEEE 754 binary64.
12. Compile-time constants use the typed forms `defint`, `defreal`, `deflog`, and `defstr`.
13. Procedures and functions use Pascal-style declarations and calls. Arguments are passed by value unless preceded by `var`; a function declares its result with `returns scalar data_type`; and calls use ordinary `name(arguments)` syntax without a `call` keyword.
14. Scalar `for` loops use inclusive Pascal-style `to` and `downto` bounds. Scalar `while`, responder-oriented parallel loops, and `loop` / `until` are distinct constructs.
15. Every block terminator is followed by a semicolon and does not repeat a control or index variable: for example, `endfor;`, `endwhile;`, and `endget;`.
16. Standard and association I/O use the predefined functions `open`, `close`, `read`, `readln`, `write`, and `writeln`. Calls use explicit integer stream handles and return a scalar `logical` success result.
17. A program occupies exactly one source file and contains exactly one `main`; imports, textual inclusion, separate compilation, and forward routine declarations are not supported.
18. Association access is explicit. Historical implicit pronouns and articles, physical-PE shift notation, and `andif` / `andfor` complex-search syntax are excluded and are not reserved.

Historical uppercase keywords, dot-delimited operators, type-before-mode declarations, closing identifiers, and all other legacy syntax are not part of the v0.1 source grammar. A conforming ASC v0.1 compiler must not provide a legacy-source compatibility mode. A separate source-conversion tool may translate historical programs into the new ASC syntax, but such conversion is outside this language specification.

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

Association entries have no language-defined order. Where a historical source says "first," "topmost," "random," or "arbitrary," v0.1 specifies arbitrary responder selection: an implementation chooses one eligible responder, no probabilistic randomness is required, and a portable program must not depend on which responder is selected. Stability and responder-set updates required by a particular construct are specified with that construct.

### 3.5 Abstract cost versus concrete cost

The historical model treats associative search, global Boolean operations, responder selection, and minimum/maximum identification as constant-time primitives under its machine assumptions. This is an algorithmic property of the ASC model, not a performance guarantee for `cpu`, `cpu-simd`, or `gpu`. A conforming backend may lower one abstract operation to multiple physical instructions.

## 4. Lexical structure

### 4.1 Source files and statement termination

An ASC v0.1 program consists of exactly one source file, conventionally using the `.asc` suffix, and must be encoded as well-formed UTF-8. Invalid UTF-8 is a compile-time error. Separate compilation, source imports, and textual inclusion are not supported. Statements shown by the sources terminate with `;`. Block introducers do not consistently carry semicolons in the Primer; the normative productions in this document show where v0.1 requires them.

Every ASC block terminator must be followed by a semicolon. This rule applies to all currently specified block terminators and to any block construct added in a later version of the language. The normative productions in Appendix C show the required semicolon explicitly.

### 4.2 Keywords and case

Keywords are lowercase only. `if` is a keyword; `IF` and `If` are identifiers if otherwise lexically valid. Identifier lookup is case-sensitive, so `value`, `Value`, and `VALUE` are distinct.

The following lowercase spellings form the complete v0.1 reserved-word set:

```text
allocate        any             associate
defint          deflog          defreal         defstr
downto          during          else            elsenany
end
endany
endallocate     endfor          endfunction     endget
endif           endifany        endloop         endnext
endprocedure    endsetscope     endstack
endwhile        for             from
function        get             if
ifany           in              index
loop            main            nany
next            parallel
procedure       release
return          returns         scalar
setscope        stack           stop
then            to
until           var             while           with
```

All of these spellings are reserved even when their grammar or semantics remain pending elsewhere in this specification. They cannot be used as identifiers.

### 4.3 Predefined identifiers

Predefined identifiers are distinct from reserved words. They have the lexical form of ordinary identifiers and are resolved through predefined language bindings rather than recognized as keyword tokens.

ASC v0.1 defines these primitive type identifiers:

| Predefined identifier | Meaning |
|---|---|
| `int` | 64-bit integer type |
| `logical` | Boolean type |
| `real` | IEEE 754 binary64 floating-point type |
| `string` | String type |

ASC v0.1 defines these constant identifiers:

| Predefined identifier | Type | Meaning |
|---|---|---|
| `false` | `logical` | Boolean false value |
| `maxint` | `int` | Largest representable `int` value |
| `stderr` | `int` | Standard-error stream handle, with value 2 |
| `stdin` | `int` | Standard-input stream handle, with value 0 |
| `stdout` | `int` | Standard-output stream handle, with value 1 |
| `true` | `logical` | Boolean true value |

ASC v0.1 defines these routine identifiers:

| Predefined identifier | Kind | Purpose or result |
|---|---|---|
| `abs` | Function | Absolute value |
| `arctan` | Function | Arc tangent |
| `chr` | Function | Convert an integer character code to a one-character string |
| `close` | Function | Close an opened external stream and report success |
| `cos` | Function | Cosine |
| `count` | Function | Return the number of responders as a scalar `int` |
| `eof` | Function | Test for end of input |
| `eoln` | Function | Test for end of the current input line |
| `exp` | Function | Exponential function |
| `ln` | Function | Natural logarithm |
| `maxdex` | Function | Return an index designating an entry containing the maximum value |
| `maxval` | Function | Return the maximum responding value |
| `mindex` | Function | Return an index designating an entry containing the minimum value |
| `minval` | Function | Return the minimum responding value |
| `nthdex` | Function | Return the index of the nth ordered responding value |
| `nthval` | Function | Return the nth ordered responding value |
| `odd` | Function | Test whether an integer is odd |
| `open` | Function | Open an external stream and report success |
| `ord` | Function | Return the integer code of a one-character string |
| `pred` | Function | Return the preceding one-character string in the ASCII character table |
| `read` | Function | Read scalar or association data and report success |
| `readln` | Function | Read scalar values, finish the current input line, and report success |
| `round` | Function | Round a real value to an integer |
| `sin` | Function | Sine |
| `sqrt` | Function | Square root |
| `succ` | Function | Return the following one-character string in the ASCII character table |
| `trunc` | Function | Truncate a real value to an integer |
| `write` | Function | Write scalar or association data and report success |
| `writeln` | Function | Write scalar values followed by a line terminator and report success |

Element-wise predefined functions use mode propagation. Given scalar arguments, they return a scalar result; given at least one compatible parallel argument, they operate independently on the current active entries and return a parallel result. This rule applies to the mathematical, numeric-conversion, parity, and character-code functions. The responder reductions `count`, `maxval`, `minval`, `maxdex`, `mindex`, `nthval`, and `nthdex` retain their separately specified reduction modes. The stream inquiries `eof` and `eoln` are scalar.

Let `M` denote either `scalar` or `parallel`. The following signatures are normative; the result has the same mode `M` as the argument unless the signature explicitly says `scalar`:

| Function signature | Result |
|---|---|
| `abs(M int)` | `M int` |
| `abs(M real)` | `M real` |
| `arctan(M real)` | `M real` |
| `chr(M int)` | `M string` |
| `cos(M real)` | `M real` |
| `eof()` | `scalar logical` |
| `eoln()` | `scalar logical` |
| `exp(M real)` | `M real` |
| `ln(M real)` | `M real` |
| `odd(M int)` | `M logical` |
| `ord(M string)` | `M int` |
| `pred(M string)` | `M string` |
| `round(M real)` | `M int` |
| `sin(M real)` | `M real` |
| `sqrt(M real)` | `M real` |
| `succ(M string)` | `M string` |
| `trunc(M real)` | `M int` |

Each signature takes exactly the arguments shown. `eof()` and `eoln()` inspect standard input. `chr`, `ord`, `pred`, and `succ` use the ASCII character table; ASC strings, rather than a separate character type, represent their one-character results and arguments. `ord`, `pred`, and `succ` require a string containing exactly one character.

`round` rounds to the nearest integer, resolving an exact halfway case to the even integer. `trunc` discards the fractional part and rounds toward zero. Either signed floating-point zero converts to integer zero. For either function, a NaN, an infinity, or a finite result outside the `int` range is a fatal run-time error that terminates the entire program. In a parallel call, an invalid argument in any active entry terminates the entire program.

ASC v0.1 uses 7-bit ASCII codes 0 through 127 for the character-code functions. `chr(n)` requires `0 <= n <= 127` and returns a one-character string; ASCII control characters are permitted. `ord(s)` requires exactly one ASCII character and returns its code. `pred(s)` requires a one-character ASCII string whose code is greater than 0. `succ(s)` requires a one-character ASCII string whose code is less than 127. Violating any of these requirements is a fatal run-time error that terminates the entire program; in a parallel call, one invalid active entry terminates the entire program.

The signatures above are Required. For every finite input whose mathematical result is finite, `sin`, `cos`, `arctan`, `exp`, and `ln` must return a binary64 value whose error is no greater than one unit in the last place (1 ULP) at the returned value's magnitude. This accuracy requirement applies equally to scalar evaluation and to every active entry of a parallel evaluation.

Predefined mathematical functions use the following IEEE-style special-value rules:

| Function and input | Required result |
|---|---|
| `abs(-0.0)` | `+0.0` |
| `abs(±infinity)` | `+infinity` |
| `sqrt(x)` for finite `x < 0.0` | NaN |
| `sqrt(-infinity)` | NaN |
| `sqrt(-0.0)` | `-0.0` |
| `sqrt(+infinity)` | `+infinity` |
| `ln(x)` for finite `x < 0.0` | NaN |
| `ln(-infinity)` | NaN |
| `ln(±0.0)` | `-infinity` |
| `ln(+infinity)` | `+infinity` |
| `exp(+infinity)` | `+infinity` |
| `exp(-infinity)` | `+0.0` |
| `sin(±infinity)` or `cos(±infinity)` | NaN |
| `sin(±0.0)` | The same signed zero |
| `cos(±0.0)` | `1.0` |
| `arctan(±infinity)` | The corresponding signed approximation to π/2 within 1 ULP |
| `arctan(±0.0)` | The same signed zero |
| Any listed mathematical function with a NaN argument | NaN |

For finite `exp` inputs, overflow produces `+infinity` and underflow produces a positive subnormal value or `+0.0` under binary64 round-to-nearest, ties-to-even. These special-value cases do not terminate the program. Division by zero remains the explicit exception specified in Section 7.2 and terminates the entire program.

ASC uses one canonical binary64 quiet NaN with bit pattern `0x7ff8000000000000`. Its sign bit is zero and its payload is zero. Every operation or predefined function that produces NaN must produce this canonical value, regardless of operand NaN signs or payloads. Operand payloads are not preserved. Any textual NaN accepted by an implementation and every reduction whose result is NaN likewise produce the canonical value. This rule is identical for `cpu`, `cpu-simd`, and `gpu` targets.

The lowercase predefined bindings are always visible and cannot be redeclared or shadowed in any scope by a conforming v0.1 program. Because identifiers are case-sensitive, differently cased identifiers such as `Read` or `Write` do not denote these routines and may be declared by a program. Adding a predefined identifier does not add that spelling to the reserved-word set.

### 4.4 Identifiers

An identifier begins with an ASCII letter and continues with zero or more ASCII letters, decimal digits, or underscores. Identifiers are case-sensitive, and a reserved word cannot be used as an identifier. The normative lexical productions are in Appendix C.1.

ASC v0.1 places no language-defined length limit on an identifier. An implementation may fail when an identifier exhausts available implementation resources, but it must not silently truncate an identifier or treat two distinct identifiers as equal.

### 4.5 White space, comments, and tokenization

Outside a string or comment, the only white-space characters are space (U+0020), horizontal tab (U+0009), form feed (U+000C), line feed (U+000A), and carriage return (U+000D). A carriage return followed immediately by a line feed is one source-line terminator. White space separates tokens but is otherwise insignificant. There is no source-line length limit.

The Primer specifies C-style comments:

```text
/* comment */
```

It also states that comments may nest. Therefore nested `/* ... */` comments are Required. A comment is equivalent to white space and may appear anywhere white space may appear, but it cannot split a token. Comment delimiters inside a string are ordinary string characters, and string delimiters inside a comment have no special meaning. An unterminated comment is a compile-time error. ASC v0.1 has no line-comment form.

The lexer uses longest-token matching. Thus `<=`, `>=`, `==`, `!=`, `&&`, `||`, and `^^` are each one token when their characters are adjacent. A reserved word is recognized only when the complete identifier-shaped token exactly matches its lowercase spelling. Any Unicode scalar value outside a string or comment that is neither an ASCII token character nor one of the white-space characters listed above is a compile-time error. The lexical productions and the complete reserved-word list together define all v0.1 tokens; there is no implementation-defined token syntax.

### 4.6 Literals

ASC v0.1 provides the predefined logical constants `true` and `false`; they are identifiers rather than reserved words or literal tokens. Integer literals may be decimal or use the lowercase prefixes `0b`, `0o`, and `0x` for binary, octal, and hexadecimal. Real literals use decimal notation with a decimal point, an exponent, or both. String literals are enclosed in double quotation marks. The normative productions are in Appendix C.1.

Every decimal real literal is converted to the nearest IEEE 754 binary64 value using round-to-nearest, ties-to-even. Overflow produces the appropriately signed infinity; underflow produces a subnormal value or signed zero under the same rounding rule.

An integer literal, including its optional sign, must have a mathematical value in the signed 64-bit `int` range -9,223,372,036,854,775,808 through 9,223,372,036,854,775,807. This rule applies uniformly to decimal, binary, octal, and hexadecimal literals. An out-of-range integer literal is a compile-time error and does not wrap. Consequently, `-9223372036854775808` is valid, while `9223372036854775808`, `-9223372036854775809`, and `0xFFFFFFFFFFFFFFFF` are invalid literals. Run-time integer arithmetic continues to wrap as specified in Section 5.3.

A `string` value is a sequence of Unicode scalar values. String literals may contain Unicode characters directly and cannot span source lines. An unescaped double quotation mark ends the literal, and a backslash begins an escape. ASC v0.1 supports exactly `\"`, `\\`, `\n`, `\r`, `\t`, `\0`, and `\u{H...}`, where the Unicode escape contains one through six hexadecimal digits. A Unicode escape must denote a scalar value no greater than `0x10FFFF` and must not denote a surrogate in `0xD800` through `0xDFFF`. An unknown or malformed escape, an unescaped source-line terminator, or an invalid Unicode scalar escape is a compile-time error.

## 5. Program structure and declarations

### 5.1 Compilation units

An ASC v0.1 compilation unit is the complete contents of its single source file and contains one or more top-level units: exactly one main program and zero or more procedures or functions. Procedures and functions cannot be nested.

```text
main program_name
    declarations
    statements
end;

procedure procedure_name(scalar int value)
    declarations
    statements
endprocedure;

function function_name(scalar int value) returns scalar int
    declarations
    statements
endfunction;
```

All declarations in a top-level unit precede its executable statements. Constant definitions, variable declarations, and association declarations may be intermingled, but every name must be declared before use. The normative compilation-unit and routine grammar is in Appendix C.12.

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
scalar int item_count;
parallel int value[$];
parallel logical selected[$];
parallel index idx[$];
```

The normative declaration productions are consolidated in Appendix C.

A dimension is a compile-time scalar `int` expression composed only of integer literals, references to earlier `defint` constants, parentheses, and the integer operators `+`, `-`, `~`, `*`, `/`, `%`, `&`, `^`, and `|`. Function calls, variables, comparisons, and logical operators are not permitted. Operators use the normal precedence and 64-bit wrapping rules. Division or remainder by zero is a compile-time error. The final value must be greater than zero.

A declared dimension is an element count, not a maximum index. Every ordinary array dimension is zero-based: a dimension of `N` permits indexes 0 through `N - 1`. For example, `scalar int values[2 * 2];` has four elements indexed 0 through 3. A scalar declaration may omit dimensions. A parallel declaration must contain the distinguished `$` dimension and may contain additional dimensions after it.

Every subscript must be within its declared dimension. An out-of-bounds scalar subscript is a fatal run-time error. In a parallel operation, an out-of-bounds subscript in any active entry terminates the entire program. A fixed declaration whose dimension sizes or multidimensional layout calculation cannot be represented within signed 64-bit implementation indexing is a compile-time error. An implementation may impose a smaller documented storage limit.

This rule currently declares one identifier per statement. A later rule may add comma-separated declarators only if explicitly adopted.

The ordering `data_type scalar` or `data_type parallel` is historical syntax and is not accepted as standard v0.1 syntax. The historical `:width` field-width suffix is also excluded: a declaration cannot specify a bit width after its identifier.

### 5.3 Data types

ASC v0.1 has four primitive value types: `int`, `real`, `string`, and `logical`. Each primitive type may be declared with either the `scalar` or `parallel` storage class. A scalar value exists once in the scalar control context; a parallel field contains one value per applicable association entry.

`logical` is an abstract two-value type containing exactly `false` and `true`. Its physical representation is not observable. An implementation may represent scalar or parallel logical values as bits, masks, bytes, or machine words, but no other logical value is permitted and program behavior must not depend on the representation.

An `int` is a signed 64-bit (8-byte) two's-complement integer with the range -9,223,372,036,854,775,808 through 9,223,372,036,854,775,807. The predefined constant `maxint` is 9,223,372,036,854,775,807. Integer overflow and underflow wrap modulo 2^64; the resulting 64-bit pattern is interpreted as a signed two's-complement value. This wrapping rule applies to scalar and parallel integer arithmetic. In particular, the positive mathematical value of `abs(-9223372036854775808)` is not representable, so the result wraps back to `-9223372036854775808`; the same rule applies independently to each active entry of a parallel `abs` call.

The mathematical quotient of `-9223372036854775808 / -1` is not representable and therefore wraps to `-9223372036854775808`; the corresponding remainder is zero.

A `real` is an IEEE 754 binary64 floating-point value. Basic arithmetic operations and `sqrt` use round-to-nearest, ties-to-even. Implementations must support signed zero, subnormal values with gradual underflow, positive and negative infinity, and NaN. Except for division by zero, floating-point exceptions do not trap and IEEE exceptional results are produced instead. Comparisons follow IEEE 754 unordered-NaN behavior: every ordered comparison with NaN is false, `==` with NaN is false, and `!=` with NaN is true.

A conforming implementation must not flush subnormal values to zero, reassociate floating-point expressions, assume that NaN or infinity cannot occur, or contract separate source operations into a fused operation when doing so changes the binary64 result. A compiler may offer a separately enabled nonconforming fast-math mode, but it must not be the default and must be clearly identified as an extension.

An `int` value converts implicitly to `real` when required by assignment, parameter passing, function return, comparison, or a mixed `int`/`real` arithmetic expression. The conversion produces the nearest binary64 value using round-to-nearest, ties-to-even. Every `int` is within the finite binary64 range, but magnitudes greater than 2^53 may lose precision. Mixed `int`/`real` arithmetic and comparisons convert the `int` operand to `real` and operate in binary64.

There is no implicit `real`-to-`int` conversion; a program must use `round` or `trunc`. There are no implicit conversions between numeric types and `string`, between numeric types and `logical`, or between `string` and `logical`. The required textual representation of `real` values is specified in Section 9.1.

`index` is a special responder-selection type, not a primitive value type. It is used for parallel index variables that designate association entries. The historical type spellings `char`, `card`, `bin`, `hex`, and `oct` are neither types nor reserved words and may be used as ordinary identifiers. Text is represented exclusively by `string`. Removing the historical numeric spellings does not affect the `0b`, `0x`, and `0o` integer-literal prefixes.

The scalar/parallel distinction, `parallel logical`, `parallel index`, and scalar/parallel `string` are Required semantic categories. The observable content of a string is its sequence of Unicode scalar values; an implementation's internal string encoding and storage representation are not language-visible. String mutability, storage management, general comparison, concatenation, and length operations remain **Unresolved**. A compiler must not infer unresolved behavior solely from a host C type. Source-level width annotations are not permitted.

### 5.4 Index variables

An index variable identifies one responder row so that an associated parallel value can be retrieved or updated using syntax such as `value[idx]`. Index variables connect search, process, and retrieve operations.

Historical descriptions sometimes call an index variable a one-bit parallel field, reflecting a one-hot representation. That representation is not source-observable and is not required of modern backends. The semantic requirement is that an index designates either no row or exactly one row within its valid scope.

### 5.5 Constants

ASC v0.1 provides four typed constant definitions:

```text
deflog(identifier, boolean_constant);
defint(identifier, integer_constant);
defint(identifier, variable + integer_constant);
defint(identifier, variable - integer_constant);
defreal(identifier, real_constant);
defreal(identifier, variable + real_constant);
defreal(identifier, variable - real_constant);
defstr(identifier, string_constant);
```

The generic historical `define` keyword is not part of ASC v0.1. In a relative `defint`, the leading identifier must name an earlier `defint` constant; in a relative `defreal`, it must name an earlier `defreal` constant. Both forms are evaluated entirely at compile time and obey declaration-before-use. Relative `defint` arithmetic uses the normal wrapping 64-bit rules. Relative `defreal` arithmetic uses the normal IEEE 754 binary64 rules. Runtime scalar or parallel variables are not permitted in constant definitions. The normative productions are in Appendix C.2.

### 5.6 Associations

An association groups parallel fields into a record/table under a logical parallel field:

```text
associate tail[$], head[$], weight[$] with graph;
```

The associated fields and association field must already be declared. Association membership determines which values belong to the same logical record. An association is a logical table: all of its member fields share exactly the same set of allocated rows, and allocation or release changes that shared row set atomically.

A parallel field may belong to at most one association: after it appears as an item in one association declaration, it cannot appear in any other association declaration. A field also cannot occur more than once within the same declaration. Only parallel `int`, `real`, `logical`, and `string` fields may be association items. A `parallel index` is a temporary row selector and cannot be association data. Violations are compile-time errors. Association declaration, exclusive field membership, and these row invariants are Required.

The normative association declaration production is consolidated in Appendix C.

## 6. Statements and control constructs

### 6.1 Assignment

The assignment operator is `=`. A single `=` is never a relational or equality operator in ASC v0.1. Equality comparisons must use `==`, and inequality comparisons must use `!=`. This intentionally replaces any historical Primer example or legacy ASC spelling that uses a single `=` for equality.

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

Retrieving a parallel value through a single index produces a scalar value. Within a `get`, `next`, or responder-iteration body, `field[idx]` denotes the value of `field` in the association entry flagged by `idx`. The indexed form is also an assignable scalar target when `field` is assignable. All fields projected through the same index variable refer to the same association entry.

A compatible scalar expression may be broadcast explicitly by context to a parallel target or combined with a parallel operand, producing an element-wise parallel operation over the current active set. A parallel value never converts implicitly to scalar; it must be scalarized by an indexed projection or a reduction. No other implicit scalar/parallel mode conversion is permitted.

### 6.2 Scalar `if`

When the condition is scalar, `if` is an ordinary branch. Exactly one selected arm executes.

```text
if (item_count == 0) then
    result = 1;
else
    result = 2;
endif;
```

The normative scalar `if` production is in Appendix C.7.

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

Scalar statements and scalar function calls are permitted in responder-masked blocks. A masked block with a nonempty active set executes each scalar statement once when control reaches it, independent of the number of active responders. A masked block with an empty active set is skipped completely, including its scalar effects. In a parallel `if`, the `then` block is processed before the `else` block; therefore, when both partitions are nonempty, scalar effects in the `then` block occur before scalar effects in the `else` block. This source ordering is Required on every backend.

Constructs selected by a scalar existence decision are not empty masked blocks. The chosen body of `any` and the chosen `elsenany` body execute once using their specified incoming active set, and their scalar statements execute once even when that incoming set is empty. In a responder-iteration body, scalar statements execute once per selected responder. These rules follow Pascal's sequential execution model for scalar effects while retaining ASC masking for parallel effects.

The normative parallel `if` production is in Appendix C.7.

### 6.4 `elsenany`

ASC uses `elsenany` ("else not any") with responder constructs. It is selected only when evaluation of the construct's parallel condition produces no responders in the incoming active set. Its statement block executes once using that original incoming active set; it does not execute under the empty responder set.

For a parallel `if`, the first body executes with the responder set when at least one responder exists. If no responders exist, the `elsenany` body executes using the original incoming active set.

```text
if (parallel_condition) then
    responder_statements
elsenany
    no_responder_statements
endif;
```

Only one alternative body is permitted. A parallel `if` may contain `else` or `elsenany`, but never both. A scalar `if` may contain `else` but not `elsenany`.

### 6.5 `setscope`

`setscope` explicitly restricts the active set:

```text
setscope selected[$]
    statements
endsetscope;
```

Its condition is evaluated over the incoming active set. The body executes on responders, and the incoming active set is restored on exit. `setscope` is Required. Its normative production is in Appendix C.11.

### 6.6 `any`

`any` performs a global existence test on a logical parallel expression. If one or more responders exist, its first body executes once using the original incoming active set, not merely the responder subset. If no responders exist, the first body is skipped and an optional `elsenany` body executes once using the original incoming active set.

The normative `any` production is consolidated in Appendix C.

```text
any (parallel_condition)
    statements
elsenany
    no_responder_statements
endany;
```

This distinction from parallel `if` is Required and comes directly from the Primer.

### 6.7 `loop` / `until`

`loop` / `until` is a post-test loop. Its statement block executes before the termination expression is evaluated, so the block always executes at least once.

```text
loop
    statement_block
    until scalar_expression
endloop;

loop
    statement_block
    until nany parallel_expression
endloop;
```

A scalar `until` exits when its expression is true. A parallel `until` without `nany` exits when its expression produces one or more responders. A parallel `until nany` exits when its expression produces no responders. Otherwise, execution repeats from the start of the statement block.

The parentheses in the source notation `(nany)` indicate that the `nany` keyword is optional; they are not literal source characters. A `loop` contains exactly one `until` clause, immediately before `endloop`. The normative productions are in Appendix C.9.

### 6.8 Scalar `for` and `while`

ASC v0.1 adds explicit scalar forms of `for` and `while`. This is a language design decision, not a claim that the historical ASC language provided these exact forms.

```text
for i = 0 to limit
    statements
endfor;

for i = limit downto 0
    statements
endfor;

while (i < limit)
    statements
endwhile;
```

In a scalar `for`, the initial and final expressions are evaluated exactly once before the first iteration. Both must produce scalar `int` values. The control variable must be a scalar `int` variable. With `to`, it takes each value from the initial value through the final value in ascending steps of one. With `downto`, it takes each value in descending steps of one. The bounds are inclusive. If the initial value is already beyond the final value in the selected direction, the body executes zero times.

The loop terminates after the iteration at the final value without computing a wrapped value beyond that bound. The control variable cannot be assigned, passed to a `var` parameter, or otherwise modified within the loop body. The body may contain any other statements valid in the current active scope, including parallel statements. Scalar loops do not accept `elsenany`. A scalar `while` tests its scalar condition before every iteration and may execute zero times.

The normative scalar-loop productions are consolidated with the parallel-loop productions in Appendix C.9.

### 6.9 Responder iteration: `for`

`for` evaluates a logical parallel expression once, records its responders, and processes them one at a time using an index variable:

```text
for idx in parallel_condition
    statements_using(idx)
endfor;
```

The condition is not reevaluated after each iteration. Each initial responder is selected once. Associations have no language-defined ordering, so responder selection is arbitrary. Each initial responder is visited exactly once, but portable programs must not depend on visitation order.

If the initial `for` condition produces no responders, the loop body does not execute and an optional `elsenany` block executes once using the incoming active set. If the initial condition produces one or more responders, the loop processes those responders and the `elsenany` block is skipped.

The normative parallel `for` production is consolidated in Appendix C.9.

The body must not structurally modify any association traversed by the loop condition. It cannot allocate or release a row of such an association or perform a collective `read` into it. A compiler must diagnose such a modification. This preserves the snapshot of initial responders.

### 6.10 Responder iteration: `while`

`while` resembles `for`, except that its logical parallel expression is reevaluated before every iteration:

```text
while idx in parallel_condition
    statements_using(idx)
endwhile;
```

The normative parallel `while` production is consolidated in Appendix C.9.

Each iteration selects one current responder. The body must eventually change relevant state so that the expression ceases to produce responders, unless nontermination is intended. As with parallel `for`, responder selection is arbitrary and programs must not depend on visitation order.

The body must not structurally modify any association traversed by the loop condition. It cannot allocate or release a row of such an association or perform a collective `read` into it. A compiler must diagnose such a modification. Ordinary assignments to fields used by the condition remain permitted and are how the program changes the responder set between iterations.

### 6.11 `get`

`get` evaluates a logical parallel expression, selects one responder, binds the index variable to it, and executes the body once. If no responder exists, the first body is skipped and an optional `elsenany` body executes.

```text
get idx in parallel_condition
    statements_using(idx)
elsenany
    no_responder_statements
endget;
```

`get` does not remove or alter the selected responder merely by selecting it. Associations have no language-defined order, so the selected responder is arbitrary rather than first, topmost, or random. If a second `get` is nested within the first and no intervening operation changes the association or relevant responder set, it must select the same association entry. No probabilistic randomness is required, and programs must not depend on which eligible entry is selected across otherwise distinct executions.

The body of a `get` must not structurally modify the association being traversed. It cannot allocate or release a row of that association or perform a collective `read` into it. A compiler must diagnose such a modification.

### 6.12 `next`

`next` operates on a parallel variable, selects an arbitrary responding entry associated with that variable, binds an index to that entry, updates the responder set so the selected entry is not selected again by a nested `next`, and executes the body once:

```text
next idx in selected[$]
    statements_using(idx)
elsenany
    no_responder_statements
endnext;
```

Unlike `get`, the historical form requires a parallel variable rather than an arbitrary expression. `next` updates the responder set; it does not thereby assign to or otherwise modify the source parallel variable. Nested `next` operations select distinct association entries. Repeated use can therefore iterate over all responders, but their visitation order is unspecified.

The body of a `next` must not structurally modify the association containing its source parallel variable. It cannot allocate or release a row of that association or perform a collective `read` into it. A compiler must diagnose such a modification.

If there are no responders associated with the parallel variable in the incoming active set, the main block is skipped and an optional `elsenany` block executes once using the incoming active set. The normative `next` production is in Appendix C.10.

## 7. Expressions and operators

### 7.1 Expression modes

An expression is scalar or parallel.

- A scalar expression is evaluated once by the scalar control context.
- A parallel expression is evaluated independently at every PE in the current active set.
- Combining a parallel operand with a scalar operand broadcasts the scalar operand, where the operand types permit the operation.
- Parentheses change grouping only; they do not convert modes or alter active-set semantics.

The normative variable-reference and expression productions are in Appendix C.5.

### 7.2 Required operators

| Category | Operators |
|---|---|
| Grouping | `(` `)` |
| Unary | `!`, `~`, unary `+`, unary `-` |
| Multiplicative | `*`, `/`, `%` |
| Additive | `+`, `-` |
| Relational | `<`, `<=`, `>`, `>=` |
| Equality | `==`, `!=` |
| Bitwise AND | `&` |
| Bitwise exclusive OR | `^` |
| Bitwise OR | `\|` |
| Boolean AND | `&&` |
| Boolean exclusive OR | `^^` |
| Boolean OR | `\|\|` |
| Assignment | `=` |

The comparison operators are exactly `<`, `>`, `<=`, `>=`, `==`, and `!=`. In particular, `==` tests equality and `=` performs assignment; they are not interchangeable. Logical operands support only `==` and `!=`; the ordering operators `<`, `<=`, `>`, and `>=` are invalid for logical values.

`!` negates a logical value. `^^` is logical exclusive OR: it is true exactly when one operand is true and the other is false. The operators `!`, `&&`, `||`, and `^^` accept only logical operands and produce logical results. They may operate on scalar logical values or element-wise on parallel logical values.

`%` is the integer remainder operator and is a confirmed addition to ASC v0.1, whether or not it was supported by the historical language. Integer division truncates its quotient toward zero. For a nonzero divisor, the remainder is defined by `a == (a / b) * b + (a % b)` under ASC's wrapping integer arithmetic and has the same sign as `a`, or is zero. Thus `-4 / 3` is `-1` and `-4 % 3` is `-1`.

Every `int` arithmetic result is reduced modulo 2^64 and interpreted as a signed two's-complement value. Consequently, overflow and underflow do not trap and have identical behavior on every backend. This includes unary negation and the `+`, `-`, `*`, `/`, and `%` operators whenever the operation is otherwise defined.

Division or remainder by zero is a fatal run-time error. For `real` division, a divisor of either positive or negative floating-point zero counts as zero. The implementation must issue a run-time diagnostic and immediately terminate the entire program with a failure status; no subsequent ASC statement executes. In a parallel division or remainder operation, a zero divisor in any active entry terminates the entire program. For division, this rule overrides the IEEE 754 infinity result that would otherwise apply to nonzero finite values divided by floating-point zero.

The bitwise operators act on `int` values. `~` complements all 64 bits; `&`, `^`, and `|` compute bitwise AND, exclusive OR, and OR respectively. They are distinct from the logical operators `!`, `&&`, `^^`, and `||`. With parallel operands, bitwise operations are performed element by element on the current active set. Mixed-type operands are not permitted.

### 7.3 Precedence and associativity

From highest to lowest:

1. Parenthesized expressions
2. Unary `!`, `~`, `+`, `-` (right associative)
3. `*`, `/`, `%` (left associative)
4. `+`, `-` (left associative)
5. `<`, `<=`, `>`, `>=` (left associative syntactically; chained comparisons are not defined)
6. `==`, `!=` (left associative syntactically; chained equality is discouraged)
7. `&` (left associative)
8. `^` (left associative)
9. `|` (left associative)
10. `&&` (left associative)
11. `^^` (left associative)
12. `||` (left associative)

Logical exclusive OR has lower precedence than `&&` and higher precedence than `||`.

Assignment is a statement, not an expression, and therefore has no expression precedence. Nested and chained assignments are invalid.

Expression operands and function arguments are evaluated from left to right. Scalar `&&` and `||` short-circuit: `&&` skips its right operand when the left operand is false, and `||` skips its right operand when the left operand is true. Scalar `^^` evaluates both operands.

Parallel `&&` and `||` use masked short-circuit evaluation. The left operand is evaluated over the incoming active set. For `&&`, the right operand is evaluated only for active entries where the left operand is true; entries where it is false receive a false result without evaluating the right operand. For `||`, the right operand is evaluated only where the left operand is false; entries where it is true receive a true result without evaluating the right operand. The incoming active set is restored after the expression. Parallel `^^` evaluates both operands for every active entry.

A skipped operand is not evaluated, performs no side effects, and cannot cause a run-time error. If a masked parallel right operand contains a scalar function call, that call executes once if its evaluation mask is nonempty and is skipped if the mask is empty. Compilers must preserve these evaluation rules across all backends.

## 8. Reductions and responder queries

Reductions consider only eligible entries in the current active set. For `count`, eligible entries are those for which the supplied parallel logical expression is true. For a value or index reduction over a parallel field, every applicable entry in the current active set is eligible.

The normative signatures are:

| Function signature | Result |
|---|---|
| `count(parallel logical)` | `scalar int` |
| `minval(parallel T)` | `scalar T` |
| `maxval(parallel T)` | `scalar T` |
| `mindex(parallel T)` | `parallel index` |
| `maxdex(parallel T)` | `parallel index` |
| `nthval(parallel T, scalar int)` | `scalar T` |
| `nthdex(parallel T, scalar int)` | `parallel index` |

For these signatures, `T` is `int`, `real`, or `string`. Ordering reductions do not accept `logical` or `index` values. Strings are ordered lexicographically by 7-bit ASCII code; every string participating in an ordering reduction must contain only ASCII characters, or the program terminates with a fatal run-time error.

`count` returns the number of true eligible responders and returns zero for an empty responder set. A conforming implementation must not expose more than `maxint` association entries or active processing elements to one program, so every possible responder count is representable as `int`. Every other reduction requires at least one eligible entry. An empty-set value or index reduction is a fatal run-time error terminating the entire program.

`nthval` and `nthdex` use one-based positions in ascending order. Position 1 denotes the minimum value. The position argument must be at least 1 and no greater than the number of ordered candidates; otherwise the program terminates with a fatal run-time error.

Equal values remain distinct candidates. A value reduction is unaffected by ties. An index reduction selects an arbitrary eligible entry from the tied group occupying the requested extreme or position, and a portable program must not depend on which tied entry is selected. No physical PE, SIMD-lane, or GPU-thread order breaks a tie.

For `real` reductions, NaNs are excluded from ordering whenever at least one non-NaN eligible value exists. The position range for `nthval` and `nthdex` is then based on the remaining non-NaN candidates. If every eligible value is NaN, all eligible entries form one tied NaN group for validating an `nth` position; a value reduction returns the canonical NaN and an index reduction selects an arbitrary eligible entry. For ordering purposes, negative zero precedes positive zero, so `minval` prefers `-0.0` and `maxval` prefers `+0.0` when both are present.

## 9. Input/output, allocation, and modularity

### 9.1 Standard and parallel I/O

ASC v0.1 provides the predefined functions `open`, `close`, `read`, `readln`, `write`, and `writeln`. These names are predefined identifiers rather than reserved words and cannot be redeclared by a conforming program. Each returns a `scalar logical` success result. Because ASC does not permit a function call to stand alone as a statement, a program must use that result, for example:

```text
ok = open(handle, "data.txt", "r");
ok = read(handle, value, name);
ok = close(handle);
```

Stream handles are scalar `int` values. The predefined constants `stdin`, `stdout`, and `stderr` have the values 0, 1, and 2 respectively. An implementation assigns every successfully opened external stream a handle other than 0, 1, or 2. Programs must treat all other handle values as opaque and must not depend on their allocation order or reuse.

The normative file-management signatures are:

```text
open(var scalar int handle, scalar string file_name, scalar string mode)
    returns scalar logical
close(scalar int handle) returns scalar logical
```

`mode` must be exactly `"r"`, `"w"`, or `"a"`, denoting reading, replacement writing, or append writing. These are string values rather than reserved words or bare identifiers. On success, `open` stores the new handle in its `var` argument and returns `true`. If the file cannot be opened, the mode is invalid, or another operating-system error occurs, it returns `false` and leaves `handle` unchanged. Such failure does not abort the program.

`close` flushes pending output, releases an external stream, and returns `true` on success. Closing `stdin`, `stdout`, or `stderr` is a successful no-op, so the predefined streams remain available. An invalid or already-closed handle, or a flush or close failure, produces `false` and does not abort the program.

The scalar transfer signatures are variadic:

```text
read(scalar int handle, var scalar T first, var scalar T ...)
    returns scalar logical
readln(scalar int handle, var scalar T first, var scalar T ...)
    returns scalar logical
write(scalar int handle, scalar expression ...)
    returns scalar logical
writeln(scalar int handle, scalar expression ...)
    returns scalar logical
```

Here each occurrence of `T` may independently be `int`, `real`, `logical`, or `string`, and one or more data arguments are required by `read` and `readln`. The output functions permit zero data arguments; `write(handle)` performs no data transfer and succeeds for a valid writable handle, while `writeln(handle)` writes only a line terminator. There are no implicit-handle overloads: the first argument is always an explicit stream handle. `readln` performs the same conversions as `read` and, after successfully obtaining all requested values, consumes through the end of the current input line. `writeln` performs the same formatting as `write` and then appends the implementation's line terminator.

Scalar input is transactional with respect to its destination variables. `read` or `readln` returns `false` and leaves every destination unchanged if the handle is invalid or not readable, end-of-file occurs before all requested values have been obtained, a token is invalid for its target type, or an input error occurs. Otherwise it assigns every destination and returns `true`. End-of-file after a complete request does not retroactively make that request fail.

For an `int` target, input is a whitespace-delimited token consisting of an optional `+` or `-` followed by one or more decimal digits. Source-literal base prefixes, digit separators, and all other characters are invalid. Leading decimal zeros are permitted, and the mathematical value must be in the signed 64-bit range.

For a `real` target, a finite input token consists of an optional `+` or `-`, decimal digits with an optional fractional part, and an optional exponent introduced by `e` or `E`. A token beginning with a decimal point must contain at least one digit after it; otherwise at least one digit must precede the optional point. An exponent consists of an optional sign and one or more decimal digits. Thus `12`, `-3.5`, `.125`, `1.25e6`, and `-4.0E-3` are valid. Base prefixes and digit separators are invalid. Conversion is correctly rounded to binary64 using round-to-nearest, ties-to-even. Overflow produces the correspondingly signed infinity; underflow produces the correctly rounded subnormal value or signed zero. The exact lowercase tokens `infinity`, `+infinity`, `-infinity`, and `nan` are also valid; `nan` produces ASC's canonical NaN.

For a `logical` target, the token must be exactly lowercase `true` or `false`; numeric spellings such as `0` and `1`, differently cased words, and every other token are invalid. A string input value uses the quoted string syntax and escapes specified for `string-constant` in Appendix C.1. Invalid input causes the call to return `false`, not a fatal run-time error.

`write` and `writeln` evaluate their data arguments from left to right. An `int` is formatted in canonical base-10 form: zero is `0`, a negative value has one leading minus sign, and no positive value has a leading sign, leading zero, grouping separator, width padding, or base prefix. A logical value is formatted as exactly `true` or `false`. A string is formatted as a quoted, escaped string that is valid `string-constant` syntax and denotes the original value.

String output uses the following canonical escaping policy. Quotation mark and reverse solidus are written as `\"` and `\\`. Null, tab, line feed, and carriage return are written as `\0`, `\t`, `\n`, and `\r`. Every other control value in U+0000 through U+001F or U+007F through U+009F is written as `\u{h}`, where `h` is the value in lowercase hexadecimal with no unnecessary leading zeros. Every other Unicode scalar value, including non-ASCII values from U+00A0 through U+10FFFF, is written directly as its well-formed UTF-8 encoding. Output does not normalize or otherwise alter Unicode scalar values.

A finite nonzero `real` is formatted as the shortest decimal representation that, when read under the rules above, converts to the identical binary64 value. When more than one shortest significand has that property, the numerically closest is used; an exact tie is resolved toward an even final digit. The implementation chooses the shorter of ordinary decimal notation and scientific notation, choosing ordinary notation when their lengths are equal. Scientific notation uses lowercase `e`, omits a positive exponent sign, and uses no unnecessary exponent leading zeros. It uses one digit before the decimal point and omits the decimal point when no fractional digits remain. Positive and negative zero are written as `0.0` and `-0.0`. Positive infinity, negative infinity, and NaN are written as `infinity`, `-infinity`, and `nan`; NaN output always represents the canonical NaN.

ASC v0.1 does not provide Pascal-style field-width or decimal-place syntax such as `value:width:places`. Formatting does not pad values with spaces. The functions return `false` for an invalid or inappropriate handle or an output error; output completed before an error is not rolled back. I/O failure never aborts the program. Calls execute once in scalar control context, and their externally visible effects occur in program execution order.

The collective parallel transfer signatures are also variadic:

```text
read(scalar int handle, association-name,
     var parallel T field[$], var parallel T ...)
    returns scalar logical
write(scalar int handle, association-name,
      parallel T field[$], parallel T ...)
    returns scalar logical
```

One or more parallel fields must follow the association name. Each field must be a member of that association, and each `T` may independently be `int`, `real`, `logical`, or `string`. General parallel expressions are not permitted as collective `write` fields. The association argument distinguishes these overloads from scalar transfer; it is the association's declared name, not a string value.

A collective `read` treats each input line as one record containing whitespace-separated values in field-argument order. It continues through normal end-of-file, allocates one new association entry for each complete valid record, and returns `true` at normal end-of-file. A blank line is not a record. If an invalid handle, malformed value, incomplete record, allocation failure, or input error occurs, the function returns `false`. Entries created for earlier complete records remain allocated, but no entry is allocated for the incomplete or invalid record.

A collective `write` emits one line per eligible association entry, with fields in argument order and separated by one space. Eligibility is the intersection of association membership and the active responder set at the call. Association entries are unordered, so record order is unspecified and must not be used to identify entries. The function returns `true` after all eligible entries are written, including when there are none. It returns `false` on a handle or output error; complete records written before the error remain externally visible.

Individual fields may still be transferred through an indexed scalar projection inside `get` or `next`. The collective forms are the normative v0.1 counterpart of the Primer's association-oriented parallel I/O, while preserving the modern explicit-handle and success-result conventions. Calls to all predefined routines use the general call grammar in Appendix C.5.

### 9.2 Dynamic record allocation

ASC provides a block form that allocates one association entry:

The normative allocation productions are consolidated in Appendix C.

Examples:

```text
allocate idx in records;
    statements_using(idx)
endallocate;
```

In an `allocate-block`, the index variable introduced after `allocate` is scoped to the block and designates the newly allocated entry.

Allocation creates one complete association row atomically. Before the block begins, every element of every member field in the new row is initialized according to its type: `int` to `0`, `real` to positive zero (`+0.0`), `logical` to `false`, and `string` to the empty string. If a member field has additional dimensions, every element is initialized. If allocation cannot create the complete row, no row is created and the entire program terminates with a fatal run-time error.

The new row is accessible through the block's index variable, but it does not automatically join an active responder set established before the allocation. It becomes eligible when a later responder expression is evaluated in a scope capable of including that association.

ASC provides parallel release:

```text
release conditional_expression from association_name;
```

The condition must produce a logical parallel result. Every matching entry belonging to the named association is removed atomically from every member field and returned to the idle pool. A released row immediately leaves the active responder set. Every index designating that row becomes invalid; any later attempt to use such an index is a fatal run-time error terminating the entire program. The normative production is in Appendix C.5.

### 9.3 Procedures, functions, and calls

ASC v0.1 uses **subroutine** only as the specification's collective term for procedures and functions; `subroutine` is not a source keyword. A procedure returns no value. A programmer-defined function returns one scalar value. Predefined element-wise functions may return scalar or parallel results under the mode-propagation rules in Section 4.3.

Parameters are passed by value unless their declarations begin with `var`.

- A value parameter is initialized from its argument when the routine is entered. It is a local variable that the routine may read or assign; assignments to it do not modify the caller's argument. A parallel value parameter is a value snapshot with the declared parallel mode and dimensions, not a writable view of the caller's field.
- A `var` parameter is an alias for the caller's variable. The argument must be an assignable variable with compatible type, scalar/parallel mode, and dimensions. Reads and assignments through the parameter access the caller's variable directly.

One storage object cannot be passed to more than one `var` parameter in the same call. This prohibition applies to a scalar variable, an entire array, or a parallel field even if the written argument expressions use different subscripts. A compiler must diagnose the conflicting arguments.

Procedures may use `return;` for early return. Functions use `return scalar_expression;`, and every normally reachable function path must return a value compatible with the declared scalar return type.

A procedure is invoked as a statement by writing its name followed by a parenthesized argument list and a semicolon. A function is invoked with the same parenthesized notation in an expression:

```text
update(value, delta);
result = sqrt(value);
ok = writeln(stdout, "result = ", result);
```

An argument for a value parameter may be any compatible expression. An argument for a `var` parameter must be a compatible assignable variable. Argument count, passing mode, type, scalar/parallel mode, and dimensions must match the selected routine declaration or predefined signature. A procedure call does not produce a value and cannot appear in an expression. A function call produces a value and cannot stand alone as a statement. Predefined and programmer-defined routines use the same call syntax.

There are no forward routine declarations in ASC v0.1. A procedure or function may call itself directly, so direct recursion is permitted. A call to any other procedure or function may name only a routine whose complete declaration appears earlier in the source file. Consequently, a recursive call cycle involving two or more routines (for example, `a` calls `b` and `b` calls `a`) is prohibited and must be diagnosed at compile time.

### 9.4 Excluded historical constructs

The Primer documents associative pronouns and articles (`them`, `their`, `its`, `each`, `the`, and `a`), physical inter-PE shifts such as `$+1`, and `andif` / `andfor` complex pattern searches. ASC v0.1 excludes all three feature groups.

Pronouns and articles depend on implicit recent-search or recent-index state and obscure which association an expression accesses. Programs must instead name the parallel condition, field, association, or explicit index variable. The excluded word spellings are not reserved and may be used as ordinary identifiers when they otherwise satisfy the identifier grammar.

Physical inter-PE shifts depend on processor adjacency that the abstract ASC association model does not define and that the `cpu`, `cpu-simd`, and `gpu` targets may realize differently. The `$+n` and `$-n` forms are therefore invalid syntax. Programs must express relationships through association fields and explicit index-based selection.

The Primer gives only skeletal `andif` and `andfor` formats and delegates their actual tree-search semantics to another source. Their implicit fixed-depth search state and restricted bodies do not fit v0.1's explicit control model. The spellings `andif`, `andfor`, `andbody`, `andthen`, `endandif`, `endandfor`, and `endandbody` are not reserved and introduce no constructs. Ordinary nested `if`, responder loops, procedures, and explicit association fields must be used instead.

Absolute storage placement and historical native-language interoperability are also excluded because execution placement is controlled by the compiler target rather than source syntax.

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
8. document implementation limits, including PE capacity and unavailable backends.

### 10.2 Reference semantics

The `cpu` target is the reference for semantic testing. It may implement parallel fields as arrays and active/responder sets as masks, but those representations are not language-visible.

The same conformance suite should run against every backend. Tests must cover at least:

- nested active-set save and restore;
- parallel `if` true and false partitions;
- `elsenany` with empty and nonempty responder sets;
- `any` executing under the incoming active set;
- scalar broadcast and masked parallel assignment;
- left-to-right scalar and masked parallel logical short-circuit evaluation, including skipped fatal operations;
- stable `get` selection versus responder-consuming `next` behavior;
- `for` snapshot versus `while` reevaluation;
- reductions with unique extrema, ties, empty sets, NaNs, signed zeros, and valid and invalid `nth` positions;
- implicit scalar and parallel `int`-to-`real` conversion and decimal-to-binary64 literal rounding;
- integer-literal range checks and valid and invalid Unicode string escapes;
- nested comments, longest-token recognition, unrestricted source-line length, and distinct long identifiers;
- valid and invalid compile-time dimension expressions;
- scalar effects in nonempty and empty masks, including `then`-before-`else` ordering;
- rejection of structural association mutation during `for`, `while`, `get`, and `next` traversal;
- successful and failed scalar and association I/O, relative constant definitions, and zero-based array bounds;
- canonical NaN results across every backend;
- lowercase logical input and output formatting;
- fatal scalar and parallel division- and remainder-by-zero behavior; and
- case sensitivity and parenthesized operator grouping.

### 10.3 Extensions

An implementation extension must be disabled by default or must use syntax that cannot be mistaken for specified ASC. Extensions must not redefine a Required construct. Diagnostics and documentation must identify extension behavior as nonstandard.

## Appendix A. Status summary

| Area | v0.1 status |
|---|---|
| Single instruction stream, active sets, responders | Required |
| Search-process-retrieve model and nested restoration | Required |
| Scalar and parallel modes | Required semantic distinction |
| Primitive value types | `int`, `real`, `string`, and `logical`; scalar and parallel forms required |
| Numeric representation | `int` is signed 64-bit two's complement with wrapping arithmetic; `real` is IEEE 754 binary64 with round-to-nearest, ties-to-even and canonical quiet NaN |
| Numeric conversions | Implicit `int` to `real`; explicit `real` to `int` through `round` or `trunc`; no numeric/string/logical conversions |
| Stream I/O | Explicit integer handles; logical success results; transactional scalar input; association-oriented collective transfer |
| Array indexing | Zero-based length semantics with fatal bounds errors |
| Array dimensions | Positive compile-time `int` expressions with Pascal-like constant-expression use |
| Relative constants | Compile-time references to earlier constants of the same type |
| Responder index type | `index` is a special parallel-only type, not a primitive value type |
| Leading mode qualifier (`parallel int`, `scalar int`) | Required modernization |
| Lowercase case-sensitive keywords and identifiers | Required modernization |
| Source and string text | UTF-8 source and output; strings contain Unicode scalar values with canonical control-character escaping and no normalization |
| Complete lowercase reserved-word set | Required |
| Lexical structure | Longest-token matching, ASCII white space, nested `/* ... */` comments, and no source-line or identifier-length limit |
| Program organization | Exactly one source file and one `main`; no imports, inclusion, or separate compilation |
| Comparisons and logical operators | `== != < <= > >=` and `! && ^^ \|\|` are required modernizations |
| Logical evaluation | Left-to-right scalar and masked parallel short-circuiting for `&&` and `\|\|`; `^^` evaluates both operands |
| Arithmetic `- + * / %` | Required; `%` is a modernization |
| Division or remainder by zero | Fatal run-time error terminating the entire program |
| Bitwise operators | `~ & ^ \|` are required modernizations |
| Parentheses and precedence | Required modernization |
| Compiler `--target=cpu|cpu-simd|gpu` | Required modernization |
| Source-level placement directives | Prohibited |
| Legacy-source compatibility mode | Prohibited; v0.1 accepts only the new ASC syntax |
| Assignment and scalar/parallel `if` | Required core |
| `setscope`, `any` | Required core |
| Scalar and parallel `for` / `while` | Required; parallel visitation order is unspecified |
| Scalar effects in masked constructs | Execute once for each entered nonempty mask, in source-defined control order |
| Structural association mutation during responder traversal | Prohibited in parallel `for`, `while`, `get`, and `next` |
| Scalar and parallel `loop` / `until` | Required |
| `next` | Required grammar; distinct arbitrary responder selection |
| `get` | Required grammar; stable arbitrary responder selection |
| Associations and index retrieval | Required atomic row model and fatal use of an index to a released row |
| Association field membership | Exclusive: each parallel field belongs to at most one association |
| Association allocation and release | Atomic complete-row operations with defined initialization and responder-set effects |
| Minimum, maximum, index, nth, and count reductions | Required operations and edge rules |
| Procedures, functions, parameters, returns, and calls | Required grammar; one object cannot bind multiple `var` parameters in one call |
| Scalar standard I/O | Required handle-based signatures, deterministic formatting, and non-aborting failure semantics |
| Collective parallel I/O | Required association-oriented signatures and record semantics using the scalar text conversions |
| Pronouns/articles, physical shifts, and `andif` / `andfor` | Excluded and not reserved |
| Multiple instruction-stream source semantics | Out of scope |

## Appendix B. Illustrative modernized fragment

This fragment demonstrates confirmed surface syntax and core responder semantics. It is not a claim that every declaration detail is final.

```text
main classify
    scalar int selected_count;
    parallel int value[$];
    parallel int result[$];
    parallel logical rows[$];

    associate value[$], result[$] with rows;

    setscope rows[$]
        if ((value[$] >= 10) && (value[$] < 20)) then
            result[$] = value[$] * 2;
        else
            result[$] = 0;
        endif;

        selected_count = count((value[$] >= 10) && (value[$] < 20));
    endsetscope;
end;
```

## Appendix C. Normative EBNF grammar

This appendix is the single location for normative ASC v0.1 EBNF productions. Productions are ordered from lexical elements through declarations to executable statements. Every nonterminal referenced by the current EBNF has a production in this appendix; semantic areas explicitly identified as unresolved remain subject to later specification.

### C.1 Lexical elements

```ebnf
uppercase-ascii-letter =
    "A" | "B" | "C" | "D" | "E" | "F" | "G" |
    "H" | "I" | "J" | "K" | "L" | "M" | "N" |
    "O" | "P" | "Q" | "R" | "S" | "T" | "U" |
    "V" | "W" | "X" | "Y" | "Z" ;

lowercase-ascii-letter =
    "a" | "b" | "c" | "d" | "e" | "f" | "g" |
    "h" | "i" | "j" | "k" | "l" | "m" | "n" |
    "o" | "p" | "q" | "r" | "s" | "t" | "u" |
    "v" | "w" | "x" | "y" | "z" ;

ascii-letter =
    uppercase-ascii-letter | lowercase-ascii-letter ;

white-space-character =
    ? U+0020 SPACE ? |
    ? U+0009 CHARACTER TABULATION ? |
    ? U+000C FORM FEED ? |
    ? U+000A LINE FEED ? |
    ? U+000D CARRIAGE RETURN ? ;

comment =
    "/*", { comment | comment-character }, "*/" ;

comment-character =
    ? any Unicode scalar value except one beginning the delimiters /* or */ ? ;

nonzero-decimal-digit =
    "1" | "2" | "3" | "4" | "5" |
    "6" | "7" | "8" | "9" ;

decimal-digit =
    "0" | nonzero-decimal-digit ;

decimal-digits =
    decimal-digit, { decimal-digit } ;

positive-decimal-integer =
    nonzero-decimal-digit, { decimal-digit } ;

binary-digit =
    "0" | "1" ;

octal-digit =
    "0" | "1" | "2" | "3" |
    "4" | "5" | "6" | "7" ;

hexadecimal-digit =
    decimal-digit |
    "a" | "b" | "c" | "d" | "e" | "f" |
    "A" | "B" | "C" | "D" | "E" | "F" ;

sign =
    "+" | "-" ;

boolean-constant =
    "true" | "false" ;

decimal-integer =
    "0" | positive-decimal-integer ;

binary-integer =
    "0b", binary-digit, { binary-digit } ;

octal-integer =
    "0o", octal-digit, { octal-digit } ;

hexadecimal-integer =
    "0x", hexadecimal-digit, { hexadecimal-digit } ;

integer-magnitude =
    decimal-integer |
    binary-integer |
    octal-integer |
    hexadecimal-integer ;

integer-constant =
    [ sign ], integer-magnitude ;

exponent-part =
    ( "e" | "E" ), [ sign ], decimal-digits ;

fractional-real =
    decimal-digits, ".", [ decimal-digits ], [ exponent-part ] |
    ".", decimal-digits, [ exponent-part ] ;

exponent-real =
    decimal-digits, exponent-part ;

real-constant =
    [ sign ], ( fractional-real | exponent-real ) ;

quotation-mark =
    ? U+0022 QUOTATION MARK ? ;

backslash =
    ? U+005C REVERSE SOLIDUS ? ;

raw-string-character =
    ? any Unicode scalar value except U+0022, U+005C, U+000A, or U+000D ? ;

simple-string-escape =
    backslash,
    ( quotation-mark | backslash | "n" | "r" | "t" | "0" ) ;

unicode-string-escape =
    backslash, "u", "{", hexadecimal-digit,
    [ hexadecimal-digit,
      [ hexadecimal-digit,
        [ hexadecimal-digit,
          [ hexadecimal-digit,
            [ hexadecimal-digit ] ] ] ] ], "}" ;

string-character =
    raw-string-character |
    simple-string-escape |
    unicode-string-escape ;

string-constant =
    quotation-mark, { string-character }, quotation-mark ;

identifier =
    ascii-letter, { ascii-letter | decimal-digit | "_" } ;
```

An `ascii-letter` is one character in `A` through `Z` or `a` through `z`. The `comment` production is recursive and therefore permits nesting; its delimiters are recognized before `comment-character`. Reserved words are excluded from `identifier` by the lexical rules in Section 4.2. In the `boolean-constant` production, `true` and `false` are lexed as identifiers and resolved to their predefined constant bindings.

The complete signed value denoted by `integer-constant` must fit the `int` range; this semantic range check applies after interpreting the magnitude in its written base. String escape results are: `\"` for U+0022, `\\` for U+005C, `\n` for U+000A, `\r` for U+000D, `\t` for U+0009, and `\0` for U+0000. A `unicode-string-escape` contains one through six hexadecimal digits and must denote a Unicode scalar value no greater than U+10FFFF and outside the surrogate range U+D800 through U+DFFF.

### C.2 Constant definitions

```ebnf
integer-definition-value =
    integer-constant |
    identifier, ( "+" | "-" ), integer-constant ;

real-definition-value =
    real-constant |
    identifier, ( "+" | "-" ), real-constant ;

logical-constant-definition =
    "deflog", "(", identifier, ",", boolean-constant, ")", ";" ;

integer-constant-definition =
    "defint", "(", identifier, ",", integer-definition-value, ")", ";" ;

real-constant-definition =
    "defreal", "(", identifier, ",", real-definition-value, ")", ";" ;

string-constant-definition =
    "defstr", "(", identifier, ",", string-constant, ")", ";" ;

constant-definition =
    logical-constant-definition |
    integer-constant-definition |
    real-constant-definition |
    string-constant-definition ;
```

In `integer-definition-value`, the relative form's leading `identifier` must name an earlier `defint` constant. In `real-definition-value`, it must name an earlier `defreal` constant. Relative definitions are evaluated at compile time; integer arithmetic wraps modulo 2^64 and real arithmetic follows IEEE 754 binary64. No runtime variable is permitted.

### C.3 Variable declarations

```ebnf
data-type =
    primitive-data-type | index-data-type ;

primitive-data-type =
    "int" | "real" | "string" | "logical" ;

index-data-type =
    "index" ;

dimension-primary =
    integer-constant |
    identifier |
    "(", dimension-expression, ")" ;

dimension-unary-expression =
    { "+" | "-" | "~" }, dimension-primary ;

dimension-multiplicative-expression =
    dimension-unary-expression,
    { ( "*" | "/" | "%" ), dimension-unary-expression } ;

dimension-additive-expression =
    dimension-multiplicative-expression,
    { ( "+" | "-" ), dimension-multiplicative-expression } ;

dimension-bitwise-and-expression =
    dimension-additive-expression,
    { "&", dimension-additive-expression } ;

dimension-bitwise-xor-expression =
    dimension-bitwise-and-expression,
    { "^", dimension-bitwise-and-expression } ;

dimension-expression =
    dimension-bitwise-xor-expression,
    { "|", dimension-bitwise-xor-expression } ;

dimension =
    dimension-expression ;

scalar-dimensions =
    "[", dimension, { ",", dimension }, "]" ;

parallel-dimensions =
    "[", "$", { ",", dimension }, "]" ;

scalar-declaration =
    "scalar", primitive-data-type, identifier, [ scalar-dimensions ], ";" ;

parallel-declaration =
    "parallel", data-type, identifier, parallel-dimensions, ";" ;

variable-declaration =
    scalar-declaration | parallel-declaration ;
```

The spellings in `primitive-data-type` are lexed as identifiers and resolved to predefined type bindings; they are not keyword tokens. An identifier in a `dimension-primary` must name an earlier `defint` constant. Each `dimension` is evaluated at compile time, must be greater than zero, and defines the zero-based range 0 through `dimension - 1`. The semantic restrictions on identifiers and dimensions are specified in Sections 4.4 and 5.2.

### C.4 Association declarations

```ebnf
association-name =
    identifier ;

parallel-field-reference =
    identifier, "[", "$", { ",", dimension }, "]" ;

association-item =
    parallel-field-reference ;

association-declaration =
    "associate", association-item, { ",", association-item },
    "with", association-name, ";" ;
```

An association declaration contains at least one `association-item`. Additional items are comma-separated, and there is no trailing comma before `with`. Every item must name a declared parallel field of primitive type `int`, `real`, `logical`, or `string`; a `parallel index` cannot be an association item. The `association-name` must name a declared `parallel logical` field. An item cannot occur more than once in one declaration, and a parallel field named as an item cannot be named as an item by any other association declaration. A compiler must diagnose any violation.

### C.5 Conditional expressions and common statement elements

The variable/value-expression productions and the statement/block productions are mutually recursive. Within each recursive group, no linear ordering can place every rule before every use; the rules are kept together and ordered from lexical references through precedence layers to their public entry productions.

```ebnf
index-variable =
    identifier ;

constant-reference =
    identifier ;

scalar-variable =
    identifier, [ scalar-subscripts ] ;

scalar-subscripts =
    "[", scalar-index-expression,
    { ",", scalar-index-expression }, "]" ;

parallel-variable =
    identifier, "[", "$",
    { ",", scalar-index-expression }, "]" ;

parallel-element-reference =
    identifier, "[", index-variable,
    { ",", scalar-index-expression }, "]" ;

scalar-index-expression =
    scalar-expression ;

argument-list =
    "(", [ expression, { ",", expression } ], ")" ;

function-call-expression =
    identifier, argument-list ;

value-primary =
    integer-constant |
    real-constant |
    string-constant |
    function-call-expression |
    constant-reference |
    scalar-variable |
    parallel-variable |
    parallel-element-reference |
    "(", value-expression, ")" ;

value-unary-expression =
    { "+" | "-" | "~" }, value-primary ;

multiplicative-expression =
    value-unary-expression,
    { ( "*" | "/" | "%" ), value-unary-expression } ;

additive-expression =
    multiplicative-expression,
    { ( "+" | "-" ), multiplicative-expression } ;

bitwise-and-expression =
    additive-expression, { "&", additive-expression } ;

bitwise-xor-expression =
    bitwise-and-expression, { "^", bitwise-and-expression } ;

bitwise-or-expression =
    bitwise-xor-expression, { "|", bitwise-xor-expression } ;

value-expression =
    bitwise-or-expression ;

scalar-value-expression =
    value-expression ;

parallel-value-expression =
    value-expression ;

comparison-operator =
    "<" | ">" | "<=" | ">=" | "==" | "!=" ;

comparison-expression =
    value-expression, comparison-operator, value-expression ;

scalar-logical-variable =
    scalar-variable ;

parallel-logical-variable =
    parallel-variable ;

logical-parallel-element-reference =
    parallel-element-reference ;

logical-value-expression =
    scalar-logical-variable |
    parallel-logical-variable |
    logical-parallel-element-reference ;

logical-primary =
    boolean-constant |
    function-call-expression |
    logical-value-expression |
    comparison-expression |
    "(", conditional-expression, ")" ;

logical-not-expression =
    { "!" }, logical-primary ;

logical-and-expression =
    logical-not-expression, { "&&", logical-not-expression } ;

logical-xor-expression =
    logical-and-expression, { "^^", logical-and-expression } ;

logical-or-expression =
    logical-xor-expression, { "||", logical-xor-expression } ;

conditional-expression =
    logical-or-expression ;

scalar-conditional-expression =
    conditional-expression ;

parallel-conditional-expression =
    conditional-expression ;

logical-parallel-expression =
    parallel-conditional-expression ;

scalar-expression =
    scalar-value-expression | scalar-conditional-expression ;

parallel-expression =
    parallel-value-expression | parallel-conditional-expression ;

expression =
    scalar-expression | parallel-expression ;

assignment-target =
    scalar-variable |
    parallel-variable |
    parallel-element-reference ;

assignment-statement =
    assignment-target, "=", expression, ";" ;

procedure-call-statement =
    identifier, argument-list, ";" ;

release-statement =
    "release", conditional-expression,
    "from", association-name, ";" ;

procedure-return-statement =
    "return", ";" ;

function-return-statement =
    "return", scalar-expression, ";" ;

return-statement =
    procedure-return-statement | function-return-statement ;

simple-statement =
    assignment-statement |
    procedure-call-statement |
    release-statement |
    return-statement ;

statement-block =
    { statement } ;

statement =
    allocation-statement |
    if-statement |
    any-statement |
    loop-statement |
    get-statement |
    next-statement |
    setscope-statement |
    simple-statement ;
```

`scalar-variable` must name a declared scalar object. Its optional subscripts must match that object's declared scalar dimensions. `parallel-variable` must name a declared parallel object; `$` denotes its whole PE dimension. Every additional subscript must be scalar and integral. Each ordinary subscript must be in the zero-based range of its corresponding dimension; an out-of-bounds subscript is a fatal run-time error, and one out-of-bounds active parallel entry terminates the entire program.

`parallel-element-reference` must name a declared parallel field and use a visible `parallel index` as its first subscript. The index variable must currently flag one entry selected by an enclosing `get`, `next`, parallel `for`, or parallel `while`, and the referenced field must belong to the association selected by that index. The reference selects that entry and therefore produces a scalar value. When the field is assignable, the reference is an assignable scalar target. Every additional subscript must be scalar and integral; for example, `matrix[idx, row, column]` selects `row, column` within the association entry flagged by `idx`. The projection is valid only within the lexical body in which the index is bound.

`constant-reference` must name a visible typed constant. A value expression is scalar if it contains only constants, scalar variables, and parallel element references. It is parallel if it contains at least one whole `parallel-variable`; compatible scalar operands are broadcast. The aliases `scalar-value-expression` and `parallel-value-expression` express these semantic subsets of the shared syntax.

A `logical-value-expression` must name or retrieve a value of `logical` type. A `comparison-expression` produces a logical result. Logical operands are valid with `==` and `!=` but invalid with `<`, `<=`, `>`, and `>=`. Arithmetic and bitwise operand compatibility is determined by the type system.

The scalar and parallel conditional productions share identical token syntax. Semantic analysis classifies a condition as scalar when all of its operands are scalar. A condition is parallel when at least one operand is parallel; compatible scalar operands are broadcast across the current active set. Mixing incompatible scalar and parallel operands is a semantic error.

An `index-variable` must resolve to a declared `parallel index` object visible at that point.

A `statement-block` may be empty. In a responder-masked block, the entire block is skipped when its active set is empty; otherwise each scalar statement executes once and each parallel statement applies to the active entries. Bodies chosen by the scalar existence decisions of `any` and `elsenany` execute once even when their restored incoming active set is empty. An assignment does not produce a value and therefore cannot be nested inside another expression. A scalar target requires a compatible scalar expression. A whole parallel target accepts a compatible parallel expression or a scalar expression broadcast to the current active set. A `parallel-element-reference` is a scalar target.

In a `procedure-call-statement`, the identifier must resolve to a procedure. In a `function-call-expression`, it must resolve to a function, and semantic analysis classifies the expression by the function's result type and mode. An argument corresponding to a `var` parameter must satisfy `assignment-target`; an argument for a value parameter may be any compatible expression. The same declared storage object cannot supply more than one `var` argument in one call, including through different subscripts. In a `release-statement`, `conditional-expression` must produce a logical parallel result and `association-name` denotes the association from which matching entries are released. A procedure accepts only `procedure-return-statement`; a function accepts only `function-return-statement`.

### C.6 Allocation statements

```ebnf
allocate-block =
    "allocate", index-variable, "in", association-name, ";",
    statement-block,
    "endallocate", ";" ;

allocation-statement =
    allocate-block ;
```

`index-variable` and `statement-block` are defined in Appendix C.5. `association-name` is defined in Appendix C.4. The index variable introduced by `allocate` is visible only within the allocation block. Allocation is an atomic complete-row operation with the initialization and failure behavior specified in Section 9.2.

### C.7 If statements

```ebnf
scalar-if-statement =
    "if", scalar-conditional-expression, "then",
    statement-block,
    [ "else", statement-block ],
    "endif", ";" ;

parallel-if-alternative =
    "else", statement-block |
    "elsenany", statement-block ;

parallel-if-statement =
    "if", parallel-conditional-expression, "then",
    statement-block,
    [ parallel-if-alternative ],
    "endif", ";" ;

if-statement =
    scalar-if-statement | parallel-if-statement ;
```

`scalar-conditional-expression`, `parallel-conditional-expression`, and `statement-block` are defined in Appendix C.5. The condition's mode determines which `if` form applies. A scalar condition permits only `else`; a parallel condition permits either `else` or `elsenany`, but not both. When both responder partitions of a parallel `if` are nonempty, the `then` block, including its scalar effects, executes before the `else` block.

### C.8 Any statement

```ebnf
any-statement =
    "any", conditional-expression,
    statement-block,
    [ "elsenany", statement-block ],
    "endany", ";" ;
```

`conditional-expression` and `statement-block` are defined in Appendix C.5.

### C.9 Loop statements

```ebnf
scalar-until-clause =
    "until", scalar-expression ;

parallel-until-clause =
    "until", [ "nany" ], parallel-expression ;

until-clause =
    scalar-until-clause | parallel-until-clause ;

loop-until-statement =
    "loop", statement-block, until-clause, "endloop", ";" ;

scalar-for-direction =
    "to" | "downto" ;

scalar-for-statement =
    "for", scalar-variable, "=", scalar-expression,
    scalar-for-direction, scalar-expression,
    statement-block,
    "endfor", ";" ;

scalar-while-statement =
    "while", "(", scalar-conditional-expression, ")",
    statement-block,
    "endwhile", ";" ;

scalar-loop-statement =
    scalar-for-statement | scalar-while-statement ;

parallel-for-statement =
    "for", index-variable, "in", parallel-conditional-expression,
    statement-block,
    [ "elsenany", statement-block ],
    "endfor", ";" ;

parallel-while-statement =
    "while", index-variable, "in", parallel-conditional-expression,
    statement-block,
    "endwhile", ";" ;

parallel-loop-statement =
    parallel-for-statement | parallel-while-statement ;

loop-statement =
    loop-until-statement |
    scalar-loop-statement |
    parallel-loop-statement ;
```

`scalar-variable`, `scalar-expression`, `parallel-expression`, `scalar-conditional-expression`, `parallel-conditional-expression`, `index-variable`, and `statement-block` are defined in Appendix C.5. The expression in each `until` clause must have logical type. An index variable introduced by a parallel `for` or `while` is visible only within that loop's main statement block. It is not visible in a parallel `for` statement's optional `elsenany` block because that block executes only when no entry was selected. A parallel-loop body must not allocate or release a row of an association traversed by its condition or perform a collective `read` into that association; such a structural modification is a compile-time error.

### C.10 `get` and `next` statements

```ebnf
get-statement =
    "get", index-variable, "in", parallel-conditional-expression,
    statement-block,
    [ "elsenany", statement-block ],
    "endget", ";" ;

next-statement =
    "next", index-variable, "in", parallel-variable,
    statement-block,
    [ "elsenany", statement-block ],
    "endnext", ";" ;
```

`parallel-conditional-expression`, `parallel-variable`, `index-variable`, and `statement-block` are defined in Appendix C.5. An index variable introduced by `get` or `next` is visible only within that construct's main block; it is not visible in the optional `elsenany` block because that block executes only when no entry was selected.

`get` selects an arbitrary responder without changing the responder set. A nested `get` with no intervening association or responder-set modification selects the same entry. `next` requires a `parallel-variable` and updates the responder set so nested `next` operations select distinct entries; selection does not assign to the parallel variable. Neither construct defines an association-entry order. If no responder exists, the main block is skipped and the optional `elsenany` block executes once using the incoming active set. The main block of either construct must not allocate or release a row of the traversed association or perform a collective `read` into it; such a structural modification is a compile-time error.

### C.11 Set-scope statement

```ebnf
setscope-statement =
    "setscope", logical-parallel-expression,
    statement-block,
    "endsetscope", ";" ;
```

`logical-parallel-expression` and `statement-block` are defined in Appendix C.5. The expression is evaluated in the incoming active set; the resulting responders form the active set for `statement-block`, and the incoming active set is restored after `endsetscope`.

### C.12 Compilation units, procedures, and functions

```ebnf
scalar-parameter =
    [ "var" ], "scalar", primitive-data-type, identifier,
    [ scalar-dimensions ] ;

parallel-parameter =
    [ "var" ], "parallel", data-type, identifier,
    parallel-dimensions ;

parameter =
    scalar-parameter | parallel-parameter ;

parameter-list =
    "(", [ parameter, { ",", parameter } ], ")" ;

declaration =
    constant-definition |
    variable-declaration |
    association-declaration ;

declaration-section =
    { declaration } ;

scalar-return-type =
    "scalar", primitive-data-type ;

main-program =
    "main", identifier,
    declaration-section,
    statement-block,
    "end", ";" ;

procedure-declaration =
    "procedure", identifier, parameter-list,
    declaration-section,
    statement-block,
    "endprocedure", ";" ;

function-declaration =
    "function", identifier, parameter-list,
    "returns", scalar-return-type,
    declaration-section,
    statement-block,
    "endfunction", ";" ;

top-level-unit =
    main-program |
    procedure-declaration |
    function-declaration ;

compilation-unit =
    top-level-unit, { top-level-unit } ;
```

A compilation unit is the complete contents of one ASC source file and contains at least one top-level unit. It constitutes the entire program; ASC v0.1 does not link multiple compilation units. Procedures and functions cannot be nested. Exactly one `main-program` is required. Declarations must precede the `statement-block` in each top-level unit and must be visible before use.

A routine's own name is visible within its body, permitting direct recursion. Any other called routine must have a complete declaration earlier in the compilation unit; ASC v0.1 has no forward routine declaration. A call-graph cycle containing more than one routine is prohibited and must be diagnosed at compile time.
