+++
title = "The data structures of a compiler"
date = 2025-10-13
+++

In 2024 I began work on [Halcyon](https://halcyon-lang.dev), a programming language based on ML.
Heres a fizzbuzz program written in Halcyon:
```
module example =
  let fizzbuzz = fn number max =>
    (match (number % 3, number % 5) with
      | (0, 0) => "FizzBuzz"
      | (0, _) => "Fizz"
      | (_, 0) => "Buzz"
      | (_, _) => format::integer number)
	|> std::println;
    if number < max then
      fizzbuzz (number + 1) max
    else
      ()

  do fizzbuzz 1 30
end
```
Each part of the compiler went through dozens of iterations before reaching its current form.
In retrospect, the most difficult part to get right was the data structures.
This will be the guide I wish I had at the start: a complete breakdown of every data structure I use in my compiler.

## Primitives
### Span
Most data structures reference some part of the source code.
I use the `Span` struct to represent a section of the source code.
```rust
struct Span {
  // First character of the span
  start: usize,
  // Number of characters in the span
  width: usize,
}
```
I chose this representation because it is the hardest to get *wrong*.
For example, if I had stored the `start` and `end` positions, I open up the possibility for bugs such as `end < start`.
This representation is much easier to work with than line and column numbers for operations like combining spans, and so I do that conversion at the very end.
A span may also extend past the end of the source file.
This is useful for error reporting, like when an opening `(` has no closing `)`.
The place where `)` is expected might be *after* the end of the file.

Very commonly, I want to attach a span to another data type.
I created a wrapper type `Spanned<T>` for this use case.
```rust
struct Spanned<T> {
  inner: T,
  span: Span,
}
```

### Log
Error reporting is the primary user interface of your compiler.
It is important to take errors into consideration from the very beginning.
The `Log` struct represents error, warning, and info messages to report.
```rust
enum Severity {
  Debug,
  Warn,
  Error,
}

struct Log {
  severity: Severity,
  span: Option<Span>, 
  message: String,
}
```

When an error is encountered, compilation will continue if possible.
All errors are collected into a vector, and presented to the user at the end.

## Tokens
Most languages will have dozens of different kinds of tokens.
Listing every token kind I use would take up too much space and not be informative, so I provided an abbreviated list.
```rust
type Token = Spanned<TokenKind>;

enum TokenKind {
    // Symbols
    Plus,
    Minus,
    Star,
    Slash,
    /* ... */

    // Keywords
    Let,
    Type,
    /* ... */

    // Literals
    Identifier(String),
    StringLiteral(String),
    GlyphLiteral(char),
    IntegerLiteral(String, Base),
    RealLiteral(String),

    // Extras
    LineComment(String),
    BlockComment(String),
    DocComment(String),
    Error,
}

enum Base {
    Binary = 2,
    Octal = 8,
    Decimal = 10,
    Hex = 16,
}
```
The integer and float (real) literals are not fully parsed during tokenization.
There are a few reasons you may want to do this.
Firstly, it is not clear at this stage what data type is actually appropriate.
If your language has multiple kinds of integers and floats, choosing the correct precision and signedness is important.
Secondly, parsing numbers later in compilation allows for better error messages, since more information about the program will be available.

Everything in the "Extras" section is filtered out before parsing, and some compilers may not represent these as tokens at all.
I keep these in because they are useful for tools that want to inspect the code.
In Particular, LSPs and documentation generators care about comments.

The `Error` token helps report better errors during parsing, like in the case `1m + 1`.
`1m` is not a valid token, but if I ignore it, I am left with effectively `+ 1`, which looks like the user intended to use `+` as a prefix unary operator, when that is clearly not the case.

## Parsing
The next step is to parse an array of tokens into a tree data structure.
I use different types to represent statements, expressions, types, and patterns.
This section will be easier to follow if the reader has a basic understanding of [Halcyon's grammar](https://halcyon-lang.dev/language_reference/anatomy.html).
Starting from the top down, modules are represented as such:
```rust
struct InnerParsedModule {
  name: Spanned<String>,
  contents: Vec<ModuleStatement>
}

type ParsedModule = Spanned<InnerParsedModule>;

enum ModuleStatementKind {
    Let {
        assignee: PatternExpression,
        value: Box<ValueExpression>,
    },
    Do(Box<ValueExpression>),
    Type {
        assignee: Spanned<String>,
        value: Box<TypeDefinition>,
    },
}

type ModuleStatement = Spanned<ModuleStatementKind>;
```
Notice that every type has an associated span.
This is necessary everywhere that an error could still be lurking.
There are many kinds of errors that parsing cannot check, so I am keeping the spans around.

### Value expressions
I call expressions that evaluate to something a "value expression".
They are represented as a tree data structure.
The `BinaryOp` and `UnaryOp` types are not interesting, they are simply a subset of the `Token` enum.
```rust
enum ValueExpressionKind {
    // let a = b in ...
    Let {
        assignee: PatternExpression,
        value: Box<ValueExpression>,
        in_: Box<ValueExpression>,
    },
    Literal(Literal),
    Identifier(String),
    // a::b
    ModulePath(Vec<String>),
    Binary {
        op: BinaryOp,
        left: Box<ValueExpression>,
        right: Box<ValueExpression>,
    },
    // A binary operator used as a function, as in `( + )`
    BinaryOp(BinaryOp),
    Unary {
        op: UnaryOp,
        child: Box<ValueExpression>,
    },
    // A unary operator used as a function, as in `( not )`
    UnaryOp(UnaryOp),
    FunctionDef {
        parameters: Vec<Spanned<String>>,
        types: Vec<Option<TypeExpression>>,
        body: Box<ValueExpression>,
    },
    // Shorthand function syntax
    FunctionShorthand {
        predicates: Vec<PatternExpression>,
        branches: Vec<ValueExpression>,
    },
    FunctionCall {
        callee: Box<ValueExpression>,
        argument: Box<ValueExpression>,
    },
    If {
        predicate: Box<ValueExpression>,
        then: Box<ValueExpression>,
        else_: Box<ValueExpression>,
    },
    Match {
        scrutinee: Box<ValueExpression>,
        predicates: Vec<PatternExpression>,
        branches: Vec<ValueExpression>,
    },
    Tuple(Vec<ValueExpression>),
    Array(Vec<ArrayInner>),
    StructureLiteral {
        lhs: Vec<String>,
        rhs: Vec<ValueExpression>,
    },
    // a.b
    Field {
        lhs: Box<ValueExpression>,
        rhs: String,
    },
}

type ValueExpression = Spanned<ValueExpressionKind>;

enum Literal {
    Unit, // ()
    Integer(String, Base),
    Real(String),
    String(String),
    Glyph(char),
    Boolean(bool),
}

enum ArrayInner {
    // Inline array expansion
    Splat(ValueExpression),
    // Normal array element
    Single(ValueExpression),
}
```

### Pattern expressions
Patterns are used in `let` and `match` expressions.
They can bind variables, unwrap algebraic data types, and provide type hints.
Because patterns can also be recursive, they are represented by another tree data structure.
Array patterns are a special enough case that they get their own node type for their variations.
```rust
enum PatternExpressionKind {
    Literal(super::Literal),
    Identifier(String),
    ModulePath(Vec<String>),
    Tuple(Vec<PatternExpression>),
    Array(Box<ParsedArrayPattern>),
    Constructor(Vec<String>, Box<PatternExpression>),
    // a : integer
    TypeHint(Box<PatternExpression>, Box<TypeExpression>),
}

type PatternExpression = Spanned<PatternExpressionKind>;

enum ParsedArrayPattern {
    // [a, b, c]
    Exact(Vec<PatternExpression>),
    // [a, b, ..c]
    Leading {
        head: Vec<PatternExpression>,
        tail: Option<String>,
    },
    // [..a, b, c]
    Trailing {
        head: Option<String>,
        tail: Vec<PatternExpression>,
    },
    // [a, ..b, c]
    LeadingAndTrailing {
        head: Vec<PatternExpression>,
        middle: Option<String>,
        tail: Vec<PatternExpression>,
    },
}
```

### Type definitions and expressions
Parsed types are separated into definitions and expressions.
There are several important differences between the two:
* Type definitions must be given an explicit name using a `type` statement.
* Only type definitions may be polymorphic.
* Only type definitions (and specifically only sum types) may recursively refer to themselves.
* Only type expressions may be used as type hints/annotations.

This distinction is really only important for ML style languages.
In general, the representation of types in a language is highly dependent on what features its type system has.

```rust
enum TypeDefinitionKind {
    // Parametric polymorphic type
    TypeFunction {
        arguments: Vec<String>,
        body: Box<TypeDefinition>,
    },
    // { .. }
    Structure {
        lhs: Vec<String>,
        rhs: Vec<TypeExpression>,
    },
    // | A of B | C of D
    Sum {
        variant_names: Vec<String>,
        variant_types: Vec<Option<TypeExpression>>,
    },
    Expression(TypeExpression),
}

type TypeDefinition = Spanned<TypeDefinitionKind>;

enum TypeExpressionKind {
    // a -> b
    Function(Box<TypeExpression>, Box<TypeExpression>),
    // a b
    Call(Box<TypeExpression>, Box<TypeExpression>),
    Identifier(String),
    // (a, b)
    Product(Vec<TypeExpression>),
    // a::b
    ModulePath(Vec<String>),
    // [a]
    Array(Box<TypeExpression>),
    // ()
    Unit,
}

type TypeExpression = Spanned<TypeExpressionKind>;
```

## Intermediate representation
The purpose of an intermediate representation (or IR) is to strip down the parse tree into a more minimal and normalized form.
Several simplifications are performed: 
* All identifiers and paths are normalized into globally unique, fully qualified `Path`'s.
* Binary and unary operators are reduced to function calls.
* Array literals are converted into a series of `push` (for normal items) and `append` (for inline expansion) calls.
* Literals are fully parsed into the `ConstValue` struct.
* `PatternExpression`s are converted to a typed `Pattern` struct which is mostly the same.
```rust
enum IrKind {
    Let {
        assignee: Pattern,
        value: Box<IrNode>,
        in_: Box<IrNode>,
    },
    Immediate(ConstValue),
    Identifier(Path),
    Tuple(Vec<IrNode>),
    Struct {
        field_names: Vec<String>,
        field_values: Vec<IrNode>,
    },
    Field {
        of: Box<IrNode>,
        index: String,
    },
    Function {
        parameter_name: Option<Spanned<Path>>,
        parameter_type: Option<Type>,
        captures: Vec<Path>,
        capture_types: Vec<Type>,
        body: Box<IrNode>,
    },
    Call {
        callee: Box<IrNode>,
        argument: Box<IrNode>,
        opt: optimize::CallOptimization,
    },
    If {
        predicate: Box<IrNode>,
        then: Box<IrNode>,
        else_: Box<IrNode>,
    },
    Match {
        scrutinee: Box<IrNode>,
        predicates: Vec<Pattern>,
        branches: Vec<IrNode>,
    },
}

type IrNode = Spanned<Typed<IrNode>>;

enum ConstValue {
    Unit,
    Integer(i64),
    Real(f64),
    Boolean(bool),
    String(String),
    Glyph(char),
}

enum PatternKind {
    // '_' is a "hole", or an intentionally ignored identifier,
    // such as in `let _ = foo`
    Hole,
    Name(Path),
    Tuple(Vec<Pattern>),
    Array(ArrayPattern),
    Constructor(Constructor, Box<Pattern>),
    Literal(ConstValue),
    TypeHint(Box<Pattern>, Type),
}
```
### Paths
A path is an unambiguous name for a variable or type.
It consists of the name of the module it is defined in, the name of the thing itself, and optionally a salt (see name spaces).
I represent paths as just a thin wrapper over the `String` type.
```rust
struct Path(String);
```

### Name spaces
A namespace is used to convert identifiers into a canonical, fully qualified `Path`.
It is responsible for handling scope, or lifetime, of variables.
Halcyon has three different namespaces:
* `let` definitions
* `type` definitions
* `module` definitions

The language's syntax ensures that it is never ambiguous *which* namespace an identifier belongs to.
This means that a source code can have `type foo`, `let foo`, and `module foo` without any collisions.

A namespace is responsible for handling the scopes of both global and local names.
Some languages (not Halcyon) also have arbitrarily nested local scopes.
When a new name is introduced that is the same as a previous name, the result is either a collision or shadowing.
Name collisions must be reported as errors, and a shadowed name temporarily replaces an older name.
The exact rules for when shadowing is allowed is a language design decision.

The standard way of representing a namespace is a dictionary paired with a vector of "events".
When a name leaves scope, the namespace's previous state is restored by popping an event from the vector.
Below is simplified version of Halcyon's namespace struct:
```rust
struct NameEvent {
  old_name: String,
  old_path: Option<Path>,
}

struct NameSpace {
  salt: usize,
  name_lookup: HashMap<String, Path>,
  history: Vec<NameEvent>,
}
```
Another important addition to namespaces is a 'salt'.
This is used to make sure that local names defined in different scopes are globally unique.
Every time a local name is defined in Halcyon, the salt is incremented and appended to the end of the name.
This is also called "mangling", and depending on the languages scoping rules can actually become quite [complicated and nasty](https://github.com/rust-lang/rfcs/blob/master/text/2603-rust-symbol-name-mangling-v0.md).

Depending on your languages needs, you may need to attach all sorts of extra information to your namespace.
Halcyon additionally keeps track of each declarations function depth, which is used to detect illegally recursive definitions.
It may also be a good idea to track the spans of where definitions where declared inside a namespace.

## Types
Types are complicated enough to warrant their own section.
Crucially, the intermediate representation is the first time types appear.
Every `IrNode` and `Pattern` has an attached type, initially set to a "null" value.
The `TypeExpression` and `TypeDefinition` seen before are converted into `Type` and `AbstractType` respectively.
Note that `Typed` is to `Type` as `Spanned` is to `Span`, simply a wrapper that associates a `Type` with something.
```rust
enum Type {
    /// The "null" or unevaluated type
    Any,
    /// The empty type ()
    Unit,
    /// Signed 64 bit integer
    Integer,
    /// IEEE 64 bit floating point
    Real,
    /// true or false
    Boolean,
    /// Fat pointer to byte array of UTF-8
    String,
    /// UTF-8 codepoint 32 bit
    Glyph,
    // Type variable
    Variable(TypeVariable),
    /// Record type
    Struct {
        name: Path,
        // IndexMap is an ordered HashMap
        // with unordered equality
        fields: IndexMap<String, Type>,
    },
    /// Array type
    Array(Box<Type>),
    /// Tuple
    Product(Vec<Type>),
    /// Variant
    Sum {
        name: Path,
        variant_names: Vec<String>,
        variant_types: Vec<Type>,
    },
    /// Function type
    Function(Box<Type>, Box<Type>),
    Instantiation(Path, Vec<Type>),
}

struct AbstractType {
  // The number of type parameters
  arity: usize,
  // Base type, with parameters replaced with
  // type variables (starting with Type::Variable(0))
  base: Type,
}
```
### Type variables
The hindley milner type inference scheme Halcyon's type system is based on uses type variables.
These are like placeholders for some yet undetermined type.
When a variable appears in a type signature, that type is "abstract".
An abstract type can be "instantiated" by providing some parameter types which will replace each type variable.

### Abstract types
Type definitions may have zero or more type parameters, and must be instantiated in order to be used.
An abstract type represents a possibly uninstantiated type created by a type definition.
All in-scope abstract types are stored in a "universe", which is just a dictionary mapping paths to a corresponding `AbstractType`.

The `Instantiation` variant of the `Type` enum represents to the instantiation of an abstract type.
But why not just replace this variant with the result of its instantiation?
The answer is self referential types.
If I were to try and fully expand a type like `LinkedList` that refers to itself, the code would recurse infinitely.
The `Instantiation` variant is an annoying edge case in the compiler that has caused me a lot of grief.
However, it is the best solution to the problem of cyclical types I have found that works inside the constraints of Rust's memory model.

## Code generation
After type checking, it is time to lower the IR tree into an array of assembly instructions.
The `Instruction` enum is part of the `wasm-encoder` crate, and contains every kind of WASM instruction.
Compiled functions are represented as such:
```rust
struct EncodedFunction {
    // Index into the WASM function section
    type_id: u32,
    // List of stack variables
    locals: Vec<ValType>,
    instructions: Vec<Instruction>,
}
```
Many types considered distinct during type checking have the same assembly representation.
It is useful to simplify types before the code generation stage.
Type instantiations must also be expanded at this stage.
The way I handle this in Halcyon is to represent sum types as type erased pointers (`anyref` in wasm).
This eliminates the possibility of infinite recursion.
```rust
enum ReducedType {
    AnyRef,
    Sum,
    I64,
    F64,
    I32,
    I8,
    Function,
    Struct(Vec<ReducedType>),
    Array(Box<ReducedType>),
}
```
