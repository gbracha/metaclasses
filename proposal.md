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

* This does double the number of class objects in runtime as each declared class introduces its own metaclass. 

* People may choose to use `C.new` instead of `new C()`, even when the latter is legal. In other words, there would be two ways of doing the same thing, which is regrettable.
Also, some may object on grounds of style. This could be mitigated by defining *new e(args)* as *e.new(args)*. I would prefer not to go that route, but it may have some merit in that the obvious, naive thing just works.

* Does not allow the use const constructors to create constant objects.

* Does not give an easy way to refer to parameterized types as expressions, e.g., `Type t = List<T>;`. That is a separate issue.


## Examples

See the bugs.


## Proposal

See above.

## Deliverables


### Language specification changes

Commentary on the specification is given in *italics*. Rationale is in **bold**.


I propose to add a dedicated subsection, 10.11, to the section on classes.

#### 10.11 Metaclasses

At runtime, every class is associated with its own metaclass. 

*Here we speak of classes as runtime objects, distinct from class declarations. The distinction matters with respect to generics, where a single declaration introduces an entire family of runtime classes.  It is helpful to recall that every object has a particular class at runtime; this class might in fact be an instantiation of a generic class declaration, e.g., `List<int>;` in any event, each such class has its own unique metaclass.*

All metaclasses are direct subclasses of class `Type`.  

*In particular, the metaclass of class `Type` is a direct subclass of `Type`. A `Type` is an instance of one of its subclasses.*

Each metaclass has a single instance. Let *C* be a class:

For each static method named *m* with signature *s* declared by *C*,  the metaclass of *C* declares an instance method named *m* with signature *s*, that, when run, returns the result of calling the static method *C.m* with the same arguments the instance method was called with. 

For each constructor named *m* with signature *s* declared by *C*,  the metaclass of *C* declares an instance method named *m* with signature *s*, that, when run, returns the result of evaluating `new C.m` with the same arguments the instance method was called with.

*Because each generic instantiation has its own metaclass, type arguments are already included in `C`.*

If class *C* declares an anonymous constructor with signature *s*, the metaclass of *C* declares an instance method named `new`  with signature *s*, that, when run, returns the result of evaluating `new C` with the same arguments the instance method was called with.





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
[form]: http://www.ecma-international.org/memento/TC52%20policy/Contribution%20form%20to%20TC52%20Royalty%20Free%20Task%20Group%20as%20a%20non-member.pdf