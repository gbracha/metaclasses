# Metaclasses in Dart


## Contact information

1. **Gilad Bracha.** 

2. **gbracha@google.com.** 

3. **https://github.com/gbracha/metaclasses** 

## Summary

Currently all classes in Dart have the same (meta)class: `Type`. `Type` has very limited capabilities. It was designed to serve as a key for the mirror system and very little else. As anticipated in the original design document, requests keep coming in for more features. I think it is time we started to address these.

I propose that the type object for a class *C* have implicitly defined instance methods corresponding to the static methods (including getters and setters) and constructors of *C*.
For each static method *sm*, an instance method of the same name, with the same signature, is defined, which simply forwards the call to the static method *sm*. For each named constructor *k*, an instance method of the same name, with the same signature, is defined. This method returns the result of calling the constructor via **new**.

Since the named constructors, static methods and instance methods cannot conflict, there is no risk of these synthetic methods conflicting with the existing instance methods of `Type`, which are all inherited from `Object`, or with each other. The anonymous constructor should be represented by a method called `new`; since **new** is a reserved word, no conflicts arise here either.

The above implies that every `Type` object is an instance of its own metaclass and that metaclasses do not mirror the inheritance hierarchy. Each metaclass is a direct subclass of `Type`. 

Since each instantiation of a generic has its own `Type` object, each one also has its own metaclass.

Every class *C* induces two types. The *instance type*, referred to by the name *C*, is the type that already exists in Dart. This proposal introduces the notion of a *class type*. The class type is written *C.class* (again, since **class** is a reserved word, there is no conflict with existing members). The members of *C.class* have the the above defined methods of the metaclass of *C*. *C.class* is an interface type as usual in Dart, and can be implemented but not extended.

Note: one can now do constructor tear-offs using the normal tear-off mechanism. There is the overhead of indirection, which presumably will be inlined away if it matters. Likewise, if we have type variables, one can write *T.new()* instead of the illegal **new** *T()*.


## Motivation

People need to abstract over the operation of creating an instance of a class. In principle, this is quite straightforward, as all one needs to do is to define a closure for this purpose and pass it around. And yet users find this to be annoying or challenging.

See for example

https://code.google.com/p/dart/issues/detail?id=10659

We have also heard similar requests from internal customers at Google.

The restrictions on the the use of `Type` values confuse people. It is very intuitive to be able to pass classes as parameters and use them as classes. A related point is the chronic desire to write `new T()` where *T* is a type parameter to a generic. See

https://code.google.com/p/dart/issues/detail?id=3633


The current state of affairs leads to an ugly wart, where parentheses can change the meaning of an expression. Compare:

`C.staticMethod()` as opposed to `(C).staticMethod()`. The former is a static method call, while the latter is an instance method call an instance of `Type`. Consequently, it fails. This sort of situation is confusing to users; understanding the difference is subtle. 

###Pros

* Simple

* Eliminates some scenarios where users might otherwise need to use reflection.

* Provides a more elegant solution than constructor tear-offs. 

* No syntax changes are needed.

* Addresses perceived needs, as evidenced by submitted bugs.




 
###Cons

* This does double the number of class objects in runtime as each declared class introduces its own metaclass. Perhaps these can be generated lazily.

* People may choose to use `C.new` instead of `new C()`, even when the latter is legal. In other words, there would be two ways of doing the same thing, which is regrettable.
Also, some may object on grounds of style. This could be mitigated by defining *new e(args)* as *e.new(args)*. I would prefer not to go that route, but it may have some merit in that the obvious, naive thing just works.

* Does not allow the use const constructors to create constant objects.

* Does not give an easy way to refer to parameterized types as expressions, e.g., `Type t = List<T>;`. That is a separate issue.

* No way to specify that a metaclass implements an interface.

* May have negative impact on tree-shaking, leading to increased size of applications deployed via Javascript. Initial studies indicate this effect is minor.


## Examples

See the bugs.
```
class A {
}

class B<T> {
  T GetBySmth() {
    return T.new(); // could even be new T() if we went that far
  }
}

void main() {
  var b = new B<A>();
  var x = b.GetBySmth();
  assert(x is B);
}
```

Now, in the case where we want to be type safe we could write

```
class B<T extends A class extends A.class> {
  T GetBySmth() {
    return T.new(); // could even be new T() if we went that far
  }
}

```

However, this is still less useful than it might seem, since even if *C* is a subtype of *A*, *C.class* is not a subtype of *A.class*. It should not be, since neither statics nor constructors are inherited. So to be more generally useful, one would need to define the desired interface that we need to use, say 

```
abstract class HasNullaryAnonymousConstructor {
  static HasNullaryAnonymousConstructor();
}
```

and then we'd need to somehow state that *A.class* implements *HasAnonymousConstructor*. Our options are to add some sort of implements clause for the class, such as

