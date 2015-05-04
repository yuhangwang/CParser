# CParser

(c) 2015 Léon Bottou -- Facebook AI Research

This pure Lua module implements (1) a standard compliant C
preprocessor with a couple useful extensions, and (2) a parser that
provides a Lua friendly description of all declarations and
definitions in a C header or C program file.

The driver program `lcpp` invokes the preprocessor and outputs
preprocessed code. Although it can be used as a replacement for the
normal preprocessor, it is more useful as an extra preprocessing step
(see option `-Zpass` which is on by default.)  The same capabilities
are offered by functions `cparser.cpp` and `cparser.cppTokenIterator`
provided by the module `cparser`.

The driver program `lcdecl` analyzes a C header file and a C program
file and outputs a short descriptions of the declarations and
definitions. This program is mostly useful to understand the
representations produced by the `cparser` function
`cparser.declarationIterator`.

---

## Program `lcpp`

### Synopsis

```sh
    lcpp [options] inputfile.c [-o outputfile.c]
```
Preprocess file `inputfile.c` and write the preprocessed code into
file `outputfile.c` or to the standard output.

### Options

The following options are recognized:

- `-Werror`  
  Cause all warning to be treated as errors.
  Note that parsing cannot resume after an error.
  The parser simply throws a Lua error.

- `-w`   
  Do not print warning messages.

- `-D`*sym*`[=`*val*`]`  
   Define preprocessor symbol `sym` to value `val`.
   The default value of `val` is `1`

- `-U`*sym*   
  Undefine preprocessor symbol `sym`.

- `-I`*dir*   
  Add directory `dir` to the search path for included files. Note
  that there is no default search path. When an include file is not
  found the include directive is simply ignored with a warning (but
  see also option `-Zpass`).  Therefore all include directives are
  ignored unless one uses option `-I` to specify the search path.

- `-I-`   
  Marks the beginning of the system include path. When an included
  file is given with angle brackets, (as in `#include <stdio.h>`),
  one only searches directories specified by the `-I` options that
  follow `-I-`. Therefore all these include directives are ignored
  unless one uses option `-I-` followed by one or more option `-I`.

- `-dM`   
  Instead of producing the preprocessed file,
  dumps all macros defined at the end of the parse.

- `-Zcppdef`   
  Run the native preprocessor using command `cpp -dM < dev/null`
  and copy its predefined symbols. This is useful when using
  `lcpp` as a full replacement for the standard preprocessor.

- `-Zpass`   
  This option is enabled by default (use `-Znopass` to disable)
  and indicates that the output of `lcpp` is going to be
  reprocessed by a C preprocessor and compiler.
  This option triggers the following behavior:

  * The preprocessor directives `#pragma` and `#ident`
    are copied verbatim into the output file.
  * When the included file cannot be found in the provided
    search path, the preprocessor directive `#include` is
    copied into the output file.
  * Preprocessor directives prefixed with a double `##` are copied
    verbatim into the output file with a single `#` prefix.  This
    feature is useful for `#if` directives that depend on symbols
    defined by unresolved `#include` directives.

- `-std=(c|gnu)(89|99|11)`  
  This option selects a C dialect.
  In the context of the preprocessor, this only impacts
  the symbols predefined by `lcpp`.
  * Symbol `__CPARSER__` is always defined with value <1>.
  * Symbols `__STDC__` and `__STDC_VERSION__` are either defined by
    option `-Zcppdef` or take values suitable for the target C
    dialect.
  * Symbols `__GNUC__` and `__GNUC_MINOR__` are either defined bu
    option `-Zcppdef` or are defined to values `4` and `2` if the
    target dialect starts with string `gnu`.
    
  This can be further adjusted using the `-D` or `-U` options.
  The default dialect is `gnu99`.
  

### Preprocessor extensions

The `lcpp` preprocessor implements several useful nonstandard features.
The main feature are multiline macros. The other features are mostly
here because they make multiline macros more useful.

####  String comparison in conditional expressions

The C standard specifies that the expressions following `#if`
directives are constant expressions of integral type. However this
processor also handles strings. The only valid operations on strings
are the equality and ordering comparisons. This is quite useful to
make special cases for certain values of the parameters of a multiline
macro, as shown later.

####  Multiline macros

Preprocessor directives `#defmacro` and `#endmacro` can be used to
define a function-like macro whose body spans several lines. The
`#defmacro` directive contains the macro name and a mandatory argument
list. The body of the macro is composed of all the following lines up
to the matching `#endmacro`. This offers several benefits:

* The line numbers of the macro-expansion is preserved. This ensures
  that the compiler produces error messages with meaningful line
  numbers.

* The multi-line macro can contain preprocessor directives.
  Conditional directives are very useful in this context.  Note
  however that preprocessor definitions (with `#define`, `#defmacro`,
  or `#undef`) nested inside multiline macros are only valid within
  the macro.

* The standard stringification `#` and token concatenation `##`
  operators can be used freely in the body of multiline macros.  Note
  that these operators only work with the parameters of the multiline
  macros and not with ordinary preprocessor definitions. This is
  consistent with the standard behavior of these operators in ordinary
  preprocessor macros.

  Example

