# 1. Introduction

Val is a research language based on the principles of mutable value semantics (MVS) (Racordon et al. 2022) for safety and efficiency. It is designed to help developers write and maintain correct programs using powerful abstractions without loss of efficiency.

Val is strongly related to the [Swift programming language](https://www.swift.org) and borrows liberally from its syntax and semantics. Nonetheless, it exposes a more elaborated memory model to the user for a tighter control over memory, blending ideas found in other languages such as Rust, C++, and Go.

On a theoretical front, Val owes greatly to linear types [(Wadler 1990)](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.31.5002&rep=rep1&type=pdf), ownership types [(Clarke et al. 2013)](https://doi.org/10.1007/978-3-642-36946-9_3), and capability-based type systems [(Smith et al. 2000)](https://doi.org/10.1007/3-540-46425-5_24), while striving to hide the complexity inherent to these approaches. It does so by excluding first-class references from the user model and using continuations to mitigate the loss of expressiveness.

# 2. General concepts

## 2.1. Objects

1. An object is the result of a scalar literal expression, the result of an aggregate literal expression, the result of a function call, the result of a call to a `sink` accessor, or the value of an escapable binding.

2. An object has a type determined at compile-time. That type might be polymorphic; in that case, the implementation generates information associated with the object that makes it possible to determine its concrete type at runtime.

3. An object may contain other objects, called sub-objects.

4. An object occupies a region of storage in its period of construction, throughout its lifetime, and in its period of destruction. The size of that region of storage is defined at compile-type by the type of the object.

### 2.1.1. Object lifetimes

1. The lifetime of an object of type `T` begins when:
   
    1. storage with the proper alignment and size for type `T` is obtained, and 

    2. its initialization is complete.

2. The lifetime of an object of type `T` ends when:
  
    1. its destructor has returned, and
    
    2. the storage which the object occupies is released, or is reused by another object.

## 2.2. Projections

1. A projection exposes an existing object.

2. An object `o1` projects an object `o2` if and only if:

    1. `o1` is a projection of `o2` or one of `o2`'s sub-objects; or
    
    2. a sub-object of `o1` projects `o2`.

3. When an object `o1` that projects an object `o2`, `o2` is said to be projected by o1`.

    *(Note: Copying creates a new object that is not a projection.)*

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

9. If an object `o1` projects an object `o2` immutably, `o2` is immutable for the duration of `o1`'s lifetime. If an object `o1` projects an object `o2` mutably, `o2` is inaccessible for the duration of `o1`'s lifetime.

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

## 2.3. Sinkability

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

## 2.4. Escapability

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

# 3. Declarations

## 3.1. Modifiers

### 3.1.1. General

1. A modifier may appear at most once in a declaration.

### 3.1.2. Member modifiers

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

## 3.2. Trait declarations

## 3.3. Binding declarations

### 3.3.1. General

1. Binding declarations have the form:

    ```ebnf
    binding-decl ::=
      binding-head binding-type-annotation? binding-initializer?

    binding-head ::=
      'sink'? binding-introducer pattern
    
    binding-introducer ::=
      'let'
      'var'
    
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

    2. A binding declaration introduced with `var` defines mutable bindings. The value of a live mutable binding may be projected mutably or immutably.

    3. A binding declaration introduced with `sink` defines an escapable binding. An escapable binding may only be assigned to sinkable objects. The value of an escapable binding may be consumed at a given program point if is escapable at that program point.

  A mutable binding can appear on the left side of an assignment, on the right side of an assignment to a non-escapable mutable binding, as the initializer of non-escapable mutable binding, as argument to an `inout` parameter, or as argument to an `assign` parameter. An escapable binding can appear as the

4. The pattern of a binding declaration may not contain any expression patterns or binding patterns.

5. A binding declaration may be defined at module scope, namespace scope, type scope, or function scope.

    1. A binding declaration at module scope or namespace scope is called a global binding declaration. It introduces one or more global bindings.
      
    2. A binding declaration at type scope is called a static member binding declaration if it contains a `static` modifier. Otherwise, it is called a member binding declaration. A static member binding declaration introduces one or more global bindings. A member binding declaration introduces one or more member bindingss.
      
    3. A binding declaration at function scope is called a local binding declaration. It introduces one or more local bindings.

6. The `sink` capability may only appear in a local binding declaration.

### 3.3.2. Initialization

1. A local binding declaration, a static member binding declaration, or global binding declaration must contain a initializer. Initialization occurs immediately after declaration.

2. A member binding declaration may not have an initializer.

3. The initialization of an escapable binding consumes the value of its initializer. Initialization occurs in the constructor of the object of which the introduced bindings are members (see Type initialization).

4. The initialization of a non-escapable binding projects the object to which its initializer evaluates. The projection is immutable if the binding is immutable. Otherwise, it is mutable. The projection is for the duration of the binding's lifetime.

5. (Example)

    ```val
    fun main() {
      var fruits = ["apple", "mango", "orange"]
      var first = fruits[0] // mutable projection of 'fruits[0]'
      first = "strawberry"
      print(first)          // projection ends afterward
    }
    ```

### 3.3.3. Lifetime

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

## 3.4. Function declarations

### 3.4.1. General

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

    1. A function declaration at module scope or namespace scope is called a global function declaration. It introduces a global function.
      
    2. A function declaration at type scope is called a static method declaration if it contains a `static` modifier. Otherwise, it is called a method declaration. A static method declaration introduces a global function. A method declaration introduces one or more methods.
      
    3. A function declaration at type scope declared with `init` is called a constructor declaration. It introduces a global function.

    4. A function declaration at type scope declared with `deinit` is called a destructor declaration. It introduces a sink method.
      
    5. A function declaration at function scope is called a local function declaration. It introduces a local function.

5. The `init` introducer and the `deinit` introducer may only appear in a function declaration at type scope.

6. A method implementation may only appear in a method declaration.

7. A declaration whose body has the form `= default` is called an explicitly defaulted. An explicitly default declaration may only be a constructor declaration in a product type declaration.

8.  An operator notation specifier defines an operator member function; it may only appear in a function declaration at type scope.

9.  A capture list may only appear in a function declaration at local scope.

### 3.4.2. Function signatures

1. Function signatures have the form:

    ```ebnf
    fun-signature ::=
      '(' param-list? ')' ('->' type-expr)?
    ```

2. The default value of a parameter declaration may not refer to another parameter in a function signature.

### 3.4.3. Method declarations

1. A method declaration must contain a `let` method implementation.

2. A method declaration may contain at most one method implementations of each kind.

3. A method declaration for a method `m` that contains an `inout` method implementation is automatically provided with a synthesized `sink` method implementation, defined as follows:

    ```val
    sink {
      sink var this = self
      this.=m(arg1, ..., argn)
      return this
    }
    ```

4. A method declaration that contains one or more explicit method implementations may not have a receiver modifier.

5. If the body of a method declaration is a brace statement, it is interpreted as the body of an implicit method implementation. The kind of that method implementation depends on the receiver modifier of the method declaration. If the method declaration does not have a receiver modifier, its implicit method implementation is `let`. Otherwise, it has the kind corresponding to the receiver modifier.

6. If a method declaration contains an explicit `inout` method implementation, 

7. (Example)

    ```val
    type Vector2 {

      var x: Double
      var y: Double

      fun scaled(by factor: Double) -> Vecto2 {
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

    The method `Vector2.scaled(by:)` has an explicit `let` implementation, an explicit `inout` implementation and a synthesized `sink` implementation. The method `Vector2.dot(_:)` has an implicit `let` implementation. The method `Vector2.transpose` has an implicit `inout` implementation and a synthesized `sink` implementation.

8. A bodiless method declaration or a method declaration that contains a bodiless method implementation defines a trait method requirement and may only appear at trait scope. A bodiless static method declaration defines a static trait method requirement and may only appear at trait scope.

### 3.4.4. Method implementations

1. Method implementations have the form:

    ```ebnf
    method-impl ::=
      method-introducer brace-stmt?
    
    method-introducer ::=
      'let'
      'sink'
      'inout'
    ```

2. A method implementation introduced with `let` is called a `let` method implementation; one introduced with `sink` is called a `sink` method implementation; one introduced with `inout` is called an `inout` method implementation.

3. The parameters declared in the signature of the method declaration containing a method implementation define the parameters of that implementation. An additional implicit parameter, called the receiver parameter, and named `self`, represents the method receiver.

4. The passing convention of the receiver parameter corresponds to the kind of the method implementation.
 
5. All parameters of a method implementation are treated as immediate local bindings in that implementation, defined before its first statement. All parameters except `assign` parameters are alive before the first statement. Further:

    1. `sink` parameters are immutable and escapable; and

    2. `inout` parameters are mutable and escapable, and they must be alive at the end of every terminating execution path; and
    
    3. `assign` parameters are dead and mutable, and they must be alive at the end of every terminating execution path.

6. The output type of a method declaration is the type declared as the return type defined by the function signature of the containing method declaration.

7. A method implementation must have a return statement on every terminating execution path, unless its output type is `()`. The `return` keyword may be omitted if the body of the method implementation consists of a single expression.

8. A `let` method implementation or a `sink` method implementation must return an escapable object whose type is a subtype of its return type

## 3.5. Subscript declarations

### 3.5.1. General

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

    1. A subscript declaration at module scope or namespace scope introduces a global subscript.
      
    2. A subscript declaration at type scope is called a static subscript declaration if it contains a `static` modifier. Otherwise, it is called a member subscript declaration. A static subscript declaration introduces a global subscript. A member subscript declaration introduces a member subscript.
      
    3. A subscript declaration at function scope introduces a local subscript.

4. A member subscript declaration without an identifier is called a nameless subscript declaration. It introduces a nameless subscript.

5. If the body of a subscript declaration is a brace statement, it is interpreted as the body of a `let` subscript implementation and the subscript declaration is interpreted as having a single subscript implementation.

6. A subscript declaration that contains a bodiless subscript implementation defines a trait subscript requirement and may only appear at trait scope.

7. An operator notation specifier defines an operator member subscript; it may only appear in a subscript declaration at type scope.

8. A capture list may only appear in a subscript declaration at local scope.

### 3.5.2. Subscript signatures

1. Subscript signatures have the form:

    ```ebnf
    subscript-signature ::=
      '(' param-list? ')' ':' type-expr
    ```

2. The default value of a parameter declaration may not refer to another parameter in a subscript signature.

### 3.5.3. Subscript implementations

1. Subscript implementations have the form:

    ```ebnf
    subscript-impl ::=
      subscript-introducer brace-stmt?
    
    method-introducer ::=
      'let'
      'sink'
      'inout'
      'assign'
    ```

2. A subscript implementation introduced with `let` is called a `let` subscript implementation; one introduced with `sink` is called a `sink` subscript implementation; one introduced with `inout` is called an `inout` subscript implementation; one introduced with `assign` is called an `assign` subscript implementation.

3. A subscript declaration may not contain multiple subscript implementations of the same kind.

4. The parameters declared in the signature of the subscript declaration containing a subscript implementation define the parameters of that implementation. An additional implicit parameter, called the receiver parameter, named `self`, and representing the method receiver is provided to the subscript implementations of a member subscript. The passing convention of the receiver parameter is defined by the receiver modifier of the member subscript declaration.

5. The passing convention of an `out` parameter dependends on the kind of the sibscript implementation:

    1. it is a `let` parameter in a `let` subscript implementation; or

    2. it is a `sink` parameter in an `sink` subscript implementation; or
    
    3. it is an `inout` parameter in an `inout` subscript implementation; or

    4. it is an `assign` parameter in an `assign` subscript implementation.

6. All parameters of a subscript implementation are treated as immediate local bindings in that implementation, defined before its first statement. All parameters except `assign` parameters are alive before the first statement. Further:

    1. `sink` parameters are immutable and escapable.

    2. `inout` parameters are mutable and escapable. They must be alive at the end of every terminating execution path.
    
    3. `assign` parameters are dead and mutable. They must be alive at the end of every terminating execution path.

7. The output type of a subscript declaration is the type declared as the yielded type of the containing subscript declaration.

8. A `let` subscript implementation or an `inout` subscript implementation must have a yield statement on every terminating execution path. A `let` subscript implementation must yield objects whose types are subtype of the implementation's output type. An `inout` subscript implementation must yield objects whose types are the implementation's output type.

9. A `sink` subscript implementation must have a return statement on every terminating execution path. It must return escapable objects whose types are subtype of the implementation's output type

## 3.6. Parameter declarations

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

# 4. Expressions

## 4.1. Consuming expressions

1. An expression is consuming if and only if its evaluation may end the lifetime of one or objects not created by the expression's evaluation.
