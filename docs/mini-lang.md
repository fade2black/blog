<div class="meta-data">03 apr 2022 </div>

# Building a Simple Functional Language Compiler in Rust

## Introduction

For fun and learning, I built a compiler for a small toy functional (Turing complete 😉) language in Rust. The language is minimal: it supports defining functions, conditionals, and basic arithmetic. To keep things simple, all values are 32-bit floating point numbers, and every expression ends with a semicolon.

The compiler takes source code written in this language and converts it into WebAssembly, producing both WebAssembly Text (`.wat`) and binary (`.wasm`) files. Everything is hand-written—from the lexer and parser to the abstract syntax tree and code generator.

Language Overview

- **Functions**: define reusable blocks of code
- **Conditionals**: `if-then-else` expressions
- **Arithmetic** : addition, subtraction, multiplication, division
- **Comments**: lines starting with symbol "#"
- **Types**: all 32-bit floats

Example 1: computing the sum of the first n integers.
```
# Sum of first `n` integers
def sum(x) 
 if x == 1 
   then 1
   else sum(x-1) + x;
```

Example 2: Solving a quadratic equation
```
# Solution of a second
# order equation with coeffients a,b,c

def root1(a b c)
  if discr(a, b, c) < 0
  then 0 
  else (-b + sqrt(discr(a, b, c)))/(2*a);

def root2(a b c)
  if discr(a, b, c) < 0
  then 0 
  else (-b - sqrt(discr(a, b, c)))/(2*a);
```

## Formal Definition

- **Keywords**: `def`, `if`, `then`, `else`
- **Identifiers**: `[a-zA-Z][a-zA-Z0-9]*`
- **Numbers**: decimal floating-point values `[0-9]?(.?[0-9])`
- **Expressions**: arithmetic operations, comparisons, function calls, and conditional expressions

The Context Free Grammar (CFG) for the language is defined as follows:

```
Program ::= def Prototype Expression ; | def Prototype Expression ; Program

Prototype ::= Identifier(Params) | Identifier()

Params ::= Identifier Params | Identifier

Expression ::= Exp | IfExp

Exp ::= SubExp | Exp < SubExp | Exp > SubExp | Exp <> SubExp | Exp == SubExp

SubExp ::= Term | SubExp + Term | SubExp - Term | SubExp | Term

Term ::= Factor | Term * Factor | Term / Factor | Term & Factor

Factor ::= -Exp | ( Exp ) | Identifier | Number | FuncionCall

FuncionCall ::= Identifier(Args) | Identifier()

Args ::= Exp | Comma Args

IfExp ::= if Exp then Exp else Exp
```
The compiler follows a straightforward pipeline: 
the lexer tokenizes the input, the parser builds an AST, and the code generator emits WebAssembly code.

## Implementation

### Lexer (a.k.a. Scanner)

The lexer is the first stage of the compiler, responsible for converting raw source code 
into a sequence of tokens that the parser can understand. 
In this project, the lexer is implemented as a generic `Lexer<T>`
struct that reads from any input implementing 
`std::io::Read`, allowing it to handle files, strings, or other streams.

Internally, the lexer keeps track of the current character, 
the current lexeme (the string being built), 
and the line number for error reporting. 
It reads characters using a UTF-8 reader to handle all valid characters safely.

The lexer handles several types of tokens:

- **Whitespace**: skipped automatically, with line numbers incremented whenever a newline is encountered.
- **Comments**: lines starting with # are skipped until the end of the line.
- **Identifiers**: sequences of letters and digits, starting with a letter. These typically correspond to function names or variable names in the language.
- **Numbers**: sequences of digits, optionally containing a decimal point. Only 32-bit floating-point numbers are supported.
- **Operators and punctuation**: such as `+`, `-`, `*`,... (see `src/lexer.rs` for more)

The lexer uses helper functions to organize the logic:

