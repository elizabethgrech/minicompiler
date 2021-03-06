Minicompiler
============

This is a fully functional by-the-book compiler for a tiny Pascal-like language written in pure Java, for educational purposes.
If you can read Java and would like to see how a basic compiler works, this is certainly not the worst place to go.

The language it compiles looks like this:

    {
      n : int := readInt();
      printInt(n);
      while n > 1 do {
          if n % 2 == 0
            then n := n / 2;
            else n := 3*n + 1;
          printInt(n);
      }
    }


There are ints, booleans, ifs and whiles and, to keep things short and simple, little else.
Functions, arrays etc. are left as an exercise for the reader ;)

The compiler outputs x86 assembly language that can be assembled with the [GNU Assembler](http://en.wikipedia.org/wiki/GNU_Assembler).
See `examples/compile.sh` for how to assemble and link the output.
The standard functions `readInt` and `printInt` are built to use Linux syscalls, so you currently need to run this on Linux.


Compilation stages
------------------

The [**compiler**](https://github.com/mpartel/minicompiler/blob/master/src/main/java/minicompiler/Compiler.java)
is split into stages like any compiler textbook would tell you to do.

First, the [**tokenizer**](https://github.com/mpartel/minicompiler/blob/master/src/main/java/minicompiler/Tokenizer.java)
reads source code and outputs a list of *tokens*.

    ArrayList<Token> tokens = Tokenizer.tokenize(sourceCode);

A token is a piece of text that the language should consider one syntactical "thing".
For instance, a variable name like `foo` is a single token, a parenthesis `(` is a token
and the operator `<=` is also a token.

Now the [**parser**](https://github.com/mpartel/minicompiler/blob/master/src/main/java/minicompiler/Parser.java)
reads the list of tokens and produces an abstract syntax tree (AST).

    Statement stmt = Parser.parseStatement(tokens);

For instance, the statement `if x + y < 3 then printInt(3);` is parsed into the following AST:

<img src="https://github.com/mpartel/minicompiler/raw/master/doc/ast-example.png" />

Next, the AST is traversed by the [**type checker**](https://github.com/mpartel/minicompiler/blob/master/src/main/java/minicompiler/TypeChecker.java).

    TypeChecker.checkTypes(stmt, StdlibTypes.getTypes());

The type checker makes sure that variables are declared before use,
that variable scopes are respected, that expressions in if-statement conditions are boolean etc.

After type checking, the AST is transformed into a high-level pseudo-assembly
[**intermediate representation**](https://github.com/mpartel/minicompiler/blob/master/src/main/java/minicompiler/IrGenerator.java) (IR).

    List<IrCommand> irCommands = IrGenerator.generate(stmt);

The IR consists of simple operations like `a := b + c` as well as jumps. The expression `if x + y < 3 then printInt(3);`
would be compiled into the following kind of intermediate representation.

        ...
        tmp := x + y
        tmp2 := tmp < 3
        goto skip if not tmp2
        unused := printInt(3)
    skip:
        ...

This is reasonably close to the final assembly language we want to produce.
The [**code generator**](https://github.com/mpartel/minicompiler/blob/master/src/main/java/minicompiler/backend/ia32/IA32CodeGen.java)
produces assembly from the IR.

    List<String> asmLines = IA32CodeGen.generateAsmProgram(irCommands);

The code generator must worry about things like where each local variable goes on the stack
and how machine registers are used.

The final output can be compiled into an object file with the GNU Assembler.
The object file should then be linked with the standard library object files
`stdlib/stdlib.o` and `stdlib/syscalls.o` to produce the final binary.


Building and running the compiler
---------------------------------

If you have [NetBeans](http://netbeans.org/) installed, simply open it as a project and hit Build.
It may work as easily with other IDEs too.

To build the compiler on the command line you need [Maven](http://maven.apache.org/). Simply run `mvn verify` in the project directory.

To run the compiler and get assembly language output, do `java -jar target/minicompiler-dev.jar inputfile`.

To compile and link the example files, use `examples/compile.sh`.

Advice
------

Here are a few recommendations if you'd like to build your own compiler.

- Consider using [ANTLR](http://www.antlr.org/) or some other parser generator or parser combinator library.
- Use [LLVM](http://llvm.org/) as your backend or IR. It gives you lots of optimizations for free and is quite portable. Alternatively you could [generate Java bytecode](http://asm.ow2.org/).
- Functional languages with pattern matching like Haskell, ML and Scala are excellent for writing compilers, since compilers involve lots of recursive algorithms and datastructures. Many functional languages come with a parser combinator library. Pure Java is serviceable but a little tedious, as can be seen from the verbose AST classes and the need for the Visitor pattern in this project.

Recommended reading
-------------------

- [Types and Programming Languages](http://www.cis.upenn.edu/~bcpierce/tapl/) is an excellent introduction to type systems.
- [Compilers: Principles, Techniques, and Tools](http://dragonbook.stanford.edu/) (aka. the dragon book) has both an approachable introduction and lots of in-depth information on all of the traditional compiler stages.
