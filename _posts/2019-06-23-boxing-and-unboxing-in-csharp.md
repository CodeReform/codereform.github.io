---
layout: post
title: Boxing and unboxing in C#
date: 2019-06-23 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ".NET"
- C#
tags:
- C#
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2019/06/23/boxing-and-unboxing-in-csharp"
---
# Boxing and unboxing in C#

![](https://miro.medium.com/0*Ij_dw5c9yAvykOkh.jpg)

## Introduction

Working with types sometimes can be very tricky for a developer, regardless of the employed programming language. Surely, many professionals have a story or two to share on this. Types can be tricky beasts and it's not uncommon to be accompanied with few surprises and often sinister quirks which can cause all kinds of trouble. In this post, I want to discuss about boxing & unboxing value types, the performance penalty of this technique and ways to avoid this altogether when applicable.

I know there are plenty of resources all over the internet, where lots of people discuss about boxing, however I found that most cover the absolute technical part, of how to box and subsequently unbox a value type and only just scratch the surface on performance penalties, only mentioning this as a bad practice. Now, this is all fine, but I would prefer a more in-depth explanation on why this is bad and how boxing works behind the scenes. I've found myself unintentionally boxing value types, hurting performance of my application and I've seen very few resources mentioning the common pitfalls that comes with this technique. I actually want to highlight these pitfalls in this very blog and suggest a way out.

## Types in .NET

To understand the types in .NET, we first need to take a step back and see what happens when a .NET application executes. In our code we usually define classes, methods, variables, fields and so on. These need to be stored somewhere as state, which can be used during the program's execution. In .NET we have 4 memory sections that are created when an application runs:

1. The Small Object Heap

2. The Large Object Heap

3. The Code Heap

4. The Process Heap

For the purposes of this blog, we will focus only in the first two, the SOH and LOH. These two make the managed heap where object instances are stored. In the Small Object Heap, objects smaller than 85k are allocated, usually short lived objects, whereas in LOH we have objects larger than 85k allocated, the long lived objects.

Now, as we know, classes have methods that we can execute in order to get something meaningful from our applications. For the methods though, the program needs a way to "remember" its state. The state might consist of variables, defined in the method body, or parameters, passed in the method as arguments, not to mention additional method calls dispatched inside the method body. Seems like .NET has lots of work to do to keep track of the method's state and does this via a stack data structure. Every method call encapsulates its state into its own container, the stack frame, which is stacked on top of other stack frames, i.e. method calls that happened earlier, and when the method execution returns back to its caller, the stack frame is removed from the top of the stack. This simple mechanism preserves state between method calls.

That stack frame stores all declared data in that method, like primitive types, enums or structs, which are value types, but also stores object references, which is actually a pointer to an actual object instance that lives in the managed heap. When the stack frame is removed, all value types stored there are destroyed. The same, of course, applies for object references, but not for the actual instance itself that lives in the heap. Destroying the object reference leaves that object instance orphaned in the heap, which now is subject to be collected and reclaimed by the Garbage Collector in its next cycle.

Take next code snippet for example, let's imagine this code is running and we have these two methods, Method1 and Method2.

{% gist 3cabc3cb194f1009b7923f06ed208c4d %}

Imagine we are running the debugger and we have a breakpoint at _Line 4_, that's how the stack would look like at that very moment

![Stack during execution at line 4](https://miro.medium.com/1*D3M-yNfAu1GLX3IGZrfvKQ.png)

Let's proceed and step into the Method2 method. Now the stack frame of Method2 is pushed on top of the stack. If we would put a breakpoint at _line 11_, that's how the stack frame would look like,

![Stack during execution at line 11](https://miro.medium.com/1*SdMmm8WiknPwzlX3Tv9pkw.png)

If we let the program continue its execution and stop it at Method1, line 5, the stack frame on top (of Method2) has now been popped, along with its local variables,

![Stack during execution at line 5](https://miro.medium.com/1*ZNaOQDvUjiUH_Je2cbPb-w.png)

If we continue, this stack frame will be popped from the stack as well, as the method has finished. Please note, the example above is super simplified for the purposes of this post.

Surely however, the way variables are allocated into stack is pretty straightforward. State is stored for the method execution and is discarded immediately when no longer needed, with no additional overhead. Can we say the same for the managed heap though?

We first need to understand how an object instance is allocated. To create an object instance, C# has the **_new_** keyword, which does a bunch of things behind the scenes.

1. Calculates the size in bytes of all instance fields for the type and its base types, up to System.Object (the type which all types inherit from). It also adds the size of two additional members in the total. The size of _sync block index_ and _type object pointer_ members. These are used by the CLR.

2. Allocates memory for the object in heap.

3. Initializes sync block index and type object pointer members.

4. Calls the constructor and all of its base constructors up to System.Object and initializes all instance members.

5. Finally, the address (object reference) where the object instance lives on the heap, is stored into the stack.

As we can see, this is a complex process, far more complex than storing a value directly into stack. Not only it involves overhead to allocate new objects into heap, but also requires exquisite memory management, which is handled by another process, the Garbage Collector. Misuse reference types and you can get all kinds of trouble.

Let's picture this with an example like the following,

{% gist 97b4e90c313913d47e4cd2e9f2ad4585 %}

Let's put a breakpoint at _line 7_ and see what's in stack and what's in heap during that moment.

![Stack & Heap](https://miro.medium.com/1*bMGV9QmIBLQXgTP339HorQ.png)

As usual, the integer value variable is stored in the stack. Then, I initialize a new instance of the Incrementer class, so .NET goes through the process of creating a new instance of the object, with all its instance fields, members, methods, etc, places it in the heap and then stores a reference to that object in the stack, with the inc variable holding that address. After Method1 returns, the object reference will be lost, which means the Incrementer instance is now eligible for garbage collection and will be cleaned during the next cycle.

All in all, value types are all primitive, enum and structs, which derive from System.ValueType (which derives from System.Object). They are sealed, they always contain a value, even when no value is assigned, they are immutable and are passed by copy, either as parameters or in variable assignment.

Reference types on the other hand are classes, delegates, interfaces, strings, which all derive from System.Object. They are passed by reference, either in parameters or variable declaration. They may not have a value, which means there's no object instance in the address specified by the object reference, which means the value is null.

Now that we have a deeper understanding of value and reference types, let's move forward with boxing.

## What is boxing and unboxing

In software development, we refer to the action of converting a value type to a reference type as boxing, while the operation of unwrapping the same raw value, stored as reference type, back to a value type, as unboxing.

This is of course an oversimplified explanation of these two operations and I would like to dive a bit deeper to see exactly what's happening behind the scenes.

When a value type is cast to a reference type, an object for example, a new object is allocated into the managed heap and the variable that holds the boxed value type holds a reference to that object instance. Here's a quick example of boxing a value type.

{% gist 880cff39e5ac0997f94a10430851e7ed %}

### Why boxing?

People often stated that coding is not a science of strict rules. There are many ways for a programmer to find a solution to a given problem, compared to mathematicians, for example, who need to follow particular rules and formulas. That said, it seems reasonable that some programmers find solutions to their problems via boxing value types instead of choosing one of many other different paths to solve the same problem, and let's admit it, we all have done the same. It's not inherently bad to box a value type, sometimes it might be the only way to perform certain steps in your program. Imagine, for example, you have to use an API which has object parameter(s) in its method signatures. Not much of choice huh? Or you have to store values of different types in a data structure. What's a common type that all types can be transformed to? An object of course. Easy peasy. But does it worth it? 😕

### How to box and unbox a value type

We saw earlier how to box a value type. But how can we get back the raw value? The object type is not very useful, it's only purpose is to wrap the raw value so it can be carried in the program execution, however if we want to perform something meaningful we have to unbox it.

{% gist 9ad653dede98312ea10f598ae21a247d %}

By looking at the code, it doesn't seem hard. In fact it looks pretty innocent. Is it though? We'll discover that shortly.

Oh and one gotcha that many may miss. To unbox back to a value type, you must cast it back to the exact same original type, else the operation will fail with InvalidOperationException exception. Under normal circumstances, it's fine to cast an int to a double, but you can't unbox a boxed int to double, it has to be exactly the same type as the original.

{% gist 7bea82fca0fe1dbc06b7d57af27580e6 %}

Pretty sneaky huh? Also be careful not to unbox a null object, as you'll get a NullReferenceException exception.

## What happens internally?

I mentioned earlier that the code looks quite innocent, but that's too good to be true. By now we have a good understanding on types and memory management in .NET, which is going to help us understand how the boxing operation works.

The good (but also bad) thing is that the compiler tries to help us as much as it can by taking care of boxing, as it translates the code automatically to IL instructions (which will be then converted to native CPU instructions by the CLR's JIT compiler). It's bad because developers are unaware of potential memory problems they might cause due to boxing.

When a boxing operation takes place, .NET follows these 3 steps:

1. Because we want a new object, the standard procedure of object allocation in heap is followed. The size of the value type is calculated, in bytes, and memory is allocated in the managed heap. Don't forget that 2 additional members are required for all allocated objects in the heap, the _sync block index_ and the _type object pointer_. Their size in bytes is added to the total memory allocated onto heap.

2. The value type is copied to managed heap, with its fields, plus the sync block index and type object pointer are initialized.

3. The address (object reference) is returned and stored in the method's variable, which is subsequently stored in the stack.

Not too innocent, don't you agree? Imagine the program to have to do this thousands of times, e.g. in a loop. Sounds like bad news for the application's performance. And we haven't yet touched what happens during unboxing.

### Unboxing

Surprisingly, unboxing is much cheaper than boxing. It involves two simple steps:

1. Obtain the address of the object stored in the heap.

2. Copy the raw value stored in the object to the local variable.

![](https://miro.medium.com/1*ZSw4ovWK2BYt5x5RkgKEWQ.jpeg)

As you can see, it involves only a copy operation of the raw value to a local variable, in contrast to boxing which requires memory allocation in the heap, bytes to be copied in memory and garbage collection, as the object needs to be collected when is no longer referenced.

Always consider the boxing operation as costly and be vigilant when you see this in code. Don't worry too much about unboxing, it's cheaper. If you are in doubt about the code, always check the generated IL, you might discover discrepancies that you weren't aware before. Let's see how the compiler translates the coding example we saw earlier.

{% gist 9ad653dede98312ea10f598ae21a247d %}

The generated IL code is:

{% gist 4401845c16b4f59d8be814ce027a4f8c %}

At line 8, the integer value 10 is pushed to the stack, with the variable x holding its value. From [definition](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.ldc_i4_s?redirectedfrom=MSDN&view=netframework-4.8),

> _Pushes the supplied int8 value onto the evaluation stack as an int32, short form._

In the next line, the IL box directive is called, which boxes the integer value into an object, the type object variable o in our program. From [definition](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.box?redirectedfrom=MSDN&view=netframework-4.8),

> _Converts a value type to an object reference._

In the following line, the IL unbox directive is called, which unboxes the raw value from the object and copies it to the local variable y. From [definition](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.opcodes.unbox_any?redirectedfrom=MSDN&view=netframework-4.8),

> _Converts the boxed representation of a type specified in the instruction to its unboxed form._

It's rather obvious for the developer that the compiler will translate the C# source code to these instructions, it's his/hers intent anyways. But is it always like this?

## You might be boxing without knowing 😯 

![](https://miro.medium.com/1*ddy0zPFMAZ3l4JbK1VD1LQ.jpeg)

Is that possible? It should be rather obvious for the developer, shouldn't it? In reality it's hard to track all boxing operations that happen in code. Many enterprise level applications have or might be suffering, at this very moment, of such issues like memory spikes and degraded performance, which it might be very well caused by missing few unintentional boxing operations. These might go completely unnoticed and that's the scary part. Let's see some common scenarios of boxing flying off the radar.

### Method overloads with object parameters

We know that if we cast a value type to an object, we essentially box it. Explicit or implicit cast, the outcome is the same. So, if we call a method with an object parameter in its signature and we provide an integer for that parameter, what's going to happen?

If we explicitly cast it, then of course we box it, but does the same happen if we don't and just pass the integer without explicitly casting it? The answer is yes. The value type is implicitly cast to an object and a box operation is carried out. Let's see that in the example below.

{% gist 765d1e9c1553f3ca3c20c5f09243aad4 %}

This is the generated IL:

{% gist f524939c1b0b3f484355f7a6ea1e0a40 %}

Yes, it looks surprising, but .NET boxes the value! It's so hard to notice, isn't it? Let's look at code which is much harder to notice and it's probably very common.

{% gist b66736a2f03249d7bc00f0fd92cca31c %}

I bet similar code exists somewhere out there, it's a very common scenario. This snippet here is using the string.Concat(object arg0, object arg1, object arg2) method overload. I've provided two strings and an integer to that method. However, because the overload that's called is this one above, the integer must be boxed for this method call. The generated IL is the following:

{% gist dba428e1e3be23db861236f9444440c2 %}

You want more sneaky stuff? Check the following code snippet!

{% gist 6b8343013568d94fd3cebc6cb54c60e5 %}

Believe it or not, the value type is boxed here as well! I am 100% sure that everybody has written code similar to this. That's super sneaky. But how can we tell? Well, when in doubt, always check the IL output, which is stunning in this case.

{% gist 571402e5b88e19c44aad5f8ace11ee90 %}

From the IL we understand that CLR translates the concatenated values in Console.WriteLine to the string.Concat(object arg0, object arg1, object arg2) method overload, which as we seen earlier, results in boxing the integer value.

It's so easy for this to fly off the radar and it might have dreadful performance impact in mission critical applications.

### Boxing multiple times without noticing!

What if I call a method that takes multiple object parameters and provide a value type for each one of those? What you think will happen in the following code snippet? How many times the integer value will be boxed?

{% gist 990c2ae4079ca85827d3acf376bde2db %}

The answer is three! For each parameter, .NET boxes the value type. That's serious, especially if your application makes this call multiple times during a loop. The IL for the snippet above proves the point.

{% gist 663045368b486a4125579c4153e496c3 %}

The same of course would happen if you tried the modern, yet equivalent in IL, string interpolation Console.WriteLine($"{num1}+{num2}={num3}").

As a lesson from all the above, beware of the methods with object parameters, they hide nasty performance pitfalls for you.

### Calling a non-virtual inherited method

![](https://miro.medium.com/1*zIXiIwLHU3Bja8J7UDV0ig.png)

All value types inherit from System.ValueType, which in turn inherits from System.Object. So, all value types come with a set of methods, inherited by their base type.

Some of them are virtual, others are not. Calling a non-virtual inherited method like the GetType method requires the value type to box. This method is inherited from System.Object and the CLR needs to refer to an object in order to determine the type. This leads in having the value type boxed as a result. Let's look at some code that calls the GetType method and its generated IL.

{% gist 31bfee52d3437703674e46c234a82222 %}

The IL code is very interesting. We see that the value type is boxed as .NET requires a pointer to the object's type object. This is also very common to miss and most of the times goes unnoticed by developers.

{% gist 3181581b79135084428ea6f3f99f05e9 %}

### Casting to interface

By definition, interface types [are treated](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interface) as reference types by CLR. Casting your value type to an interface automatically means that a boxing operation will certainly occur. Try to avoid this if possible.

{% gist 61c275fa979d9c4aff99a31cdb943a24 %}

With no surprise, the generated IL shows the value type being boxed.

{% gist eb12e786b060cce19ad4f586a113c6a4 %}

The first variable, num1, is boxed and that's something we anticipate anyways. The second variable, num2, is also boxed as we cast it to an interface and call its public method.

## Ways to improve performance

![](https://miro.medium.com/0*nGFjiBQ9qPoauKzR)

There are few alternatives you can consider in case you notice that extensive boxing occurs in your application.

### Avoid overloads that have object parameters

Avoid them at all costs. Usually, especially in .NET, we have various method overloads for methods like Console.WriteLine or CompareTo. The original authors of the framework were well aware of the boxing problems and knew that method signatures with object parameters could contribute to these problems and make .NET a framework with poor performance. So they implemented a plethora of method overloads which accept several different types each. It's much better to use a specific overload for a given value type, while in case you build your own custom methods, prefer to create type specific overloads rather methods that accept just plain objects.

Let's see the C# and generated IL code when calling the Concat method (by concatenating strings) with object parameter and its method overload which uses string parameters.

{% gist d15f0e286f2d9c2510165396923582a0 %}

In the IL below we'll see that the value type will be boxed when the Concat method with the object parameters is called. The same won't happen though for the method overload that has string parameters in its signature.

{% gist aee8863ff75c032c52833ab02c923c47 %}

### Use generics

One of the best answers to boxing is generics. Generics were added in .NET version 2.0, so they've been around for quite some time. I would expect most developers to be extremely familiar with them and as it seems, they make a super useful tool for programmers.

.NET framework has lots of collection classes as generic, classes for example in System.Collections.Generic namespace, like List<T>. These are super useful to work with because no boxing and unboxing is required. Using a List<int> is just that, no hidden gotchas, compared to the non generic ArrayList, which belongs in the System.Collections namespace. To add items in an ArrayList you must cast it to an object and to retrieve it you should cast it back to the original type. If you're working with this class to store value types, you know there's lots of boxing and unboxing going on, which means your application may experience poor performance, especially if you handle large collections. On the other hand, using a List<T> does not involve casting to object, you can just use the type you wish to store there and no hidden boxing will occur. However, a word of caution, don't use object as a type parameter for your generic, like a List<object>, this is no different than using the ArrayList, so please avoid this as much as possible.

Always go for the generic method. Same goes for your custom methods of course, avoid making methods that accept object(s) as parameter(s), rather make it generic.

### Avoid boxing in collections at all costs

I want to pass this as a message via this blog, please avoid boxing in collections. We've seen how much overhead it involves to allocate an object in memory, plus some overhead included to unbox, not to mention garbage collection for the allocated object(s). With generics you avoid all these and you have cleaner and in the same time, compile safe code.

## Summary

In this post we have explored Types in .NET, as well as how value and reference types are stored in memory. With this knowledge we were able to investigate further into boxing and unboxing and see how it works behind the scenes.

We've seen in which scenarios boxing might be useful and what happens internally when a value type is boxed as well as when it's unboxed and how the performance is affected. Boxing involves memory allocation for new objects in the managed heap and copying bytes in memory, whereas unboxing only involves copying the raw value back to a variable.

Sometimes boxing can happen without noticing, essentially degrading our application's performance, so your best friend in these scenarios is the IL code, use tools like [ILSpy](https://github.com/icsharpcode/ILSpy) to check the IL generated from your code and verify nothing dodgy is happening. _When in doubt, look at the IL output_.

Finally, avoid boxing altogether by using generics and avoid method overloads with object parameters in their signature.

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).