- `get_char()` reads the next character from the input.
- `skip_whitespace()` ignores spaces, tabs, and newlines.
- `skip_comment()` ignores all characters until the end of a line.
- `get_identifier()` and `get_number()` build identifiers and numeric literals, respectively, adding characters to the lexeme string.
- `other()` handles operators and punctuation, including multi-character tokens like `<>` (not equal) and `==` (equal).

By the end of this stage, the source code has been transformed into a stream 
of well-defined tokens that the parser can use to construct the abstract syntax tree. 
The lexer is entirely hand-written, giving full control over how tokens are recognized and providing a clear foundation for learning compiler design in Rust. 
You can find the source code [here](https://github.com/fade2black/mini-language/blob/main/src/lexer.rs).


### Parser

The parser transforms the stream of tokens from the lexer into an Abstract Syntax Tree (AST),
representing the structure of the program.
It is implemented as a hand-written recursive-descent parser (for LL(1) grammars),
following a standard operator-precedence hierarchy:

`factors` > `terms` > `subexpressions` > `expressions`

This ensures that multiplication and division bind tighter than addition/subtraction, and comparison operators are evaluated at the top level.

The parser handles:

- **Function definitions**, including name, parameters, and body expressions
- **Expressions**, including arithmetic, variable references, function calls, and conditional `if-then-else` expressions
- **Error logging**, using an `ErrorLogger` that records parsing errors with line numbers
- **Error recovery**, via a synchronization mechanism that skips tokens until a logical resumption point (`;` or `end-of-file`)

A key component is the `main_loop()`, which drives parsing by repeatedly reading function definitions until the end of the source. 
The parser also distinguishes between variable references and function calls in `parse_identifier_expr()`, correctly collecting arguments when present.

Overall, the parser converts the source code into a structured, navigable AST,
providing a solid foundation for generating WebAssembly code.
Most other parsing functions are straightforward and follow standard recursive-descent patterns,
so only the main loop, expression hierarchy, and argument handling require special attention. You can find the source code [here](https://github.com/fade2black/mini-language/blob/main/src/parser.rs).

### AST and Code Generation

The parser produces an Abstract Syntax Tree (AST) that represents
the structure of the program independently of the source syntax. 
In this compiler, the AST is intentionally minimal and expression-oriented,
closely mirroring the language design where everything evaluates to a value.

### AST Design
Expressions are represented by the ExprNode enum.
Each variant corresponds to a core language construct: 
numeric literals, variables, unary and binary expressions, function calls,
and conditional expressions. 
Recursive structures are modeled using `Box<ExprNode>`, 
allowing complex expressions such as nested arithmetic
or conditionals to be represented naturally.


Rather than separating analysis and code generation into multiple passes,
each AST node knows how to emit its own **WebAssembly Text** (**WAT**) 
representation via the `to_wat()` method. 
This keeps the implementation compact and makes the mapping between 
language constructs and WebAssembly instructions explicit.

Some notable design choices:

- Expression-oriented code generation: each expression emits instructions
that leave its result on the WebAssembly stack.
- Operator-driven emission: binary and unary expressions delegate instruction
selection to the Operator type, keeping operator semantics centralized.
- Conditionals map directly to Webassembly’s structured control 
flow using `if (result f32)`, `else`, and `end`.

Function calls are handled uniformly, with a small exception for
built-in functions such as `sqrt` or `abs`.
These are mapped directly to WebAssembly instructions instead of emitted as
function calls, avoiding unnecessary indirection.

### Functions and Prototypes

Functions are represented by a `Prototype` (name and parameters) and a body expression.
The prototype is responsible for emitting the function signature,
while the body emits the instructions computing the return value.
All functions return a single `f32`, matching the language’s simplified type system.

This design closely follows the idea that a function is simply a named expression with parameters.

### Code Generator

The `CodeGenerator` is responsible for assembling the final WebAssembly module. 
It iterates over the AST, emits function definitions, and exports all defined functions
so they can be called from JavaScript or other WebAssembly hosts.

The generation process is straightforward:

1. Open a WebAssembly module.
2. Emit each function definition by delegating to the AST.
3. Export all function symbols.
4. Close the module.

By keeping code generation mostly inside the AST nodes, 
the CodeGenerator remains small and declarative, 
acting as a coordinator rather than a complex transformation stage.

### Wrapping Up

The modules described so far—lexer, parser, AST, 
and code generation—cover the core of the compiler pipeline. 
There are a few additional supporting modules in the repository 
(such as error handling and operator definitions),
but they follow naturally from the design already discussed
and do not introduce new concepts.

Rather than walking through every remaining file, 
I prefer to leave those details to the reader. 
The full source code is available on [GitHub](https://github.com/fade2black/mini-language/blob/main/README.md) and is intentionally kept small and readable, 
making it easy to explore individual modules independently.

At this point, the interesting question becomes how all of these pieces
are wired together and how the compiler is actually executed. 
That logic lives in `main.rs`.

## Putting Everything Together (`main.rs`)

```rust
use minilang::code_generator::CodeGenerator;
use minilang::parser::Parser;
use std::process::Command;

use std::env;
use std::fs::File;

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    if args.len() < 3 {
        println!("Not enough arguments. Please specify input and output files names.");
        return Ok(());
    }

    let src = File::open(args[1].as_str())?;
    let target = File::create(args[2].as_str())?;

    let mut parser = Parser::new(src);
    parser.main_loop();

    let err_logger = parser.get_error_logger();

    if err_logger.has_errors() {
        for error in err_logger.iter() {
            println!("SYNTAX ERROR: {}", error);
        }
    } else {
        // Generate WebAssembly text
        CodeGenerator::new(parser.get_asts(), target).run()?;
        // Generate binary WebAssembly
        Command::new("sh")
            .arg("-c")
            .arg(format!("wat2wasm {}", args[2].as_str()))
            .output()
            .expect("failed to execute 'wat2wasm'");
    }

    Ok(())
}
```

If `wat2wasm` is not available, you can install it using `brew install wabt`.

Create a file named `fib.txt` with the following content:

```
# Computes Fibbonaci sequence
def fib(x)
  if (x == 1) | (x == 2)
    then 1
    else fib(x-1) + fib(x-2);
```

and the run
```bash
cargo run fib.txt target.wat
```
It'll generate two files, `target.wat` and `target.wasm`.
Next create a javascript file `fib.js`

```JavaScript
const { readFileSync } = require("fs");

const run = async () => {
  const buffer = readFileSync("./target.wasm");
  const module = await WebAssembly.compile(buffer);
  const instance = await WebAssembly.instantiate(module);
  console.log(instance.exports.fib(5)); // Call fibbonaci function
};

run();
```

and run it

```bash
node fib.js
```

The following example computes roots of a quadratic equation (assuming they are different from 0).

```
# Computes factorial sequence
# Solution of a second
# order equation with coeffients a,b,c

def discr(a b c)
  b*b - 4*a*c;

def root1(a b c)
  if discr(a, b, c) < 0
  then 0
  else (-b + sqrt(discr(a, b, c)))/(2*a);

def root2(a b c)
  if discr(a, b, c) < 0
  then 0
  else (-b - sqrt(discr(a, b, c)))/(2*a);
```

## Conclusion

This project was an exercise in building a compiler from first principles, deliberately avoiding frameworks 
or generator tools in favor of handwritten components.
By keeping the language small and the type system uniform, the focus stays on the essential pipeline: lexing, parsing, AST construction, and code generation. 
Targeting WebAssembly made the output both concrete and inspectable, allowing each design decision to be traced directly to emitted WAT and wasm instructions.
While the compiler is intentionally minimal, it provides a complete, end-to-end implementation that can serve as a foundation for experimentation, extension, or simply a deeper understanding of how compilers work under the hood.