```
class A class implements HasAnonymousConstructor {
}

```

or we could go with a structural approach instead. We could use this in

```
class B<T extends A class extends HasNullaryAnonymousConstructor> {
  T GetBySmth() {
    return T.new(); // could even be new T() if we went that far
  }
}

```




## Proposal

The basic proposal is described above. 

## Deliverables


### Language specification changes

Commentary on the specification is given in *italics*. Rationale is in **bold**. Deleted sections are ~~struck out~~.

The specification already states that `Type` objects have members corresponding to the static members of a type.  However, access to them is disallowed. The necessary spec changes are mainly about removing these restrictions, and introducing the concept of a metaclass type. In addition, the spec neglected to incorporate members that correspond to constructors, so the following text must be added to section 10.6, **Constructors**:

The declaration of a constructor named *m* in class *C* has the effect of adding an instance method with the same name and signature to the `Type` object for class *C* that, when run, returns the result of evaluating `new C.m` with the same arguments the instance method was called with. 

*Because each generic instantiation has its own metaclass, type arguments are already included in `C`.*

The declaration of an anonymous constructorin class *C* has the effect of adding an instance method with the same name and signature to the `Type` object for class *C* that, when run, returns the result of evaluating `new C` with the same arguments the instance method was called with.


I propose to add a dedicated subsection, 10.11, to the section on classes.

#### 10.11 Metaclasses

At runtime, every class is associated with its own metaclass. 

** Each class *C* has a metaclass, because the `Type` object representing *C* has specific behavior corresponding to its constructors and static getters (10.2), setters (10.3), and methods (10.7).**

*Here we speak of classes as runtime objects, distinct from class declarations. The distinction matters with respect to generics, where a single declaration introduces an entire family of runtime classes.  It is helpful to recall that every object has a particular class at runtime; this class might in fact be an instantiation of a generic class declaration, e.g., `List<int>;` in any event, each such class has its own unique metaclass.*

Let *C* be a class:

The metaclass of *C* has a single instance, the unique instance of `Type` associated with *C*. The type of *C* may be referred to as *C.class*, where *C* is a type literal.

All metaclasses are direct subclasses of class `Type`. 

*In particular, the metaclass of class `Type`, `Type.class`, is a direct subclass of `Type`.*

It is a compile time error to extend a metaclass. All metaclasses are instances of class `Metaclass`.  

*`Metaclass` itself is a class, and as such it is an instance of `Metaclass.class` which is in turn an instance of `Metaclass`.*

**This circularity resolves what would otherwise be an infinite regress.**


In addition, the restrictions disallowing access to such members must be removed from sections 16.17.1, 16.18, 16.19

#### 16.17.1 Ordinary Invocation

