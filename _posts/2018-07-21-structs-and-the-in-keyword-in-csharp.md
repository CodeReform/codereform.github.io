---
layout: post
image: https://miro.medium.com/1*MfOHvI5b1XZKYTXIAKY7PQ.png
title: Structs and the in keyword in C#
date: 2018-07-21 18:45:30.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ".NET Core"
- C#
tags:
- data structures
- C#
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2018/07/21/structs-and-the-in-keyword-in-csharp/"
---
# Structs and the in keyword in C#

Since the release of 7.0 version for C#, we've seen many small, yet useful features added to the language, aiming to aid developers, directly or indirectly. A new keyword was added in version 7.2, the in keyword, which makes the reference semantics of the language richer. In this post, I will explore this new keyword, along with readonly structs and the ref readonly modifier.

## Source code

Code outlined in this article can be found on my GitHub [repository](https://github.com/gdyrrahitis/bpost-csharp-in-modifier).

## How to enable C# 7.x features

To enable C# 7.x features in your project, you can do this via the project properties. In Visual Studio, right click on the project (or select it and press Alt+Enter), select the Build tab and from there the Advanced button, as shown in the picture below.

![](https://miro.medium.com/0*_x5R5I1EvVrlSFLy)

In the General section, you can select the version via the _Language version_ dropdown. It will list all the version languages since version 3, but also it has some other options as well.

- _Latest major version (default)_

- _Latest minor version (latest)_

The major version option, which is the default, selects the major version of the language. For example, at the time of this writing, the C# version is on **7****.3**. The major version is **7** and minor is **.3**. So this option selects version **7**.0 by default, but not **7**.1 or **7**.2 neither **7****.3**. Only the latest major.
The minor version option though let's you opt-in to the latest version, so if C# latest version is **7****.3**, it's picking up that. When **7**.4 becomes available, your project will automatically pick that up (assuming you updated your Visual Studio).

For this post, you can either choose explicitly the version (7.3) or choose the _Latest minor version_ option.

Alternatively, you can set the version in the .csproj file by setting the LangVersion property.

{% gist 4bab483720a2ffd065ed0d3a4924ce8e %}

Or the explicit version by typing the version number

{% gist 8322d38b1ecfb807ed5531c41092160f %}

## Value types

In programming, it is common to declare variables to store some kind of state and work with it along the way. These variables provide useful information to our programs, inputs, outputs, that kind of goodies. A variable is just an association between a name that is used in the program and a slot in memory. Now, what is stored and where in memory for that value, depends on the type of the variable. Thus, we distinct values in two categories, reference types and value types.

A value that is reference type can be either a reference in memory, compatible to the object it was declared, or null, which means it does not point to anything at all. These type of values are stored in the heap. A special mechanism, called Garbage Collector, at least in most modern languages, is responsible for maintaining a balance of the references that live in the heap, by organizing them and freeing memory when certain conditions are met. Reference types can be classes, delegates, etc.

A value of a value type will be always the instance of the type, with such types being primitive types like int, double, float, etc or structs. These are not allowed to have a null value and are stored in the stack. When the current execution context goes out of scope, the value is removed from the stack.

> Local variables_:_

> _A local variable is stored on the stack, the same applies for reference type variables, though in this case the variable itself lives on the stack, the value of the reference type variable is only a reference, or null, and not the actual object. In the same manner, method parameters count as local variables too, but if they are declared with the ref or out modifier, they don't get their own slot, but share a slot with the variable used in the calling code._

The great advantage of value types is that they are allocated to the stack which is a very cheap operation to perform, for the runtime, while the heap is more cumbersome. The downside though, is that because the value types are not allocated in heap, they are passed in methods by value, compared to reference types, which pass the actual reference to the same object that was used in the context of the calling code. Copying a value can also be costly as the copy has the same size as the original value passed.

## The in keyword

This is a new keyword introduced in version 7.2, an addition to the rest modifiers, ref and out. You can use this in methods, in the same manner as ref and out, as the in modifier specifies that the value is passed by reference, but also the method is not allowed to modify it. This is different from ref and out modifiers, as both allow the value to be modified. Using this modifier, programmers communicate code's intent better, by saying explicitly that this method will receive a value by reference, but it will not change it.

- The ref modifier allows a value type to be passed as reference and the calling method is allowed to change it and it _might_ change within that method.

- The out modifier allows a value type to be passed as reference and the calling method is _required_, by the compiler, to assign a value to the value's argument within that method.

- The in modifier allows a value type to be passed as reference and the compiler does not allow any modifications to that value within that method.

It is very useful to pass around struct value types and ensure that the method called will not mutate them. But what is the big beef here? The reasoning is improvement in performance.
Imagine that you have created a struct of 28 bytes and you pass it to a method by value (without any modifier). This method will receive a copy of that struct, requiring additional data of 28 bytes. This might lead to performance overhead due to excessive copying.

Let's see it in action

{% gist 12352dfef9e5a878bdf68874be4b31a8 %}

When I read about the benefits, I really wanted to stress-test this scenario and see with my own eyes how much I gain in performance. Please find [benchmark source code](https://github.com/gdyrrahitis/bpost-csharp-in-modifier/tree/master/CSharp.7_3.Features.Benchmark/Scenarios) in GitHub repository.
The results of the benchmarks are below, I am calling a method 1.000, 10.000 and 100.000 times by passing really big struct. Let's put it into test.

{% gist 28c33b16e1c435bb3ecd115e40c7f116 %}

These results are weird. I expected the in modifier benchmark to be much faster, but it is slower than the original! What happened?
The answer lies in the IL de-compiled code. I am using [ILSpy](https://marketplace.visualstudio.com/items?itemName=SharpDevelopTeam.ILSpy), a free tool which de-compiles your .dll back to C#. I really want to see how these features are implemented under the hood. Let's see the code shown previously, what the compiler sees?

{% gist 8ca478ebd203938e3184965690bf7b05 %}

Hmm, the method with the in modifier, is a bit weird, it has created a copy of the struct. But wait a minute, wasn't this modifier supposed to resolve this? Well, actually no, this is by design and the compiler is the culprit here, but for a good reason.

## Defensive copies

Enter the defensive copies. This is how you can call what happened in the code above. When we are declaring an in value type, we are essentially passing a readonly reference. The compiler in order to protect us from any unwanted mutations, is deliberately creating a defensive copy of the value passed in.

Let's look at the readonly struct next and then I am going to retry the benchmarks to see if I gained anything.

## The readonly struct

As the name states, this allows you to create an immutable struct, by adding the readonly modifier before the struct keyword when declaring it. By doing this, the compiler forces all the struct members to be readonly, rendering the struct immutable.

{% gist 54b1bd71c12eea2daf1a4c9af1462dfc %}

Notice in code above, the properties are readonly. If I tried to add a setter in any of the properties, I would have received a compile time error stating: "_Auto-implemented instance properties in readonly structs must be readonly_". This is semantically correct, I cannot have an immutable struct with mutable public members. ILSpy emits the following.

{% gist 02a25f725e7f25a9a97c2909a5182581 %}

## Immutability

An immutable object is an object whose state cannot be modified once it is created, in contrast to mutable objects, which allow altering their state after their creation. Immutability is one of the core principles of functional programming and it is enforced in most functional languages, like [Haskell](https://www.haskell.org/). In [F#](https://fsharp.org/) language, you are allowed to mutate the state of an object, if you want, that is because of interoperability with the .NET framework, which has many imperative data structures. Normally, it is not the way to write functional programs in [F#](https://fsharp.org/).

Immutable objects have a lot of advantages over mutable, and it is normally best practice to make your objects immutable, whenever possible. It will make your code very much easier to reason about, more predictable, with clean intent. Immutable objects really eliminate side effects, as you are not allowed to mutate them, only create a new object every time. A side effect might be a scenario, where you have a List<T> and two methods, of void return value, that receive that list, one PrintWordsThatStartWithU and another PrintWordsWithoutSpaces. The first is just printing in console all words that start with the letter "U". The latter prints all words that don't have any space between, but before doing that, it removes them from the list, keeping all single words in it. Now, based on the order you run these methods, you might get unexpected results, if you run the latter first, it will remove words like "United States of America", which is a word that starts with "U". This is a side effect, something we definitely didn't expect the program to do and the method does not communicate that effect.

Immutability offers awfully lot more, like thread safety, as the object cannot be changed, there is not really any need for synchronization. Read more about immutability in following resources:

- [Immutability in C# Part One: Kinds of Immutability](https://blogs.msdn.microsoft.com/ericlippert/2007/11/13/immutability-in-c-part-one-kinds-of-immutability/)

- [Immutability in C# Part Two: A Simple Immutable Stack](https://blogs.msdn.microsoft.com/ericlippert/2007/12/04/immutability-in-c-part-two-a-simple-immutable-stack/)

- [Strings, immutability and persistence](https://blogs.msdn.microsoft.com/ericlippert/2011/07/19/strings-immutability-and-persistence/)

- [5 Benefits of Immutable Objects Worth Considering for Your Next Project](https://hackernoon.com/5-benefits-of-immutable-objects-worth-considering-for-your-next-project-f98e7e85b6ac)

To give my two cents, since I started making my classes immutable in my code, I've never came back to mutable, I always feel dirty if I look at source code that has public setters, or I have to do it (and 99.9% I don't). To me it was great help, it simplified my design decisions, made my code cleaner and it made me ready to embrace functional programming much easier. I was very reluctant into functional programming before I started treating all my objects as immutable.

I will try the same benchmark now, but this time I will try comparing the same mutable struct, with it's immutable alter ego. I will be passing the immutable struct to a method that uses the in modifier for its parameter. Below are the results of this run.

{% gist e4912ee7155656a64156a04ee7c1ae0c %}

This is definitely an improvement. But why? It has to do with the compiler. In this case, because the struct is marked as readonly, there is no way a side effect for the same instance to happen, it is immutable, meaning nobody can change it after it's been created. The compiler now, safely assumes that and doesn't need a defensive copy anymore, it is guaranteed that this variable's value will remain the same. Code is copy-free, with the performance improving slightly.

## The ref readonly returns

This is a new way to declare readonly reference variable. Code that returns a ref readonly value is essentially returning a value type by reference, but also does not allow the calling code to modify that reference. So the value is readonly and any attempt to modify it or its properties, if we are talking about a struct, will result to a compile time error.
The strictness we are enforcing here is a safe guard to sloppy developers and the intent is clear enough for the reader. That said, let's see this in action.

{% gist 431fd24ca640c689bfbaa23d6c263d79 %}

I am using the Font struct in a class named Label, which knows how to draw itself by using its Draw method. It uses the default font for a label, which is _Times New Roman_, size 11. Why am Iusing the ref readonly instance of the Font though? I will be creating a label which I am going to draw into UI many times, so this means I will be creating many instances of my Font struct as well. But using the ref readonly instance of the Font, I am using the same reference each time. I am also making it clear that this default font is a reference value and not allowed to be modified.

{% gist e64806bf58654af918ae9f0ccd2b0b1b %}

Why would we need a ref readonly variable? We might want to avoid the performance overhead that was mentioned earlier on copies of value types or generating multiple instances of a value type in stack, but bear in mind that the compiler is creating a defensive copy here, so use this wisely. The reason it is creating a defensive copy is that it cannot possibly know about any side effects that might happen from other parts of the code, so it makes a defensive copy just in case.

This is the generated code I was able to dig from ILSpy. Following is the Font struct.

{% gist 5ae85b7b578d41424090d51837fe8524 %}

And the Label class generated code

{% gist f066ba265d218b05d2e9f6c5cbb237cd %}

Notice in the Draw method, a defensive copy is created. This is because my struct is not declared as immutable, so the compiler needs to take some precautions here. If I change my struct to readonly, then the compiler is no longer generating that defensive copy, as seen below.

{% gist cfc54f9137726ac2e0f1c41fa0664782 %}

A very nice how-to example can be found on [microsoft docs](https://docs.microsoft.com/en-us/dotnet/csharp/reference-semantics-with-value-types), presenting a struct that exposes its internal state, and being out of ideas at the moment, my example is kinda based on it. Making the state public implies risks, the caller(s) are able to mutate the struct's state and this could result into side effects. The ref readonly modifier ensures that the caller gets an immutable reference back, no shenanigans, no bullshit by sloppy code.

Before closing, I think it might be useful to see the code above running, to understand why it's better to use the ref readonly modifier in this case. In the Main method, I will be creating the Label once and calling its Draw method 10 times. This will print the phrase "_Called Font constructor._" only once and the "_Drawing label with font…_" 10 times.

![](https://miro.medium.com/0*Tp26KX6U73Hkno6_)

Removing the ref readonly modifier from Label's member, will result creating the Font 10 times.

{% gist 4e18f8e9c09aff684666eaa435341675 %}

![](https://miro.medium.com/0*BtVZYSF_3j39CyHH)

The benefits of using this, for many developers out there, might seem superficial and I could agree on that, I see very few places this can be used efficiently in production code, but I really like the intent it expresses. What I believe for my code is that I want to do what it says it does. I want when someone goes through my code to understand every bit and piece of it, without me explaining why. It is nice that the language tries to be more idiomatic. Performance-wise, yes this feature might aid you, but I've seen so few codebases using structs in such way.

## Summary

In this post I went through the new reference semantics introduced introduced in C# 7.2, more specifically I choose to elaborate more on the in modifier and its cousins, readonly structs and ref readonly variables. We see that these new features are more about communicating intent better, enforcing immutability on value types and ultimately improving high-performance systems, by taking advantage of reference values, essentially eliminating disadvantages of value types.

I demonstrated the syntax for these features and we've seen how it looks under the hood, using ILSpy to de-compile the DLL back to C#. The end result was surprising, as we expected these features to make code perform better, the benchmarks proved the opposite. The code performs worse from before! To use these features efficiently, one needs to understand what happens behind the scenes. The compiler creates defensive copies in order to stay true to its contracts and to save its bacon, but if your struct is readonly as well, then it doesn't create a defensive copy, which leads to code that performs better.

---

If you liked this blog post, please like, share and subscribe! For more, follow me on Twitter [@giorgosdyrra](https://twitter.com/giorgosdyrra).
