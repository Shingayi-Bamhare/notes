# Ch1: How CLOS is Implemented

- **What is Closette and what's its point?**

Closette is a CLOS interpreter, simplified for pedagogical purposes. It is presented to give a glimpse of the parts of implemetation for CLOS.

- **What will Closette contain?**

Closette will contain _classes_, _instances_, _generic functions_, and _methods_. There's a list of major restrictions on p14.

- **What are the fundamental CLOS forms?**

`defclass`: Define slots, accessors, and method of instantiation.
`defgeneric`: Define a generic function for a class, along with an interface for calling that function.
`defmethod`: An implementation for a generic function. Supports simple pattern matching on argument types.
`make-instance`: Returns an object based on a class. The arguments received by `make-isntance` are dependent on the class and are defined in a `defclass` form.

- **How are metaobjects involved in classes?**

CLOS uses _class metaobjects_ to define the structure that represents classes created using the `defclass` form. For instance, things like the name of the class, the direct superclasses, the slots, the precedence list, methods defined on the class, etc. are stored in the class metaobject.

The class `standard-class` provides the default for what class metaobjects look like, which means that class metaobjects are really instances of `standard-class`.

- **What is `defclass` doing behind the scenes?**

`defclass` is a macro that expands to a call to `ensure-class`, which accepts a symbol for the class name, a list of `:direct-superclasses`, and a list of `:direct-slots`. Each direct slot itself has: `:name`, `:initform`, `:initfunction`, `:initargs`, `:readers`, and `:writers`. The `:accessor` is turned into a reader and writer.

`ensure-class` then uses all of this information to create an instance of the class metaobject `standard-class`, which is registered to the global database. All of the keyword arguments are simply passed to the `standard-class`, which means that `ensure-class` doesn't actually care about them; all it really does is class naming. (This is performed using a settable `find-class` fn.)

- **How does `make-instance` operate on class metaobjects?**

`make-instance` must do extra work when creaing class metaobjects, like providing defaults, adding subclass links to superclasses, creaing direct slot definitions, defining accessor methods, and performing inheritance. This is done with an `:after` method on `initialize-instance` defined for `standard-class`.

A very readable implementation of this after-method can be found on p23. Basically, add this class to the superclasses list of direct subclasses, and add those superclasses to this class' list of superclasses. Then, generate direct slot definitions for each slot, and set this classes direct slots to be those new definitions. For each reader, create a reader method, and do the same for writers. Finally, create the inheritance chain.

- **How is inheritance precedence determined?**

All of the superclasses are ordered using local topological precedence, with special rules to resolve ties. This always leads to `standard-object`, followed by `t`, to be the last things from which a class inherits.

- **How are objects printed in Closette?**

All object printing is controlled by the generic function `print-object`, and a method is defined for all instances of `standard-object`.

- **What are important considerations for a model of instances?**

Instances should have some kind of distinct identity, some place for the bindings of slots, some way of retaining knowledge of their class, and the ability to change that class while retaining the same identity.

- **How do instances know about and access their slots?**

When an object is instantiated, space is allocated for each slot, and each slot is bound to some value. Then a number of slot accessor functions are created, such that a slot name and an instance could return the actual location for that slot. When a slot is accessed, a `boundp` predicate determines whether or not the value at that slot is the default; if it is, an error is thrown. Thus, `slot-value` can be defined, as well as `(setf slot-value)`.

During initialization, an instance and its slots will first be allocated, and then those slots will be filled based on the following precedence:

1. Explicit initialization argument
2. Existing binding of the slot
3. Default `:initform` value if the slot has one
4. Do nothing

- **What does it mean to change an instance's class?**

In effect, the instance is used to initialize a new instance of the new class, while still retaining its identity and shared slot values. Slots that aren't common to both classes, though, will be dropped.

- **What kind of object underlies generic functions?**

A generic function is also represented by a metaobject, the _generic function metaobject_! This metaobject stores all of the methods defined on the generic function as well as the parameter signature for the implementation. Methods defined using `defmethod` also have a metaobject associated with them, which contains information relevant to that particular method, including a reference to its generic function's metaobject.

A _method metaobject_ also needs to know all of the definitions in lexical scope, so that they may be used during a method invocation. This is apparently difficult to do in Common Lisp, and so Closette chooses to only retain the more simple top-level environment.

- **How does Closette dispatch on generic function invocation?**

At a high level, Closette dispatches on generic fn calls using a process called _method lookup_, which has three parts: 1) determine which methods are applicable based on the given arguments, 2) sort those methods in order of decreasing precedence, and 3) sequence the execution of the sorted list of methods based on the qualifiers (`:before`, `:after`, etc).

In order to accomplish this, calls to generic functions are immediately dispatched to `apply-generic-function`, such that:

```cl
(generic-function arg1 arg2)
;; becomes
(apply-generic-function #'generic-function (list arg1 arg2))
```

Then:

1. Find applicable methods

This is done by comparing the arguments to the generic fn with the specializers defined on the various methods. If the classes of the arguments match the specializers, then the method is applicable. (The arguments could also be instances of subclasses of the specializers.)

2. Sort by decreasing precedence

Method precedence is determined by class precedence, where the specializer that is deemed more "specific" (a subclass, for instance) will have higher precedence than a less specific specializer (a superclass). This is slightly more complicated in the case of multiple inheritance, but not incredibly so. You can just use the argument's class precedence list.

3. Sequence the applicable methods

All of the sorted, applicable methods are then grouped into `:before`, primary, and `:after` methods. All of the `:before` methods are executed, from most specific to least, then the most specific primary method, then all of the `:after` methods, similar to `:before`. The values returned from the call to the generic fn are the values returned from the primary method.

Within the most specific primary method, `call-next-method` can be called to call the next most specific primary method.
