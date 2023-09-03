---
layout: page
title: Design Patterns
---
## Pattern purpose
Behavioral
 : Deal with composition of classes or objects
* [Chain of responsibility](#chain-of-responsibility)
* [Command](#command)
* [Interpreter](#interpreter)
* [Iterator](#iterator)
* [Mediator](#mediator)
* [Memento](#memento)
* [Observer](#observer)
* [State](#state)
* [Strategy](#strategy)
* [Template method](#template-method)
* [Visitor](#visitor)

Structural
 : Deal with composition of classes or objects
* [Adapter](#adapter)
* [Bridge](#bridge)
* [Composite](#composite)
* [Decorator](#decorator)
* [Facade](#facade)
* [Flyweight](#flyweight)
* [Proxy](#proxy)

Creational
 : Creation of object
* [Abstract factory](#abstract-factory)
* [Builder](#builder)
* [Factory method](#factory-method)
* [Prototype](#prototype)
* [Singleton](#singleton)

## Behavioral patterns
These patterns are concerned with algorithms and the assigment of responsibilities between objects.
* Behavioral class patterns use inheritance to distribute behavior between classes
* Behavioral object patterns use object composition rather inheritance 

### Chain of responsibility
> Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

![Chain of responsiblity]({{site.baseurl}}/assets/img/chain_of_responsibility.jpg)
* Usage: When more than 1 object may handle a request and handler is not known.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/chain_of_responsibility/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/chain_of_responsibility/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/chain_of_responsibility/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/chain_of_responsibility/notebook.ipynb)

### Command
> Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests and support undoable operations.

![Command]({{site.baseurl}}/assets/img/command.jpg)
* Usage: To parameterize objects by an action to perform or support undo.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/command/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/command/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/command/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/command/notebook.ipynb)

### Interpreter
> Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

* Usage when there is a language to interpret and you can represent statements in the language as abstract trees. It works best when the grammar is simple and efficiency is not a critical concern.

![Interpreter]({{site.baseurl}}/assets/img/interpreter.jpg)
* Usage: To parameterize objects by an action to perform or support undo.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/interpreter/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/interpreter/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/interpreter/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/interpreter/notebook.ipynb)

### Iterator
> Provide a way to access the elements of an aggregate object sequentially without exposing tits underlying representation.

![Iterator]({{site.baseurl}}/assets/img/iterator.jpg)
* Usage when
    * to access an aggregate object's contents without exposing its internal representation
    * to support multiple traversals of aggregate objects
    * to provide a uniform interface for traversing different aggregate structures
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/iterator/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/iterator/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/iterator/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/iterator/notebook.ipynb)

### Mediator
> Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly and it lets you vary their interactions indepentently.

![Mediator]({{site.baseurl}}/assets/img/mediator.jpg)
* Usage: When have set of objects that communicate but interdependencies increase complexity, or want to customize behavior that is distributed across several classes without lots of subclassing.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/mediator/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/mediator/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/mediator/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/mediator/notebook.ipynb)

### Memento
> Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.

![Memento]({{site.baseurl}}/assets/img/memento.jpg)
* Usage: When need to implement undo operation.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/memento/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/memento/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/memento/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/memento/notebook.ipynb)

### Observer
> Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

![Observer]({{site.baseurl}}/assets/img/observer.jpg)
* Usage: When you don't want objects tightly coupled, when a change to oone object requires changing others.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/observer/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/observer/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/observer/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/observer/notebook.ipynb)

### State
> Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

![State]({{site.baseurl}}/assets/img/state.jpg)
* Usage: An object's behavior depends on its state and must change its behavior at run-time depending on that state, or when operations have large, multipart conditional statements that depend on the object's state.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/state/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/state/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/state/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/state/notebook.ipynb)

### Strategy
> Define a family of algorithms, encapsulate each one and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

![Strategy]({{site.baseurl}}/assets/img/strategy.jpg)
* Usage: When need different variants of an algorithm or have many related classes differ only in their behavior allowing to configure a class with one of many behaviors.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/strategy/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/strategy/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/strategy/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/strategy/notebook.ipynb)

### Template method
> Define a skeleton of an algorithm in an operation, deferring some steps to subclasses. Template method lets subclasses redefine certain steps of an algorithm without changing the algorithm structure.

![Template method]({{site.baseurl}}/assets/img/template_method.jpg)
* Usage: Implement invariant parts of an algorithm once in abstract class and leave to subclasses to implement varying behavior.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/template_method/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/template_method/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/template_method/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/template_method/notebook.ipynb)

### Visitor
> Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.

![Visitor]({{site.baseurl}}/assets/img/visitor.jpg)
* Usage when
    * an object structure contains many classes of objects with differing interfaces and you want to perform operations on those objects that depend on the concrete class
    * many distinct and unrelated operations need to be performed on objects in an object structure and you want to avoid polluting their classes with these operations.
    * the classes defining the object structure rarely change but you often want to define new operations over the structure.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/visitor/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/visitor/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/visitor/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/visitor/notebook.ipynb)

## Structural patterns
Structural patterns are concerned with how classes and objects are composed to form larger structures.
* Structural class patterns use inheritance to compose interfaces or implementations
* Structural object patterns describe ways to compose objects to realize new functionality

### Adapter
> Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

Two flavors of adapters
* Class adapter uses multiple inheritance to adapte one interface to another
* Object adapter relies on object composition

![Adapter]({{site.baseurl}}/assets/img/adapter.jpg)
* Usage: When you want to use an existing class and its interface does not match the one you need.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/adapter/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/adapter/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/adapter/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/adapter/notebook.ipynb)

### Bridge
> Decouple an abstraction from its implementation so that the two can vary indepentently.

### Composite
> Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of object uniformly.

### Decorator
> Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

![Decorator]({{site.baseurl}}/assets/img/decorator.jpg)
* Usage: When want to add responsibilities to individual objects dynamically or when extension by subclassing is impractical.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/decorator/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/decorator/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/decorator/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/decorator/notebook.ipynb)

### Facade
> Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

![Facade]({{site.baseurl}}/assets/img/facade.jpg)
* Usage: When you want to provide a simple interface to a complex subsystem. Another use-case is when you have many dependencies between clients and implementation classes of an abstraction, use Facade to decouple the subsystem from clients and other subsystems. Usually one Facade object is required, Facades are often Singletons.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/facade/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/facade/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/facade/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/facade/notebook.ipynb)

### Flyweight
> Use sharing to support large numbers of fine-grained objects efficiently.

### Proxy
> Provide a surrogate or placeholder for another object to control access to it.

## Creational patterns

### Abstract Factory
> Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

![Abstract factory]({{site.baseurl}}/assets/img/abstract_factory.jpg)
* Usage when
    * system should be independent of how its products are created, composed and represented
    * a system should be configured with one of multiple families of products
    * a family of related product objects is designed to be used together and you need to enforce this constraint
    * you want to provide a class library of products and you want to reveal their interfaces not their implementations
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/abstract_factory/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/abstract_factory/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/abstract_factory/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/abstract_factory/notebook.ipynb)

### Builder
> Separate the construction of a complex object from its representation so that the same construction process can create different representations.

### Factory method
> Define an interface for creating an object, but let subclasses to decide which class to instantiate. Factory method lets a class defer instantiation to subclasses.

![Factory method]({{site.baseurl}}/assets/img/factory.jpg)
* Usage when
    * a class cannot anticipate the class of objects it must create
    * a class wants its subclasses to specify the objects it creates
    * classes delegate responsibility to one of several helper subclasses and you want to localize the knowledge of which helper subclass is the delegate.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/factory/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/factory/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/factory/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/factory/notebook.ipynb)

### Prototype
> Specify the kinds of objects to create using a prototypical instance and create new objects by copying its prototype.

### Singleton
> Ensure a class only has one instance and provide a global point of access to it.

![Singleton]({{site.baseurl}}/assets/img/singleton.jpg)
* Usage: When need to create one instance of a class and use it throughout the lifecycle of a program. Another usage is if want to limit concurrent access to a shared resource.
* Implementation: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/singleton/pattern.py](https://github.com/gdyrrahitis/design-patterns-python/blob/main/singleton/pattern.py)
* Execution: [https://github.com/gdyrrahitis/design-patterns-python/blob/main/singleton/notebook.ipynb](https://github.com/gdyrrahitis/design-patterns-python/blob/main/singleton/notebook.ipynb)
