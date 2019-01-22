# Typed Function References

[To be moved into its own proposal repo.]

This proposal adds function references that are typed and can be called directly. Unlike `funcref` and the existing `call_indirect` instruction, typed function references need not be stored into a table to be called (though they can), they cannot be null, and a call through them does not require any runtime check. A typed function reference can be formed from any function index.

In addition to instructions for producing and consuming function references, the proposal also adds an instruction for forming a *closure* from a function reference, which takes a prefix of the function's arguments and returns a new function reference with those parameters bound. (Hence, conceptually, all function references are closures of 0 or more parameters.)

Typed references have no canonical default value, because they cannot be null. To enable storing them in locals, which so far depend on default values for initialisation, the proposal also introduces a new instruction `let` for block-scoped locals whose initialisation values are taken from the operand stack.


## Language

Based on [reference types proposal](https://github.com/WebAssembly/reference-types), which introduces type `anyref` and `funcref`.


### Types

#### Value Types

* `ref <typeidx>` is a new reference type
  - `reftype ::= ... | ref <typeidx>`
  - `ref $t ok` iff `$t` is defined in the context

Question: Include optref as well?


#### Subtyping

* Any function reference type is a subtype of `funcref`
  - `ref $t <: funcref`
     - iff `$t = <functype>`


#### Defaultability

* Any numeric value type is defaultable (to 0)

* A reference value type is defaultable (to `null`) if it is not of the form `ref $t`

* Function-level locals must have a type that is defaultable.

* Table definitions with non-zero minimum size must have an element type that is defaultable. (Imports are not affected.)


### Instructions

#### Functions

* `ref.func` creates a function reference from a function index
  - `ref.func $f : [] -> [(ref $t)]`
     - iff `$f : $t`
  - this is a *constant instruction*

* `call_ref` calls a function through a reference
  - `call_ref : [t1* (ref $t)] -> [t2*]`
     - iff `$t = [t1*] -> [t2*]`

* `func.bind` creates or extends a closure by binding one or several parameters
  - `func.bind $t : [t1^n (ref $t)] -> [(ref $t')]`
    - iff `$t = [t1^n t1'*] -> [t2*]`
    - and `$t' = [t1'*] -> [t2*]`

Questions:
- The naming conventions for these instructions seem rather incoherent, are there better ones?
- The requirement to provide type `$t'` instead of just a number is a hack to side-step the issue of expressing an anonymous function type. Should we try better?


#### Local Bindings

* `let <blocktype> (local <valtype>)* <instr>* end` locally binds operands to variables
  - `let bt (local t)* instr* end : [t* t1*] -> [t2*]`
    - iff `bt = [t1*] -> [t2*]`
    - and `instr* : bt` under a context with `locals` extended by `t*` and `labels` extended by `[t2*]`


## Binary Format

TODO.


## JS API

Based on the JS type reflection proposal.

### Type Representation

* A `ValueType` can be described by an object of the form `{ref: DefType}` and `{optref: DefType}`
  - `type ValueType = ... | {ref: DefType} | {optref: DefType}`


### Value Conversions

#### Reference Types

In addition to the rules for basic reference types:

* Any function that is an instance of `WebAssembly.Function` with type `<functype>` is allowed as `ref <functype>`.


### Constructors

#### `Global`

* `TypeError` is produced if the `Global` constructor is invoked without a value argument but a type that is not defaultable.

#### `Table`

* The `Table` constructor gets an additional optional argument `init` that is used to initialise the table slots. It defaults to `null`. A `TypeError` is produced if the argument is omitted and the table's element type is not defaultable.