```C
      #defmacro DEFINE_VDOT(TNAME, TYPE)
        TYPE TNAME##Vector_dot(TYPE *a, TYPE *b, int n)
        {
          /* try cblas */
        #if #TYPE == "float"
          return cblas_sdot(n, a, 1, b, 1);
        #elif #TYPE == "double"
          return cblas_ddot(n, a, 1, b, 1);
        #else
          int i;
          TYPE s = 0;
          for(i=0;i<n;i++)
            s += a[i] * b[i];
          return s;
        #endif
        }
      #endmacro

      DEFINE_VDOT(Float,float);
      DEFINE_VDOT(Double,double);
      DEFINE_VDOT(Int,int);
```

Details -- The values of the macro parameters are normally
macro-expanded before substituting them into the text of the
macro. However this macro-expansion does not happen when the
substitution occurs in the context of a stringification or token
concatenation operator.  All this is consistent with the standard. The
novelty is that this macro-expansion does not occur either when the
parameter appears in a nested preprocessor directive or multiline
macro.
	
More details -- The stringification operator only works when the next
non-space token is a macro parameter.  This provides a good way to
distinguish a nested directive from a stringification operator
appearing in the beginning of a line.

####  Negative comma in variadic macros

Consider the following variadic macro

```C
     #define macro(msg, ...)  printf(msg, __VA_ARGS__)
```

The C standard says that it is an error to call this macro with only
one argument. Calling this macro with an empty second argument
--`macro(msg,)`-- leaves an annoying comma in the expansion
--`printf(msg,)`-- and causes a compiler syntax error.

This preprocessor accepts invocations of such a macro with a single
argument. The value of parameter `__VA_ARGS__` is then a so-called
negative comma, meaning that the preceding comma is eliminated when
this parameter appears in the macro definition between a comma and a
closing parenthesis.

####  Recursive macros

When a new invocation of the macro appears in the expansion of a
macro, the standard specifies that the preprocessor must rescan the
expansion but should not recursively expand the macro.  Although this
restriction is both wise and useful, there are rare cases where one
would like to use recursive macros.  As an experiment, this recursion
prevention feature is turned off when one defines a multiline macro
with `#defrecursivemacro` instead of `#defmacro`. Note that this might
prevent the preprocessor from terminating unless the macro eventually
takes a conditional branch that does not recursively invoke the macro.

---

## Program `lcdecl`


### Synopsis

```sh
    ldecl [options] inputfile.c [-o outputfile.txt]
```

Preprocess and parse file `inputfile.c`.
The output of a parser is a sequence of Lua data structures
representing each C definition or declaration encountered in the code.
Program `ldecl` prints each of them in two forms. The first form
directly represent the Lua tables composing the data structure. The
second form reconstructs a piece of C code representing the definition
or declaration of interest.

This program is mostly useful to people working with the Lua functions
offered by the `cparser` module because it provides a quick way to inspect
the resulting data structures.



### Options

Program `lcdecl` accepts all the preprocessing options
documented for program `lcpp`. It also accepts an additional
option `-T`$typename$ and also adds to the meaning of
options `-Zpass` and `-std=`$dialect$.

- `-T`$typename$   
   Similar to `lcpp`, program `lcdecl` only reads the include files
   that can be found along the path specified by the `-I` options. It
   is generally not desirable to read all include files because they
   usually contain declarations that are not directly useful. This
   also means that the C parser is not aware of type definitions found
   in these ignored include files. Fortunately the C syntax is
   sufficiently unambiguous to allow the parser to guess that an
   identifier is a type name rather than a variable name.  However
   this can lead to confusing error messages.

   Option `-T`$typename$ can then be used to inform the parser than
   symbol `typename` represents a type and not a constant, a variable,
   or a function.

- `-Zpass`
   Unlike `lcpp`, program `lcdecl` processes the input file
   with option `-Zpass` off by default. Turning it on will
   simply confuse the parser with unexpected preprocessor
   constructs.

- `-std=(c|gnu)(89|99|11)`  
   The dialect selection options also control whether the parser
   recognizes keywords introduced by later version of the C standard
   (e.g., `restrict`, `_Bool`, `_Complex`, `_Atomic`, `_Pragma`,
   `inline`) or by the GCC compiler (e.g., `asm`). Many of these
   keywords have a double-underline-delimited variant that is
   recognized in all cases (e.g, `__restrict__`).

Example.

Running `ldecl` on the following program

```C
const int size = (3+2)*2;
float arr[size];
typedef struct symtable_s { const char *name; SymVal value; } symtable_t;
void printSymbols(symtable_t *p, int n) { do_something(p,n); }
```

produces the following output (with very long lines).