An ordinary method invocation *i* has the form *o.m(a1,...,an,xn+1 : an+1,...,xn+k : an+k*.

Evaluation of an ordinary method invocation *i* of the form *o.m(a1,...,an,xn+1 : an+1,...,xn+k : an+k)*
proceeds as follows:

First, the expression *o* is evaluated to a value *vo*. Next, the argument list *(a1, . . . , an, xn+1 : an+1, . . . , xn+k : an+k)* is evaluated yielding actual argument objects *o1,...,on+k*. Let *f* be the result of looking up (16.15.1) method *m* in *vo* with respect to the current library *L*.

Let *p1 . . . ph* be the required parameters of *f*, let *p1 . . . pm* be the positional parameters of *f* and let *ph+1, . . . , ph+l* be the optional parameters declared by *f*.

*We have an argument list consisting of *n* positional arguments and *k* named arguments. We have a function with *h* required parameters and *l* optional parameters. The number of positional arguments must be at least as large as the number of required parameters, and no larger than the number of positional parameters. All named arguments must have a corresponding named parameter.*

If *n < h*, or *n > m*, the method lookup has failed. Furthermore, each *xi, n + 1 ≤ i ≤ n + k*, must have a corresponding named parameter in the set *{pm+1,...,ph+l}* or the method lookup also fails.~~ If *vo* is an instance of `Type` but *o* is not a constant type literal, then if *m* is a method that forwards (9.1) to a static method, method lookup fails.~~ Otherwise method lookup has succeeded.

If the method lookup succeeded, the body of *f* is executed with respect to the bindings that resulted from the evaluation of the argument list, and with this bound to *vo*. The value of *i* is the value returned after *f* is executed.

If the method lookup has failed, then let *g* be the result of looking up getter (16.15.2) *m* in *vo* with respect to *L*.~~ If *vo* is an instance of `Type` but *o* is not a constant type literal, then if *g* is a getter that forwards to a static getter, getter lookup fails.~~ If the getter lookup succeeded, let *vg* be the value of the getter invocation *o.m*. Then the value of *i* is the result of invoking the static method `Function.apply()` with arguments *v.g, [o1, . . . , on], {xn+1 : on+1, . . . , xn+k : on+k}*.

If getter lookup has also failed, then a new instance *im* of the predefined class `Invocation` is created, such that :

* *im.isMethod* evaluates to `true`.

* *im.memberName* evaluates to `’m’`.

* *im.positionalArguments* evaluates to an immutable list with the same values as *[o1,...,on]*.

* *im.namedArguments* evaluates to an immutable map with the same keys and values as *{xn+1 : on+1, . . . , xn+k : on+k}*.

Then the method `noSuchMethod()` is looked up in *vo* and invoked with argument *im*, and the result of this invocation is the result of evaluating *i*. However, if the implementation found cannot be invoked with a single positional argument, the implementation of `noSuchMethod()` in class `Object` is invoked on *vo* with argument *im′*, where *im′* is an instance of `Invocation` such that :

* *im.isMethod* evaluates to `true`.

* *im.memberName* evaluates to `noSuchMethod`.

* *im.positionalArguments* evaluates to an immutable list whose sole element is *im*.

* *im.namedArguments* evaluates to  to the value of **const** `{}`.

and the result of the latter invocation is the result of evaluating *i*.

*It is possible to bring about such a situation by overriding `noSuchMethod()` with the wrong number of arguments:*

```
class Perverse { noSuchMethod(x,y) => x + y; }
new Perverse.unknownMethod();
```

*Notice that the wording carefully avoids re-evaluating the receiver *o* and the
arguments *ai*.*

Let *T* be the static type of *o*. It is a static type warning if *T* does not have
an accessible (6.2) instance member named *m* unless ~~either:~~ *T* or a superinterface of *T* is annotated with an annotation denoting a
constant identical to the constant `@proxy` defined in `dart:core`. ~~Or
*T* is `Type`, *e* is a constant type literal and the class corresponding to *e* has
a static getter named *m*.~~

If *T.m* exists, it is a static type warning if the type *F* of *T.m* may not be assigned to a function type. If *T.m* does not exist, or if *F* is not a function type, the static type of *i* is dynamic; otherwise the static type of *i* is the declared return type of *F*.
It is a compile-time error to invoke any of the methods of class `Object` on a prefix object (18.1)  ~~or on a constant type literal that is immediately followed by the token ‘.’~~.



#### 16.18 Property Extraction

*Property extraction* allows for a member of an object to be concisely extracted from the object. A property extraction can be either:

1. A *closurization* (16.18.1) which allows a method to be treated as if it were a getter for a function valued object. Or
2. A *getter invocation* which returns the result of invoking of a getter method.

Evaluation of a property extraction *i* of the form *e.m* proceeds as follows:

First, the expression *e* is evaluated to an object *o*. Let *f* be the result of looking up (16.15.1) method (10.1) *m* in *o* with respect to the current library *L*. ~~If *o* is an instance of `Type` but *e* is not a constant type literal, then if *m* is a method that forwards (9.1) to a static method, method lookup fails.~~ If method lookup succeeds and *f* is a concrete method then *i* evaluates to the closurization of *o.m*.

Otherwise, *i* is a getter invocation, and the getter function (10.2) *m* is looked up (16.15.2) in *o* with respect to*L*. ~~If *o* is an instance of `Type` but *e* is not a constant type literal, then if *m* is a getter that forwards to a static getter, getter lookup fails~~. Otherwise, the body of *m* is executed with this bound to *o*. The value of *i* is the result returned by the call to the getter function.

If the getter lookup has failed, then a new instance *im* of the predefined class `Invocation` is created, such that :

* *im.isGetter* evaluates to `true`.

* *im.memberName* evaluates to `’m’`.

* *im.positionalArguments* evaluates to **const** `[]`.

* *im.namedArguments* evaluates to **const** `{}`.

Then the method `noSuchMethod()` is looked up in *o* and invoked with argument *im*, and the result of this invocation is the result of evaluating *i*. However, if the implementation found cannot be invoked with a single positional argument, the implementation of `noSuchMethod()` in class `Object` is invoked on *o* with argument *im′*, where *im′* is an instance of `Invocation` such that :

* *im.isMethod* evaluates to `true`.

* *im.memberName* evaluates to `noSuchMethod`.

* *im.positionalArguments* evaluates to an immutable list whose sole element is *im*.

* *im.namedArguments* evaluates to  to the value of **const** `{}`.

and the result of the latter invocation is the result of evaluating *i*.

It is a compile-time error if *m* is a member of class `Object` and *e* is ~~either~~ a
prefix object (18.1)~~ or a constant type literal~~.

~~*This precludes `int.toString` but not `(int).toString` because in the latter case, *e* is a parenthesized expression.*~~

Let *T* be the static type of *e*. It is a static type warning if *T* does not have a method or getter named *m* unless ~~either:~~ *T* or a superinterface of *T* is annotated with an annotation denoting a constant identical to the constant `@proxy` defined in `dart:core`.~~ Or
* *T* is `Type`, *e* is a constant type literal and the class corresponding to *e* has a static method or getter named *m*.~~

If *i* is a getter invocation, the static type of *i* is:

* The declared return type of *T.m*, if *T.m* exists.

~~* The declared return type of *m*, if *T* is `Type`, *e* is a constant type literal and the class corresponding to *e* has a static method or getter named *m*.~~

* The type **dynamic** otherwise.

If *i* is a closurization, its static type is as described in section 16.18.1.


**Rest of Section Unchanged **

#### 16.19 Assignment

The section that changes is:

Evaluation of an assignment of the form *e1.v = e2* proceeds as follows:

The expression *e1* is evaluated to an object *o1*. Then, the expression *e2* is evaluated to an object *o2*. Then, the setter *v =* is looked up (16.15.2) in *o1* with
respect to the current library.~~If *o1* is an instance of `Type` but *e1* is not a constant type literal, then if `v =` is a setter that forwards (9.1) to a static setter, setter lookup fails. Otherwise, t~~The body of *v =* is executed with its formal parameter bound to *o2* and **this** bound to *o1*.

If the setter lookup has failed, then a new instance *im* of the predefined class `Invocation` is created, such that :

* *im.isMethod* evaluates to `true`.

* *im.memberName* evaluates to `v=`.

* *im.positionalArguments* evaluates to an immutable list with the same values as *[o2]*.

* *im.namedArguments* evaluates to  to the value of **const** `{}`.

Then the method `noSuchMethod()` is looked up in *o1* and invoked with argument *im*.
However, if the implementation found cannot be invoked with a single positional argument, the implementation of `noSuchMethod()` in class `Object` is invoked on *o1* with argument *im′*, where *im′* is an instance of `Invocation` such that :

* *im.isMethod* evaluates to `true`.

* *im.memberName* evaluates to `noSuchMethod`.

* *im.positionalArguments* evaluates to an immutable list whose sole element is *im*.

* *im.namedArguments* evaluates to  to the value of **const** `{}`.

The value of the assignment expression is *o2* irrespective of whether setter lookup has failed or succeeded.

In checked mode, it is a dynamic type error if *o2* is not **null** and the interface of the class of *o2* is not a subtype of the actual type of *e1.v*.

Let *T* be the static type of *e1*. It is a static type warning if *T* does not have an accessible instance setter named *v =* unless ~~either:~~ *T* or a superinterface of *T* is annotated with an annotation denoting a constant identical to the constant `@proxy` defined in `dart:core`.~~Or
* *T* is `Type`, *e1* is a constant type literal and the class corresponding to *e1* has a static setter named *v =*.~~

It is a static type warning if the static type of *e2* may not be assigned to the static type of the formal parameter of the setter *v =*. The static type of the expression *e1.v = e2* is the static type of *e2*.

Evaluation of an assignment of the form *e1[e2] = e3* is equivalent to the evaluation of the expression `(a, i, e){a.[]=(i, e); return e; } (e1 , e2 , e3 )`. The static type of the expression *e1[e2] = e3* is the static type of *e3*.

It is a static warning if an assignment of the form *v = e* occurs inside a top level or static function (be it function, method, getter, or setter) or variable initializer and there is neither a local variable declaration with name *v* nor setter declaration with name *v =* in the lexical scope enclosing the assignment.

It is a compile-time error to invoke any of the setters of class `Object` on a prefix object (18.1)~~or on a constant type literal that is immediately followed by the token ‘.’~~.


In addition, we must update section 16.32 which specifies the static type of an identifier so that type literals are assigned the type of their metaclass.

#### 16.32 Identifier Reference

One word change here:

* If *d* is a class, type alias or type parameter *T* the static type of *e* is ~~`Type`~~ *T.class*.

### A working implementation

TBD

### Tests

TBD

## Patents rights

TC52, the Ecma technical committee working on evolving the open [Dart standard][], operates under a royalty-free patent policy, [RFPP][] (PDF). This means if the proposal graduates to being sent to TC52, you will have to sign the Ecma TC52 [external contributer form]() and submit it to Ecma.

[tex]: http://www.latex-project.org/
[language spec]: https://www.dartlang.org/docs/spec/
[dart standard]: http://www.ecma-international.org/publications/standards/Ecma-408.htm
[rfpp]: http://www.ecma-international.org/memento/TC52%20policy/Ecma%20Experimental%20TC52%20Royalty-Free%20Patent%20Policy.pdf
[external contributer form]: http://www.ecma-international.org/memento/TC52%20policy/Contribution%20form%20to%20TC52%20Royalty%20Free%20Task%20Group%20as%20a%20non-member.pdf