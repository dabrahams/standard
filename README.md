# 1. Introduction

Val is a research language based on the principles of mutable value semantics (MVS) (Racordon et al. 2022) for safety and efficiency. It is designed to help developers write and maintain correct programs using powerful abstractions without loss of efficiency.

Val is strongly related to the [Swift programming language](https://www.swift.org) and borrows liberally from its syntax and semantics. Nonetheless, it exposes a more elaborated memory model to the user for a tighter control over memory, blending ideas found in other languages such as Rust, C++, and Go.

On a theoretical front, Val owes greatly to linear types [(Wadler 1990)](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.31.5002&rep=rep1&type=pdf), ownership types [(Clarke et al. 2013)](https://doi.org/10.1007/978-3-642-36946-9_3), and capability-based type systems [(Smith et al. 2000)](https://doi.org/10.1007/3-540-46425-5_24), while striving to hide the complexity inherent to these approaches. It does so by excluding first-class references from the user model and using continuations to mitigate the loss of expressiveness.

# 2. Lexical conventions

## 2.1. Program text

1. A Val program is written in text format. The text of a program is written using the [Unicode](https://home.unicode.org) character set and kept in units called source files. A source file is a sequence of Unicode characters represented with the UTF-8 encoding.

2. This document refers to individual Unicode characters with the notation `U+n` where `n` is an hexadecimal value representing a Unicode code point, and refers to the Unicode general categories to identify groups of Unicode characters.

## 2.2. Lexical translations
    
1. An sequence of Unicode characters is translated to a sequence of tokens, which are the terminal symbols of the syntactic grammar. The following riles apply during the translation:

    1. Comments are ignored and treated as though they were substituted by a single space `U+20`.
    
    2. Inline spaces are ignored, unless they appear between the opening and closing delimiters of a character or string literal. Inline spaces are recognized as `U+9` and/or `U+20`.

    3. New-line delimiters are ignored, unless they appear between the opening and closing delimiters of a character or string literal. New-line delimiters are recognized as `U+A`, and/or `U+D`, and/or the `U+D` directly followed by `U+A`.

## 2.3. Comments

1. The characters `//` start a single-line comment, which terminates immediately before the next new-line delimiter.

2. The characters `/*` denote a comment opening delimiter and the characters `*/` denote a comment closing delimiter. An opening comment delimiter starts a multiline comment, which terminates after a matching closing delimiter. Each opening delimiter must have a matching closing delimiter. Multiline comments may nest.

3. The characters `//` have no special meaning in a multiline comment. The characters `/*` and `*/` have no special meaning in a single-line comment. The characters `//` and `/*` have no special meaning in a string literal. Conversely, string opening delimiters have no special meaning in a comment.

## 2.4. Tokens

1. A token is a terminal symbol of the syntactic grammar. It falls into one of five categories: scalar literals, keywords, raw identifiers, raw operators, and punctuators.

2. A token is associated with a three-valued flag specifying whether it was followed by a raw character, an inline space (including an inline space substituted for a comment), or a new-line delimiter in the sequence of Unicode characters.

3. (Example)

    The input "a << b" is translated to a sequence of 4 tokens: *identifier*, *operator*, *operator*, *identifier*. The first and third tokens are known to be followed by an inline space.

4. Unless otherwise specified, tokens are recognized using the longest possible sequence of characters.

### 2.4.1. Scalar literals

#### 2.4.1.1. Boolean literals

1. The Boolean literals are the keywords `true` and `false`:

    ```ebnf
    boolean-literal ::=
      true
      false
    ```

#### 2.4.1.2. Integer literals

1. Integer literals have the form:

    ```ebnf
    integer-literal ::=
      binary-literal
      octal-literal
      decimal-litera
      hexadecimal-literal

    binary-literal ::=
      0b binary-digit
      0b _
      binary-literal binary-digit
      binary-literal _

    binary-digit ::= (one of)
      0 1

    octal-literal ::=
      0o octal-digit
      0o _
      octal-literal octal-digit
      octal-literal _

    octal-digit ::= (one of)
      0 1 2 3 4 5 6 7

    decimal-literal ::=
      decimal-digit
      decimal-literal decimal-digit
      decimal-literal _

    hexadecimal-literal
      0x hexadecimal-digit
      0x _
      hexadecimal-literal hexadecimal-digit
      hexadecimal-literal _

    hexadecimal-digit ::= (one of)
      0 1 2 3 4 5 6 7 8 9 a A b B c C d D e E f F
    ```

2. The sequence of digits of a literal are interpreted as follows, ignoring all occurences of `_`:

    1. In a binary literal, as a base 2 integer.
   
    2. In an octal literal, as a base 8 integer.

    3. In a decimal literal, as a base 10 integer.
    
    4. In an hexadecimal literal, as a base 16 integer, where the characters `a` through `f` and `A` through `F` have decimal values ten through fifteen.

3. (Example)

    The integer literal `0o12_34_5__` is interpreted as the integer `5349` in base 10.

4. The default inferred type of an integer literal is the Val standard library `Int`, which represents a 64-bit signed integer. If the interpreted value of an integer literal is not in the range of representable values for its type, the program is ill-formed.

#### 2.4.1.3. Floating-point literals

1. Floating-point literals have the form:

    ```ebnf
    floating-point-literal ::=
      decimal-floating-point-literal
      hexadecimal-floating-point-literal

    decimal-floating-point-literal ::=
      decimal-fractional-constant exponent?
      decimal-literal exponent

    decimal-fractional-constant ::=
      decimal-literal . decimal-literal

    exponent ::=
      e exponent-sign? decimal-literal
      E exponent-sign? decimal-literal

    hexadecimal-floating-point-literal ::=
      hexadecimal-fractional-constant binary-exponent
      hexadecimal-literal binary-exponent

    hexadecimal-fractional-constant ::=
      hexadecimal-literal . hexadecimal-literal

    binary-exponent ::=
      p exponent-sign? decimal-literal
      P exponent-sign? decimal-literal

    exponent-sign ::= (one of)
      + -
    ```

2. The significand of a decimal floating-point literal the *decimal-fractional-constant* or the *decimal-literal* preceeding the *exponent*. The signficand of an hexadecimal floating-point literal is the *hexadecimal-fractional-constant* or the *hexadecimal-literal*. In the significand, the digits and optional period are interpreted as a base `N` real number `s`, where `N` is 10 for a decimal floating-point literal and 16 for an hexadecimal floating-point literal, ignoring all occurences of `_`. If *exponent* or *binary-exponent* is present, the exponent `e` of the floating-point-literal is the result of interpreting the sequence of an optional `sign` and the digits as a base 10 integer. Otherwise, the exponent `e` is 0. The scaled value of the literal is `s × 10e` for a decimal floating-point literal and `s × 2e` for a hexadecimal floating-point literal.

3. The default inferred type of an integer literal is the Val standard library `Double`, which represents a 64-bit floating point number. If the interpreted value of a floating-point literal is not in the range of representable values for its type, the program is ill-formed. Otherwise, the value of a floating-point literal is the scaled value if representable, else the larger or smaller representable value nearest the scaled value, chosen in an implementation-defined manner.

#### 2.4.1.4. Character literals

1. Character literals have the form:

    ```ebnf
    character-literal ::=
      ' escape-char '
      ' c-char '

    escape-char ::=
      simple-escape
      unicode-escape
    
    simple-escape ::= (one of)
      \0 \t \n \r \' \"

    unicode-escape ::=
      \u hexadecimal-digit
    
    c-char ::= (any unicode character except ')
    ```

2. The *hexadecimal-digit*  of a *unicode-escape* represents a Unicode code point.

#### 2.4.1.5. String literals

1. String literals have the form:

    ```ebnf
    string-literal ::=
      simple-string
      multiline-string

    simple-string ::=
      " simple-quoted-text? "

    simple-quoted-text ::=
      simple-quoted-text-item
      simple-quoted-text simple-quoted-text-item

    simple-quoted-text-item ::=
      escape-char
      s-char

    s-char ::= (any character except ", U+A, and U+D)

    multiline-string ::=
      """ multiline-quoted-text """

    multiline-quoted-text ::=
      multiline-quoted-text-item
      multiline-quoted-text multiline-quoted-text-item

    multiline-quoted-text-item ::=
      escape-char
      m-char

    m-char ::= (any character except " leading a sequence of 3 or more contiguous ")
    ```

2. The first new-line delimiter in a multiline string literal is not part of the value of that literal if it immediately succeeds the opening delimiter. The last new-line delimiter that is succeeded by a contiguous sequence of inline spaces followed by the closing delimiter is called the indentation marker. The indentation marker and the succeeding inline spaces specify the indentation pattern of the literal and are not part of its value. The pattern is defined as the sequence of inline spaces between the indentation marker and the closing delimiter. That sequence must be homogeneous. If the literal has no indentation marker, its indentation pattern is an empty sequence. Each line of a multiline string literal must begin with the indentation pattern of that literal. That prefix is not part of the value of the literal.

3. (Example)

    ```val
    let s = """
        Hello,
        World!
      """
    print(s) // "  Hello,\n  World!"
    ```

### 2.4.2. Keyowrds

1. A keywords are reserved identifiers. They have the form:

    ```ebnf
    keyword ::= (one of)
      Any Never as as! _as!! async await break catch continue deinit else extension false for fun
      if import in infix init let match namespace nil postfix prefix private public return sink static true try type typealias var where while yield
    ```

### 2.4.3. Raw identifiers

1. Raw identifiers are case-sensitive sequences of letters and digits. They have the form:

    ```ebnf
    raw-identifier ::=
      raw-identifier-head
      raw-identifier raw-identifier-tail
      contextual-keyword

    raw-identifier-head ::= (_ and any character in categories Lu, Ll, Lt, Lm, Lo, Nl)

    raw-identifier-tail ::= (any character in categories Lu, Ll, Lt, Lm, Lo, Mn, Mc, Nl, Nd, Pc)

    contextual-keyword ::= (one of)
      mutating some
    ```

2. Contextual keywords are identifiers that have a special meaning when appearing in a certain context. When referred to in the grammar, these identifiers are used explicitly rather than using the identifier grammar production. Unless otherwise specified, any ambiguity as to whether a given identifier has a special meaning is resolved to interpret the token as a regular identifier.

### Raw operators

1. Raw operators have the form:

    ```ebnf
    raw-operator ::= (-, *, /, ^, %, &, ! ?, and any character in category Sm)
    ```

    [Note: The Unicode category Sm include +, =, <, >, |, and ~.]

### Punctuators

1. Punctuators have the form:

    ```ebnf
    punctuator ::= (any of)
      . , ; ( ) [ ] { } : ::
    ```

# 3. General concepts

## 3.1. Objects

1. An object is the result of a scalar literal expression, the result of an aggregate literal expression, the result of a function call, the result of a call to a `sink` accessor, or the value of an escapable binding.

2. An object has a type determined at compile-time. That type might be polymorphic; in that case, the implementation generates information associated with the object that makes it possible to determine its concrete type at runtime.

3. An object may contain other objects, called sub-objects.

4. An object occupies a region of storage in its period of construction, throughout its lifetime, and in its period of destruction. The size of that region of storage is defined at compile-type by the type of the object.

### 3.1.1. Object lifetimes

1. The lifetime of an object of type `T` begins when:
   
    1. storage with the proper alignment and size for type `T` is obtained, and 

    2. its initialization is complete.

2. The lifetime of an object of type `T` ends when:
  
    1. its destructor has returned, and
    
    2. the storage which the object occupies is released, or is reused by another object.

## 3.2. Projections

1. A projection exposes an existing object.

2. An object `o1` projects an object `o2` if and only if:

    1. `o1` is a projection of `o2` or one of `o2`'s sub-objects; or
    
    2. a sub-object of `o1` projects `o2`.

3. When an object `o1` that projects an object `o2`, `o2` is said to be projected by o1`. [Note: Copying creates a new object that is not a projection.]

4. (Example)

    ```val
    type A { fun zero: Int { 0 } }

    fun main() {
      let x = A()
      let y = x.zero 
        // the value of 'y' projects that of 'x'
        // corollary: the value of 'x' is projected by that of 'y'
    }
    ```

5. The object yielded by an accessor projects the arguments bound to the projection's out parameters.

6. (Example)

    ```val
    fun min<T, E>(
      _ a: out T, _ b: out T, by less_than: [E] (T, T) -> Bool
    ): T {
      let { if less_than(a, b) { yield a } else { yield b } }
    }

    fun main() {
      let comparator = Int.< 
      let (x, y) = (2, 3)
      let z = min(x, y, by: comparator)
        // the value of 'z' projects both the values of 'x' and 'y',
        // but not that of 'comparator'
      print(z)
    }
    ```

    The expression `min(x, y, by: comparator)` is a call to the `let` accessor of `min`, which projects the values of its first and second arguments, but not that of its third argument.

7. A closure projects the objects it captures by projection.

8. An object projects the objects captured by the stored projections with which it has been initialized.

9.  If an object `o1` projects an object `o2` immutably, `o2` is immutable for the duration of `o1`'s lifetime. If an object `o1` projects an object `o2` mutably, `o2` is inaccessible for the duration of `o1`'s lifetime.

10. (Example) Mutable projection:

    ```val
    fun main() {
      var x = 42
      var y = x // mutable projection begins here
      print(x)  // error: the value of 'x' is projected mutably
      x += 1    // error: the value of 'x' is projected mutably
      print(y)  // mutable projection ends afterward
    }
    ```

11. (Example) Immutable projection:

    ```val
    fun main() {
      var x = 42
      let y = y // immutable projection begins here
      print(x)  // OK
      x += 1    // error: the value of 'x' is projected immutably
      print(y)  // immutable projection ends afterward
    }
    ```

## 3.3. Sinkability

1. A stored projection is unsinkable.

2. A value initialized with an immovable value is unsinkable.

3. (Example)

    ```val
    type Wrapper<T> { var wrapped: T }

    fun main() {
      let w = 2
      let x = Wrapper(wrapped: w.copy()) // the value of 'x' is sinkable
      let y = Wrapper(wrapped: [let w])  // the value of 'y' is unsinkable
      let z = Wrapper(wrapped: y)        // the value of 'z' is unsinkable
    }
    ```

## 3.4. Escapability

1. A sinkable object evaluated by a non-consuming expression is escapable at a given program point if it is not projected by any other object at that program point.

2. A sinkable object bound to an escapable binding is escapable at a given program point if it is not projected by any other object at that program point.

3. (Example)

    ```val
    fun print_and_return_forty_two() -> Int {
      sink let x = 42 // the value of 'x' is escapable here
      let y = x       // the value of 'x' is not escapable here
      print(y)        // the value of 'x' is escapable afterward
      return x        // the value of 'x' escapes here
    }
    ```

    The value of `y` is never escapable because `y` is not an escapable binding.

4. An escapable object may be consumed.

# 4. Declarations

## 4.1. Modifiers

### 4.1.1. General

1. A modifier may appear at most once in a declaration.

### 4.1.2. Member modifiers

1. Member modifiers have the form:

    ```ebnf
    member-modifier ::=
      receiver-modifier
      'static'
    
    receiver-modifier ::=
      'sink'
      'inout'
      'out'
    ```

2. The `static` may only apply to a binding declaration, a function declaration, or a subscript declaration at type scope.

3. A receiver modifier may only appear in a function declaration or a subscript declaration at type scope.

4. The `out` modifier may only appear in a subscript declaration.

## 4.2. Trait declarations

## 4.3. Binding declarations

### 4.3.1. General

1. Binding declarations have the form:

    ```ebnf
    binding-decl ::=
      binding-head binding-type-annotation? binding-initializer?

    binding-head ::=
      'sink'? binding-introducer pattern
    
    binding-introducer ::=
      'let'
      'var'
      'inout'
    
    binding-type-annotation ::=
      ':' type-expr
    
    binding-initializer ::=
      '=' expr
    ```

2. (Example)

    ```val
    let (name, age): (String, Int) = ("Thomas", 3)
    ```

    This binding declaration defines two new immutable bindings: `name` and `age`.

3. A binding declaration defines a new binding for each named pattern in pattern. All new bindings are defined with the same capabilities.

    1. A binding declaration introduced with `let` defines immutable bindings. The value of a live immutable binding may be projected immutably. The value of an immutable binding may not be projected mutably for the duration of that binding's lifetime.

    2. A binding declaration introduced with `var` and `inout` defines mutable bindings. The value of a live mutable binding may be projected mutably or immutably.

    3. A binding declaration introduced with `sink` defines an escapable binding. An escapable binding may only be assigned to sinkable objects. The value of an escapable binding may be consumed at a given program point if is escapable at that program point.

  A mutable binding can appear on the left side of an assignment, on the right side of an assignment to a non-escapable mutable binding, as the initializer of non-escapable mutable binding, as argument to an `inout` parameter, or as argument to an `assign` parameter. An escapable binding can appear as the

4. The pattern of a binding declaration may not contain any binding patterns and, unless it appears in the condition of a loop statement or in the condition of a selection expression, it may not contain any expression patterns.

5. A binding declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A binding declaration at module scope or namespace scope is called a global binding declaration. It introduces one or more global bindings. A global binding declaration must be introduced with `let`.
      
    2. A binding declaration at type scope is called a static member binding declaration if it contains a `static` modifier. Otherwise, it is called a member binding declaration. A static member binding declaration introduces one or more global bindings. A member binding declaration introduces one or more member bindingss.
      
    3. A binding declaration at function scope is called a local binding declaration. It introduces one or more local bindings.

6. The `sink` capability may only appear in a local binding declaration introduced with `let` or `var`.

### 4.3.2. Initialization

1. A static member binding declaration, or global binding declaration must contain an initializer. Initialization occurs at the first dynamic use of one of the declared bindings.

2. A local binding declaration must contain an initializer unless it appears in the condition of a match expression. If the binding declaration is a sub-statement in a brace statement, initialization occurs immediately after declaration. If the binding declaration is a condition in a loop statement or a condition in a selection expression, initialization occurs immediately after the a successful pattern matching.

3. A member binding declaration may not have an initializer. Initialization occurs in the constructor of the object of which the introduced bindings are members (see Type initialization).

4. The initialization of an escapable binding or a binding whose declaration is introduced with `var` consumes the value of its initializer.

5. The initialization of a non-escapable binding whose declaration is introduced with `let` or `inout` projects the object to which its initializer evaluates. The projection is immutable if the binding is immutable. Otherwise, it is mutable. The projection is for the duration of the binding's lifetime.

6. (Example)

    ```val
    fun main() {
      var fruits = ["apple", "mango", "orange"]
      inout first = fruits[0] // mutable projection of 'fruits[0]'
      first = "strawberry"
      print(first)            // projection ends afterward
    }
    ```

### 4.3.3. Lifetime

1. Binding lifetimes are not bound to lexical scopes.

2. A binding is said to be alive at a given program point if that program point falls within one of its lifetimes. Otherwise, it is said to be dead.

3. The lifetime of a global binding begins with its initialization is complete and ends when the program terminates.

4. The lifetime of a member binging `b` begins when the initialization of the object `o` of which it is a sub-object is completes and ends when `o` is consumed, or when `b` is consumed in a `sink` method of `o`.

5. The lifetime of a local binding begins when its initialization is complete and ends after its last use in an operation, or after any consuming operation.

6. (Example) Lifetime ending at last use.

    ```val
    fun main() {
      let count = 1 // lifetime of 'count' begins here
      count += 1    // a use of 'count'
      print(count)  // last use of 'count', lifetime ends afterward
      print("done")
    }
    ```

7. (Example) Lifetime ending because of a consuming operation.

    ```val
    fun main() {
      sink let count = 1 // lifetime of 'count' begins here
      sink _ = count     // lifetime of 'count' ends here
      print(count)       // error: 'count' has been consumed
    }
    ```

8. A dead immediate local mutable binding can be re-initialized, starting a new lifetime.

9. (Exemple) A binding with two lifetimes.

    ```val
    fun borrow<T>(_ thing: inout T) {
      let a = [thing]         // lifetime of 'thing' ends here
      print(a)
      thing = a.remove_last() // new lifetime of 'thing' starts here.
    }
    ```

    The lifetime of `thing` ends when `a` is initialized because construction an array literal consumes the literal's elements. A new lifetime starts before `borrow` returns as the call to `Array.remove_last` produces an independent value.

## 4.4. Function declarations

### 4.4.1. General

1. Function declarations have the form:

    ```ebnf
    fun-decl ::=
      fun-head fun-signature fun-body?

    fun-head ::=
      member-modifier* fun-ident generic-clause? capture-list?

    fun-ident ::=
      'init'
      'deinit'
      'infix'? 'fun' IDENT
      oper-notation 'fun' OPER

    fun-body ::=
      brace-stmt
      '{' method-impl+ '}'
      '=' 'default'
    ```

2. (Example)

    ```val
    fun gcd(_ a: Int, _ b: Int) -> Int {
      if b == 0 { a } else { gcd(b, a % b) }
    }
    ```

3. A function declaration introduces one or more function objects.    

4. A function declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A function declaration at module scope or namespace scope is called a global function declaration. It introduces one global function.
      
    2. A function declaration at type scope that contains a `static` modifier is called a static method declaration; it is also a global function declaration. A static method declaration introduces one global function.

    3. A function declaration at type scope that does not contain a `satic` modifier is called a method declaration. It introduces one or more methods.
      
    4. A function declaration at type scope declared with `init` is called a constructor declaration. It introduces one global function.

    5. A function declaration at type scope declared with `deinit` is called a destructor declaration. It introduces one sink method.
      
    6. A function declaration at function scope is called a local function declaration. It introduces a local function.

5. The `init` introducer and the `deinit` introducer may only appear in a function declaration at type scope.

6. A method implementation may only appear in a method declaration.

7. A declaration whose body has the form `= default` is called an explicitly defaulted. An explicitly default declaration may only be a constructor declaration in a product type declaration.

8.  An operator notation specifier defines an operator member function; it may only appear in a function declaration at type scope.

9.  A capture list may only appear in a function declaration at local scope.

### 4.4.2. Function signatures

1. Function signatures have the form:

    ```ebnf
    fun-signature ::=
      '(' param-list? ')' ('->' type-expr)?
    ```

2. The default value of a parameter declaration may not refer to another parameter in a function signature.

3. The output type of a function signture defines the output type of the containing declaration. If that type is omitted, the output type of the declaration is interpreted as `()`.

### 4.4.3. Function implementations

1. The brace statement in the body of a global or local function declaration defines its implementation. A global or local  function declaration must have a function implementation, unless it is static method declaration.

2. The parameters declared in the signature of the function declaration containing a function implementation define the parameters of that implementation.

3. All parameters of a function implementation are treated as immediate local bindings in that implementation, defined before its first statement. All parameters except `assign` parameters are alive before the first statement. Further:

    1. `sink` parameters are immutable and escapable; and

    2. `inout` parameters are mutable and escapable, and they must be alive at the end of every terminating execution path; and
    
    3. `assign` parameters are dead and mutable, and they must be alive at the end of every terminating execution path.

4. The output type of a function implementation is the output type of the containing function declaration, unless it is an explicit `inout` method implementation (see Method implementations). A function implementation must have a return statement on every terminating execution path, unless the output type of the containing declaration is `()`. In that case, explicit return statements may be omitted. A function implementation must return an escapable object whose type is a subtype of the output type of the containing method declaration.

5. The `return` keyword may be omitted if the body of the function implementation consists of a single expression.

6. (Example) Expression-bodied function

    ```val
    fun factorial(_ n: Int) -> Int {
      if n > 0 { n * factorial(n - 1) else { 1 } }
    }
    ```

### 4.4.4. Method declarations

1. A bodiless method declaration or a method declaration that contains a bodiless method implementation defines a trait method requirement and may only appear at trait scope. A bodiless static method declaration defines a static trait method requirement and may only appear at trait scope.

2. A method declaration may contain at most one method implementation of each kind. A method declaration that contains one or more explicit method implementations may not have a receiver modifier.

3. A non-bodiless method declaration must contain an explicit `let` method implementation, or its body must be a brace statement. In that case, that brace statement is interpreted as an implicit method implementation whose kind depends on the receiver modifier of the method declaration.

4. The declaration of a method `m` that contains an explicit `inout` method implementation is automatically provided with a synthesized `sink` method implementation, defined as follows:

    ```val
    sink {
      sink var this = self
      this.=m(arg1, ..., argn)
      return this
    }
    ```

5. The declaration of a method `m` that contains an explicit `sink` method implementation is automatically provided with a synthesized `inout` method implementation, defined as follows:

    ```val
    inout {
      self = m(arg1, ..., argn)
    }
    ```

6. (Example)

    ```val
    type Vector2 {

      var x: Double
      var y: Double

      fun scaled(by factor: Double) -> Self {
        let   { Vector2(x: x * factor, y: y * factor) }
        inout { x *= factor; y *= factor }
      }

      infix fun dot(_ other: Vector2) -> Double {
        x * other.x + y * other.y
      }

      inout fun transpose() {
        swap(&x, &y)
      }

    }
    ```

    The method `Vector2.scaled(by:)` has an explicit `let` implementation, an explicit `inout` implementation and a synthesized `sink` implementation. The method `Vector2.dot(_:)` has an implicit `let` implementation. The method `Vector2.transpose` has an implicit `inout` implementation.

### 4.4.5. Method implementations

1. A method implementation is a function implementation defined in a method. It may be defined implicitly or explicitly (see Method declarations). Explicit method implementations have the form:

    ```ebnf
    method-impl ::=
      method-introducer brace-stmt?
    
    method-introducer ::=
      'let'
      'sink'
      'inout'
    ```

2. An explicit method implementation introduced with `let` is called a `let` method implementation; one introduced with `sink` is called a `sink` method implementation; one introduced with `inout` is called an `inout` method implementation.

3. The parameters declared in the signature of the method declaration containing a method implementation define the parameters of that implementation. An additional implicit parameter, called the receiver parameter, and named `self`, represents the method receiver.

4. In an explicit method implementation, the passing convention of the receiver parameter corresponds to the kind of the method implementation: it is `let` in a `let` implementation; it is `sink` in a `sink` implementation; it is `inout` in a `inout` implementation. In an implicit method implementation, the passing convention of the receiver parameter is defined by the receiver modifier of the containing method declaration.
 
5. The output type of an explicit `inout` method implementation is `()`.

## 4.5. Subscript declarations

### 4.5.1. General

1. Subscript declarations have the form:

    ```ebnf
    subscript-decl ::=
      subscript-head subscript-signature subscript-body
    
    subscript-head ::=
      member-modifier* subscript-ident? generic-clause? capture-list?

    subscript-ident ::=
      'infix'? 'subscript' IDENT
      oper-notation 'subscript' OPER

    subscript-body ::=
      brace-stmt
      '{' subscript-impl+ '}'
    ```

2. (Example)

    ```val
    subscript min<T, E>(_ a: T, _ b: T, by comparator: [E](T, T) -> Bool): Int {
      let    { if comparator(a, b) { yield a } else { yield b } }
      inout  { if comparator(a, b) { yield &a } else { yield &b } }
      assign { if comparator(a, b) { yield &a } else { yield &b } }
      sink   { if comparator(a, b) { return a } else { return b } }
    }
    ```

3. A subscript declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A subscript declaration at module scope or namespace scope introduces a global subscript declaration. It introduces a global subscript.
      
    2. A subscript declaration at type scope that contains a `static` modifier is called a static subscript declaration; it is also a global subscript declaration. A static member subscript declaration introduces a global susbcript.

    3. A subscript declaration at type scope that does not contain a `static` modifier is called a member subscript declaration. It introduces a member subscript. A member subscript declaration may not have a receiver modifier.
      
    4. A subscript declaration at function scope is a local subscript declaration. It introduces a local subscript.

4. A member subscript declaration or a static member subscript declaration without an identifier is called a nameless subscript declaration. It introduces a nameless subscript. A nameless subscript declaration must have an explicit parameter list.

5. A member subscript declaration or a static member subscript declaration may be defined without an explicit parameter list. Such a declaration is called a property subscript declaration. It introduces a property subscript.

6. A bodiless subscript declaration or a subscript declaration that contains a bodiless subscript implementation defines a trait subscript requirement and may only appear at trait scope. A bodiless static subscript declaration defines a static trait subscript requirement and may only appear at trait scope.

7. A subscript declaration may contain at most one subscript implementation of each kind.

8. If the body of a subscript declaration is a brace statement, it is interpreted as the body of a `let` subscript implementation.

9. An operator notation specifier defines an operator member subscript; it may only appear in a subscript declaration at type scope.

10.  A capture list may only appear in a subscript declaration at local scope.

### 4.5.2. Subscript signatures

1. Subscript signatures have the form:

    ```ebnf
    subscript-signature ::=
      explicit-subscript-param-list? ':' 'var'? type-expr

    explicit-subscript-param-list ::=
      '(' param-list? ')'
    ```

2. The default value of a parameter declaration may not refer to another parameter in a subscript signature.

3. The output type of a subscript signture defines the output type of the containing declaration. If that type is prefixed by `var`, all projections produced by the subscript are mutable. Otherwise, only the projections produced by the `inout` implementation of the subscript are mutable.

### 4.5.3. Subscript implementations

1. A subscript implementation may be defined ikmplicitly or explicitly. Explicit subscript implementations have the form:

    ```ebnf
    subscript-impl ::=
      subscript-introducer brace-stmt?
    
    method-introducer ::=
      'let'
      'sink'
      'inout'
      'assign'
    ```

2. Am explicit subscript implementation introduced with `let` is called a `let` subscript implementation; one introduced with `sink` is called a `sink` subscript implementation; one introduced with `inout` is called an `inout` subscript implementation; one introduced with `assign` is called an `assign` subscript implementation.

3. The parameters declared in the signature of the subscript declaration containing a subscript implementation define the parameters of that implementation. In a member subscript declaration, an additional implicit `out` parameter, called the receiver parameter, named `self`, and representing the subscript receiver.

4. The passing convention of an `out` parameter dependends on the kind of the sibscript implementation: it is a `let` parameter in a `let` subscript implementation; or it is a `sink` parameter in an `sink` subscript implementation; or it is an `inout` parameter in an `inout` subscript implementation; or it is an `assign` parameter in an `assign` subscript implementation.

5. All parameters of a subscript implementation are treated as immediate local bindings in that implementation, defined before its first statement. All parameters except `assign` parameters are alive before the first statement. Further:

    1. `sink` parameters are immutable and escapable.

    2. `inout` parameters are mutable and escapable. They must be alive at the end of every terminating execution path.
    
    3. `assign` parameters are dead and mutable. They must be alive at the end of every terminating execution path.

6.  A `let` subscript implementation or an `inout` subscript implementation must have exactly one a yield statement on every terminating execution path. Given a subscript declaration with an output type `T`, a `let` subscript implementation must yield an immutable projection of an object whose type is subtype of `T`, unless the output signature of the subscript declaration is prefixed by `var`. In that case it must yield an mutable projection of an object of type `T`. An `inout` subscript implementation must yield a mutable projection of an object of type `T`.

7. A `sink` subscript implementation must have a return statement on every terminating execution path. It must return an escapable object whose type is subtype of the output type of the containing subscript declaration.

## 4.6. Parameter declarations

1. Parameter declarations have the form:

    ```ebnf
    param-list ::=
      param-decl (',' param-decl)?
      
    param-decl ::=
      (IDENT | '_') IDENT? (':' param-type-expr)? default-value?

    default-value ::=
      '=' expr
    ```

1. If the declaration contains two identifiers, the first is used as an argument label and the second is used as a parameter name. Otherwise, the same identifier is used as both an argument label and as a parameter name. If the declaration contains a wildcard followed by an identifier, it defines a positional parameter. The identifier is used as a parameter name.

2. Parameter declarations define the passing convention of the arguments to the declared parameters in a function call:

    1. A parameter declared without any explicit convention is called a `let` parameter.

    2. A parameter declared with the `sink` convention is called a `sink` parameter.
    
    3. A parameter declared with the `inout` convention is called an `inout` parameter.
    
    4. A parameter declared with the `assign` convention is called an `assign` parameter.

3. A `let` parameter of a `sink` parameter may have a default value.

4. A default value must be a non-consuming expression. A default value to a `sink` parameter must evlauate to an escapable object.

# 5. Statements

## 5.1. General

1. Statements of the form:

    ```ebnf
    stmt ::=
      brace-stmt
      loop-stmt
      jump-stmt
      decl
      expr
    ```

2. Except as indicated, statements are executed in sequence.

3. Statements do not require explicit statement delimiter. Semicolons can be used to separate statements explicitly, for legibility or to disambiguate exceptional situations. [Note: A common practice is to write each statement on a new line.]

## 5.2. Brace statements (a.k.a. code blocks)

1. Brace statements are sequences of statements executed in a lexical scope.

2. Brace statements have the form:

    ```ebnf
    brace-stmt ::=
      '{' stmt-list? '}'

    stmt-list ::=
      stmt (';'* stmt)+ ';'*
    ```

3. The statements contained in a brace statements are called its sub-statements.

4. Control enters the lexical scope of a brace statement before executing any sub-statements and exits that lexical scope when it reaches the end of the brace statement.

## 5.3. Loop statments

### 5.3.1. General

1. Loop statements describe iteration. They have the form:

    ```ebnf
    loop-stmt ::=
      loop-label? do-while-stmt
      loop-label? while-stmt
      loop-label? for-stmt

    loop-label ::=
      IDENT ':'
    ```

2. A loop statement introduces a lexical scope. The body of a loop is a brace statement lexically nested inside the loop's scope.

3. A loop has three control entry points: a head, a body, and a tail. Control enters a loop from its head. The head belongs to the loop scope and the tail belongs to the body's scope. When a loop statement is executed, control is transferred to its head, entering the loop's scope. If control reaches the end of a loop's body, it is unconditionally transferred to its tail. The body's scope and then the loop's scope are exited when control exits the tail, whether or not it is transferred back to the head.

4. A continuation test is a procedure that determines whether an additional iteration should take place, or whether control should exit the loop. A continuation test may take in the head or in the body of a loop.

### 5.3.2. Do-while statments

1. `do-while` statements have the form:

    ```ebnf
    do-while-stmt ::=
      'do' brace-stmt 'while' expr
    ```

2. The condition of a `do-while` statement belongs to the tail of the loop. It must be an expression of type `Bool`, which is evaluated by each continuation test. The test succeeds if the condition evaluates to `true`. That value is not consumed.

3. (Example)

    ```val
    var counter = 0
    do {
      counter += 1
      let x = counter
    } while x < 3
    ```

    The binding `x` that occurring in the condition of the `do-while` statement is declared it its body.

4. The head of a `do-while` statement unconditionally transfers control to the body of the loop. The tail performs a continuation test. If it succeeds, control is transferred back to the head. Otherwise, it exits the loop.

### 5.3.3. While statments

1. `while` statements have the form:

    ```ebnf
    while-stmt ::=
      'while' while-condition-list brace-stmt
    
    while-condition-list ::=
      while-condition-item (',' while-condition)*

    while-condition-item ::=
      binding-decl
      expr
    ```

2. The condition of a `while` belongs to the head of the loop. It is a non-empty sequence of condition items. A condition item that is a binding declaration is considered satisfied if and only if the value of the initializer matches the pattern and can initialize its new bindings. A condition item that is an expression must be of type `Bool` and is considierd satisfied if and only if it evaluates to `true`. That value is not consumed. If the condition contains more than a single item, the n+1th item is evaluated if and only if the nth item is satisfied. The continuation test succeeds if and only if all items are satisfied.

3. The head of a `while` statement performs a continuation test. If it succeeds, control is transferred to the body of the loop. Otherwise, it exits the loop. The tail unconditionally transfers control back to the head.

### 5.3.4. For statements

1. `for` statements have the form:

    ```ebnf
    for-stmt ::=
      'for' for-binding-decl for-range loop-filter? brace-stmt

    for-binding-decl ::=
      binding-head binding-type-annotation?

    for-range ::=
      'in' expr

    loop-filter ::=
      'where' expr
    ```

2. (Example)

    ```val
    fun main() {
      var things: Array<Any> = [1, "abc", 3, 2]
      for inout x: Int in things where x < 3 {
        x += 1
      }
      print(things) // [2, "abc", 3, 3]
    }
    ```

3. The binding declaration and the filter expression of a `for` statement belong to the head of the loop. The range of a `for` statement belongs to the lexical scope in which that statement is defined.

4. (Example)

    ```val
    fun main() {
      for let x in 0 until x { print(x) }
    }
    ```

    This program is ill-typed. The range is referring to a binding introduced in the loop binding.

5. The range must be an expression of a type that conforms to the `Iterable` trait. When a `for` statement is executed, an iterator object `it` is produced by calling `make_iterator` on the object evaluated by the loop range. The continuation test consists of verifying that `it.next()` does not result in a `nil` object. The loop iterates over all elements produced by the iterator, unless it is exited early via a break or a return statement.

6. The head of a `for` statement performs a continuation test. If it fails, control exits the loop. Otherwise, it tests whether the latest object projected by the loop's iterator matches the pattern of the binding declaration and can initialize its new bindings. If and only if it does, the loop filter is evaluated. If and only if the filter evaluates to `true`, control enters the body of the loop. The value of the filter is not consumed. If either the pattern of the binding declaration does not match, or the filter is not satisfied, control is transferred back the head.

7.  (Example)

    ```val
    fun main() {
      var things: Array<Any> = [1, "abc", 3, 2]
      for inout x: Int in things where x < 3 {
        x += 1
      }
      print(things) // [2, "abc", 3, 3]
    }
    ```

    This program can be understood as though it had be written:

    ```val
    fun main() {
      var things: Array<Any> = [1, "abc", 3, 2]
      var iterator = things.make_iterator()
      while true {
        if inout x: Int = iterator.next() where x < 3 {
          x += 1
        } else {
          break
        }
      }
      print(things) // [2, "abc", 3, 3]
    }
    ```

## 5.4. Jump statements

### 5.4.1. General

1. Jump statements unconditionally transfer control. They have the form:

    ```ebnf
    jump-stmt ::=
      'return' expr?
      'yield' expr
      'break' jump-stmt-label?
      'continue' jump-stmt-label?

    jump-stmt-label ::=
      IDENT
    ```

2. `break` and `continue` statements are called loop jump statements. A loop jump statement applies to the innermost loop unless it defines a loop label. In that case, it applies to the innermost loop defined with the same label.

3. (Example)

    ```val
    var numbers: Array<Int> = []
    outer:for let i in 1 to 3 {
      for let j in 1 to 3 {
        if j % i == 0 { continue outer }
        numbers.append(j.copy())
      }
    }
    ```

    After executing these statements, `numbers` contains the sequence `1, 1, 2`.

### 5.4.2. Return statments

1. Return statements return an object from a function, terminating the execution path and transferring control back to the function's caller.

2.  The expression in a return statement is called its operand. If the operand is omitted, it is interpreted as `()`. A return statement consumes the value of the operand to initialize an escapable object as result of the call to the containing function.

### 5.4.3. Yield statments

1. Yield statements project an object out of a subscript, suspending the execution path and temporarily transferring control to the subscript's caller. Control comes back to the subscript once after the last use of the yielded projection at the call site, resuming execution at the statement that directly follows the yield statement.

2. (Example)

    ```val
    subscript element<T>(at: index, in array: Array<T>): T {
      print("will yield")
      yield array[position]
      if a > b { yield a } else { yield b }
      print("did yield")
    }

    fun main() {
      let fruits = ["apple", "mango", "orange"]
      let f = element(at: 1, in: fruits) // "will yield"
      print("foo")                       // "foo"
      print(f)                           // "mango"
                                         // "did yield"
      print("bar")                       // "bar"
    }
    ```

3. The expression in a yield statement is called its operand. A yield statement projects the value of its operand as the result of the call to the containing subscript. The yielded object is projected immutably in a `let` subscript implementation and mutably in an `inout` subscript implementation. The mutability marker `&` must prefix a mutable projection.

### 5.4.4. Break statements

1. Break statements exit a loop. Control is transferred to the statement immediately following the loop, if any.

### 5.4.5. Continue statements

1. Continue statements skip the remainder of a loop body. Control is transferred to the begin of the loop.

# 6. Value expressions

## 6.1. General

1. An expression is a sequence of operators and operands that specifies a computation. An expression results in a value and may cause side effects.

2. If during the evaluation of an expression, the result is not mathematically defined or not in the range of representable values for its type, the behavior is undefined.

## 6.2. Properties of expressions

### 6.2.1. Consuming expressions

1. An expression is consuming if and only if its evaluation may end the lifetime of one or objects not created by the expression's evaluation.

## 6.3. Primary expressions

### 6.3.1. General

1. Primary expressions have the form:

    ```ebnf
    primary-expr ::=
      scalar-literal
      compound-literal
      primary-decl-ref
      implicit-member-ref
      capture-expr
      lambda-expr
      async-expr
      await-expr
      selection-expr
      tuple-expr
      'nil'
      '_'
    ```

### 6.3.2. Scalar literals

1. Scalar literals have the form:

    ```ebnf
    scalar-literal ::=
      BOOL-LITERAL
      INT-LITERAL
      FLOAT-LITERAL
      STRING-LITERAL
    ```

### 6.3.3. Compound literals

#### 6.3.3.1. General

1. Compound literals have the form:

    ```ebnf
    compound-literal ::=
      buffer-literal
      map-literal
    ```

2. A compound literal consumes the value of each of its components.

#### 6.3.3.2. Buffer literals

1. Buffer literals have the form:

    ```ebnf
    buffer-literal ::=
      '[' buffer-component-list? ']'
    
    buffer-component-list ::=
      expr (',' expr)* ','?
    ```

2. The type of a buffer literal is `T[n]`, where `n` is the number of components in the literal and `T` the type of all components. If a buffer literal appears in a typed context, `T` may be inferred from that context. Otherwise, the literal may not be empty, the type of `T` is inferred from the first component, and all other components must have the same type. The implementation may issue a warning if the literal has no component.

3. (Example)

    ```val
    let a = []               // error: cannot infer empty buffer type without context
    let b = [1, 2]           // OK, 'b' has type 'Int[2]'
    let c = [1, 2.0]         // error: '2.0' does not have type 'Int'
    let d: Double[] = [1, 2] // OK, 'd' has type 'Double[2]'
    let e: Int[] = []        // warning: zero-lenght buffer
    ```

#### 6.3.3.3. Map literal

1. Map literals have the form:

    ```ebnf
    map-literal ::=
      '[' map-literal-component-list ']'
      '[' ':' ']'
    
    map-component-list ::=
      map-component (',' map-component)* ','?
  
    map-component ::=
      expr ':' expr
    ```

2. A map literal is said to be empty if it has the form `[:]`.

3. The type of a map literal is `Map<Key, Value>`, where `Key` is the type of the map's keys and `Value` is the type of the map's values. If a map literal appears in a typed context, `Key` and `Value` may be inferred from that context. Otherwise, the literal may not be empty, the types of `Key` and `Value` are inferred from the first component, and all other components must have the same type.

4. (Example)

    ```val
    let a = [:]                // error: cannot infer empty map type without context
    let b = [1: "a", 2: "b"]   // OK, 'b' has type 'Map<Int, String>'
    let c = [1: "a", 2.0: "b"] // error: '2.0' does not have type 'Int'
    let d: Map<Double, String> = [1: "a", 2.0: "b"] // OK
    let e: Map<Double, String> = [:] // OK
    ```

### 6.3.4. Primary declaration references

1. Primary declaration references have the form:

    ```ebnf
    primary-decl-ref ::=
      ident-expr type-argument-list?
    ```

### 6.3.5. Identifiers

1. Identifiers have the form:

    ```ebnf
    ident-expr ::=
      entity-ident impl-ident?
    
    entity-ident ::=
      IDENT
      fun-entity-ident
      oper-notation OPER

    fun-entity-ident ::=
      IDENT '(' argument-label+ ')'

    argument-label ::=
      (IDENT | '_') ':'

    impl-ident ::=
      '#' method-introducer
    ```

2. An identifier may not have any whitespace between its constituent tokens.

3. An identifier that denotes a non-static binding member or a non-static method may only be used as part of a value member access. An identifier that denotes a static binding member or a static method may only be used as part of a type member access.

4. (Example)

    ``val
    type A {
      let m: Int
      static let n = 0
    }

    let foo = A(m: 0)
    let i = foo.m   // OK
    let j = A.m     // OK
    let k = foo.n   // error
    let l = A.n     // OK
    ```

5. An identifier may contain be suffixed by a method introducer if and only if its entity identifier refers to a method declaration for which an explicit or synthesized method implementation for the same method introducer exists.

6. (Example)

    ```val
    type Vector2 {

      var x: Double
      var y: Double

      fun scaled(by factor: Double) -> Self {
        let   { Vector2(x: x * factor, y: y * factor) }
        inout { x *= factor; y *= factor }
      }

    }

    let f = Vector2.scaled(by:)#let  // OK
    let g = Vector2.scaled(by:)#sink // OK
    ```

## 6.4. Compound expressions

### 6.4.1. Member accesses

#### 6.4.1.1. General

1. Member accesses have the form:

    ```ebnf
    value-member-expr ::=
      expr '.' primary-decl-ref
      type-expr '.' primary-decl-ref
    ```

2. A member access whose declaration reference denotes a non-static member binding or a non-static method is called a value member access. A member access whose declaration reference denotes a a static binding member or a static method is called a type member access.

3. A value member access whose base is an expression is called an unbound value member access.

### 6.4.2. Function calls

#### 6.4.2.1. General

1. Function calls have the form:

    ```ebnf
    call-expr ::=
      expr '(' call-argument-list? ')'

    call-argument-list ::=
      call-argument ( ',' call-argument )*

    call-argument :=
      (IDENT ':')? expr
    ```

    The opening parenthesis preceding the call argument list must be on the same line as the callee.

2. The kind of the call expression depends on its callee:

    1. if the callee is a type expression, the call expression is an initializer call; or

    2. if the callee is a value member expression, the call expression is a method call; or

    3. if the callee is a type member expression referring to a function declaration, the call expression is a static method call; or

    4. otherwise, the call expression is a function call.

3. Arguments to `sink` parameters are consumed. Arguments to `let` parameters are projected immutably in the entire call expression. Arguments to `inout` and `set` parameters are projected mutably in the entire call expression.

## 6.5. Operators

### 6.5.1. Operator notations

1. Operator notations have the form:

    ```ebnf
    oper-notation ::=
      'infix'
      'prefix'
      'postfix'
    ```