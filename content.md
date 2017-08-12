# Introduction to C++ modules in clang

## 1. Textual inclusion

Parsing code in C++ can be roughly divided into two different parts, the preprocessor step and the actual parsing which acts on the output of the preprocessor. The preprocessor acts upon multiple files, usually one source file and multiple header files that are included. In the following example the preprocessor copies the contents of `foo.h` into the source file `main.cpp` which is then forwarded to the actual parsing logic. The copying of the contents into the soruce file is what we from now on call an **textual include**:

    // foo.h
    int foo();

    // main.cpp
    #include "foo.h"
    
    int main() {
      foo();
    }

The result of the clang preprocessor (obtained via `clang -E main.cpp` and stripped of some internal clang source location markers like `# 1 ...`) looks like this:

    int foo();
    int main() {
      foo();
    }

This is now getting parsed by the parser that transforms this code into an AST, containing two FunctionDecls where one also contains a code body with the `foo();` call.

## 2. Limitations of the textual inclusion model

Above's example works pretty well and is as optimized as it gets, but that's only because we have only one source files. The situation changes when we have multiple source files or headers that include our `foo.h`. Let's assume we have a second source file called `foo.cpp` implementing the `foo` function and also including the `foo.h`:

    // foo.cpp
    #include "foo.h"
    int foo() {
       // lots of code here
    }

When compiling our project now, we would have two spawn two clang processes that both end up have to parse the contents of `foo.h`  and both end up with the same FunctionDecl of `foo` with the same attributes in their AST (1). This means that when compiling our project, the two clang processes perform redundant work when parsing this header.

But so far the compilation time isn't so bad. The header in our example only contains a simple forward declaration that only takes a few CPU cycles to parse. Even when we include this header hundred of times, we would barely notice that we repeatedly parse this tiny forward declaration - the bigger function body is after all in the source file which we only parse once.

Things change when we now write more modern C++ that contains classes and templates. Both require that we add a lot of code to the header itself: For classes we have to list the class, it's parents, all function decls and all members in the header. C++ also encourages here to write the method bodies in the header by marking all method inline. For templates we even *have to* add the full function body to the header.

Now we suddenly do notice that we reparse header contents, as the workload for parsing these hader dramatically increases and we spend a lot of time just reparsing these header contents when building our project. Depending on the project, this easily means that we spend half of our time just parsing the same code over and over again. When using trendy header only libraries, the situation gets worse and worse and the redundant parsing becomes a real performance problem.

The solution for this sounds pretty forward: If we do redundant work, we just have to add a cache inbetween where we store the result of the first parsing process. But things aren't as simple as they sound.

*(1) Actually the FunctionDecls are slightly different, as the FunctionDecl in `foo.cpp` has a redeclaration chain containing the definition and the one in `main.cpp` doesn't, but for a brief moment during parsing they contain FunctionDecls with the same attributes and an empty redeclaration chain.*

## 3. Precompiled headers

Before jumping into modules, let's look at **p**re**c**ompiled **h**eaders (**PCH**s) which work in a similar way (and are even using the same underlying code in clang).

Even the simplest real-world C++ program includes some the standard C++ headers. These are provided by the implementation and provide basic functionality like writing to stdout/stderr and so on. Yet, they are actually quite large - including the libstdc++ `iostream` header for example provides around 40 000 AST nodes (2)!

As some of these headers like `vector` or `string` are nearly always textually included in an translation unit (either indirectly or directly), we could make an simple optimization and just allow changing the starting point of the parsing process to a point after the those headers are already in the AST.

In practice this would be done by parsing all those headers like `vector` into a translation unit and then serializing this translation unit (TU). One way of serializing them would be to just dump the memory of the TranslationUnitDecl/Preprocessor state and loading this dump when we create our 'empty' translation unit/preprocessor at the start of the parsing process (but now this translation unit is no longer empty, but already filled with the declarations from these headers). E.g. we from this starting TU:

    TranslationUnitDecl 0x556bc3b85c30 <<invalid sloc>> <invalid sloc>

to something like this:

	TranslationUnitDecl 0x55bf67561c30 <<invalid sloc>> <invalid sloc>
	`-NamespaceDecl 0x55bf67597c18 <<stdin>:1:1, col:51> col:11 std
	  |-CXXRecordDecl 0x55bf67597c80 <col:17, col:31> col:23 class string definition
	  | `-CXXRecordDecl 0x55bf67597da8 <col:17, col:23> col:23 implicit class string
	  `-CXXRecordDecl 0x55bf67597e60 <col:34, col:48> col:40 class vector definition
	    `-CXXRecordDecl 0x55bf67597f88 <col:34, col:40> col:40 implicit class vector

(Note: I omitted the builtin declarations that are always in the TU)

Same goes for the preprocessor which now also contains the header guards for those headers etc. Now the parsing process can continue like usual and just start appending the user's code at this translation unit. When the user includes `string`, we don't parse it (as we already define it's header guards) and when `std::string` is referenced the deserialized declaration is found. This is more or less the idea behind a PCH.

So this solves the parsing problem for the STL, but does it work also for all generic headers in our program?

Imagine we have 1000 headers in our program and we would all serialize them like the STL above. Now we would have a few hundred megabytes on the disk (as the declarations in a 1000 headers easily accumulate this amount of memory) and we would have to load this enitire giant memory dump every time before we start parsing. And given that the average source file in a project probably doesn't even need all declarations from all 1000 headers, we even do unnecessary work when we load this giant PCH!

So we're stuck again but this time we have some ways to optimize. 

# 4. On-demand deserialization

The first optimizaion that comes to mind is that we don't just load the entire file, but rather construct it like a database containing serialized declarations where the keys are lookups (e.g. the key `std::string` would point to the serialized decl inside this datbase). From this database we now just lazily load the declarations our current source file needs.

This mechanic however require some larger changes in our compiler infrastructure beyond just changing the initial translation unit. We now have an interface to our data base (which we still call a PCH) which clang calls an `ExternalASTSource` and we have multiple calls to this source from various parts of the compiler where loading a declaration could occur. For example the call `requireCompleteType` now also checks if the current ExternalASTSource of the compiler can complete the current type we need a definition of (for example by deserializing the specific type from the PCH and adding it to our AST).



*(2) You can check this by running* `echo "#include <iostream>" | clang++ -Xclang -ast-dump -fsyntax-only -xc++ - | wc -l`

## 2. The module compilation process

## 3. Entity loading

## 4. PCM cache

## 5. Standalone headers

## 6. Common bugs