```
+--------------------------
| Definition{where="test.c:2",intval=10,type=Qualified{t=Type{n="int"},const=true},name="size",init={..}}
| const int size = 10
+--------------------------
| Definition{where="test.c:3",type=Array{t=Type{n="float"},size=10},name="arr"}
| float arr[10]
+--------------------------
| TypeDef{sclass="[typetag]",where="test.c:4",type=Struct{Pair{Pointer{t=Qualified{t=Type{n="char"},const=true}},"name"},Pair{Type{n="SymVal"},"value"},n="symtable_s"},name="struct symtable_s"}
| [typetag] struct symtable_s{const char*name;SymVal value}
+--------------------------
| TypeDef{sclass="typedef",where="test.c:4",type=Type{_def={..},n="struct symtable_s"},name="symtable_t"}
| typedef struct symtable_s symtable_t
+--------------------------
| Definition{where="test.c:5",type=Function{Pair{Pointer{t=Type{_def={..},n="symtable_t"}},"p"},Pair{Type{n="int"},"n"},t=Type{n="void"}},name="printSymbols",init={..}}
| void printSymbols(symtable_t*p,int n){..}
+--------------------------
```



---

## Module `cparser`

Module `cparser` exports the following functions:

### Preprocessing function


##### `cparser.cpp(filename, outputfile, options)`

Program `lcpp` is implemented by function `cparser.cpp`.

Calling this function preprocesses file `filename` and writes the
preprocessed code to the specified output.  The optional argument
`outputfile` can be a file name or a Lua file descriptor.  When this
argument is `nil`, the preprocessed code is written to the standard
output.  The optional argument `options` is an array of option
strings.  All the options documented with program `lcpp` are
supported.

##### `cparser.cppTokenIterator(options, lines, prefix)`

Calling this function produces two results:
* A token iterator function.
* A macro definition table.

Argument `options` is an array of options.
All the options documented for program `lcpp` are supported.
Argument `lines` is an iterator that returns input lines.
Lua provides many such iterators, including `io.lines(filename)` to
return the lines of the file named `filename` and `filedesc:lines()`
to return lines from the file descriptor `filedesc`. You can also use
`string.gmatch(somestring,'[^\n]+')` to return lines from string
`somestring`.

Successive calls of the token iterator function describe the tokens of
the preprocessed code. Each call returns two strings.  The first
string represent the token text. The second string follows format
`"filename:lineno"` and indicates on which line the token was
found. The filename either is the argument `prefix` or is the actual
name of an included file. When all the tokens have been produced, the
token iterator function returns `nil`.

Each named entry of the macro definition table contains the definition
of the corresponding preprocessor macros. Function
`cparser.macroToString` can be used to reconstruct the macro
definition from this information.

Example:
```Lua
      ti,macros = cparser.cppTokenIterator(nil, io.lines('test/testmacro.c'), 'testmacro.c')
      for token,location in ti do
        print(token, location)
      end
      for symbol,_ in pairs(macros) do
        local s = cparser.macroToString(macros,symbol)
	if s then print(s) end
      end
```

##### `cparser.macroToString(macros,name)`

This function returns a string representing the definition
of the preprocessor macro `name` found in macro definition table `macros`.
It returns `nil` if no such macro is defined.
Note that the macro definition table contains named entries that
are not macro definitions but functions implementing
magic symbols such as `__FILE__` or `__LINE__`.



### Parsing functions

##### `cparser.parse(filename, outputfile, options)`

Program `lcdecl` is implemented by function `cparser.parse`.

Calling this function preprocesses and parses file `filename` and
writes a trace into the specified file. The optional argument
`outputfile` can be a file name or a Lua file descriptor.  When this
argument is `nil`, the preprocessed code is written to the standard
output.  The optional argument `options` is an array of option
strings.  All the options documented with program `lcdecl` are
supported.


##### `cparser.declarationIterator(options, lines, prefix)`

Calling this function produces three results:
* A declaration iterator function.
* A symbol table.
* A macro definition table.

Argument `options` is an array of options.
All the options documented for program `lcdecl` are supported.
Argument `lines` is an iterator that returns input lines.
Lua provides many such iterators, including `io.lines(filename)` to
return the lines of the file named `filename` and `filedesc:lines()`
to return lines from the file descriptor `filedesc`. You can also use
`string.gmatch(somestring,'[^\n]+')` to return lines from string
`somestring`.

Successive calls of the declaration iterator function
returns a Lua data structure that represents a C name
declaration or definition. The format of these
data structures is described under function
`cparser.declToString`.

The symbol table contains the definition of all
the C language identifiers defined or declared
by the parsed files. Type names are represented
by the `Type{}` data structure documented under
function `cparser.typeToString`. Constants,
variables, and functions are represented
by `Definition{}` or `Declaration{}`
data structures similar to those returned
by the declaration iterator.

The macro definition table contains
the definition of the preprocessor macro.
See the preprocessor documentation for details.

Example

```Lua
      di = cparser.declarationIterator(nil, io.lines('tests/testmacro.c'), 'testmacro.c')
      for decl in di do print(decl) print(">>", cparser.declToString(decl)) end
```


##### `cparser.typeToString(ty,nam)`




##### `cparser.declToString(decl)`



