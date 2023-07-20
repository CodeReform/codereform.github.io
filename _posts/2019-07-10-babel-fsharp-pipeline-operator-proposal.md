---
layout: post
image: https://miro.medium.com/1*JZoffTvJsUbUz003vzW0Gw.jpeg
title: Babel F# pipeline operator proposal
date: 2019-07-10 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- "JavaScript"
tags:
- ES6
- "functional programming"
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2019/07/10/babel-fsharp-pipeline-operator-proposal"
---
# Babel F# pipeline operator proposal

## Introduction

Writing readable and declarative code in JavaScript, how nice it would be! 😆

I know, _readable_ and _declarative_ don't go in the same sentence when talking about JavaScript, but should it be always like this? The community has been looking for ways to improve these problems with the language for years.

Since Babel 7.0 there have been a few proposals implemented that lean towards the readable and declarative JavaScript goal. I'm sure you have heard about the pipeline operator, but if you haven't then it's fine, this post will guide you through. This operator is syntactic sugar with the intention to make our code really sweet (pun intended). In this post I'm going to discuss one of the competing proposals, the F# variant.

## Babel pipeline operator proposal

The community has been working on the pipeline operator proposal for quite sometime now. To be precise, they managed to release the minimal variant of this proposal in [Babel 7.0.0-beta](https://babeljs.io/blog/2018/08/27/7.0.0).

The [minimal](https://github.com/tc39/proposal-pipeline-operator/wiki#proposal0-original-minimal-proposal) proposal covers only the basics around the pipeline operator, so features like partial function application and async/await don't work.

They decided to play around with this proposal and work with few more variants on the same operator, which actually extend the minimal variant. Some proposed variants didn't make it, like [Hack](https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-2-hack-style-only) or [Split](https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-3-split-mix) style variants, but we have two at the moment, which both are available in [Babel 7.5.0](https://babeljs.io/blog/2019/07/03/7.5.0) version. The people who are working on this proposal want to compare the trade-offs between the two competing proposals, which as I mentioned earlier, are based on the minimal variant.

One of the competing proposals is the [Smart](https://github.com/js-choi/proposal-smart-pipelines/blob/master/readme.md) style variant, which was introduced in [Babel 7.3.0](https://babeljs.io/blog/2019/01/21/7.3.0), while the other is the [F#](https://github.com/valtech-nyc/proposal-fsharp-pipelines/blob/master/README.md) style variant, which has been introduced in Babel 7.5.0. To use any of these three variants (minimal, smart, [F#](https://github.com/valtech-nyc/proposal-fsharp-pipelines/blob/master/README.md)), you must use a Babel plugin. More on this below.

## The operator

![](https://miro.medium.com/1*wiXBiJHIAPPE58M7b5Mbow.jpeg)

The pipeline operator `(|>)` takes two arguments. The second one, on the right side, is a function, while the first one, on the left side, is a value. The pipeline operator provides that value as an argument to the function on the right and returns the result.

If you think about it, it's very similar to chaining functions, like the old-fashioned jQuery for example, however it's not limited to intrinsic methods of an object and also it's way more declarative.

{% gist 84ca8d194957ed791849b41ac7654b81 %}

This technique is commonly used to pipe multiple operations in one go. As discussed above, the operator passes a value from the left side to a function on the right side. For the next operation, it passes the result of the previously executed operation, which is a value, to the function on the right and so on and so forth. This continues, until there's no operation left, i.e. no more pipelining.

{% gist 36a81526f5188ec7324424c28cde5800 %}

Such coding style is very common in FP languages like F#, OCaml, Elixir, etc. and also it can be combined with other powerful functional programming techniques like [partial function application](https://en.wikipedia.org/wiki/Partial_application) and [function composition](https://en.wikipedia.org/wiki/Function_composition_(computer_science)).

## Why use it?

Generally speaking, in the functional world, this is a very useful operator. FP languages are notorious of their declarative programming style, which results in more readable code, well at least for people familiar with the language, as for developers who come from an OOP background the code might look cryptic. 😥

However, it's proven when using these languages, the clutter is minimal and the code has a tendency to read naturally from left to right, like a proper sentence, which makes it easier for one's brain to follow the code flow and finally make sense from the jargon.

## The F# pipeline operator in JavaScript

This operator borrows its style from FP languages like F#, OCaml, Elm, etc. and makes it easier for developers to develop functional solutions in JavaScript. I must inform you that this proposal is still at [Stage 1](https://github.com/tc39/proposals/blob/master/stage-1-proposals.md) and it's been evaluated, so expect things to change in the future, as the proposal matures.

## The variants

The F# and Smart variants are quite similar in how they operate, however the smart variant uses a # topic reference. For this variant, each step of the pipeline creates its own lexical scope, with the # topic reference being the result of the previous step. You don't need to use arrow functions, you just provide the result via the topic reference.

In the F# pipeline, you must use arrow functions, a style which is more close to the JavaScript we all know and love. The partial function application is not supported out of the box if you choose the F# variant, so make sure to install and add it as a plugin in your .babelrc file if you wish to use this technique. You will need the [@babel/plugin-proposal-pipeline-operator](https://babeljs.io/docs/en/babel-plugin-proposal-pipeline-operator) plugin for the pipeline operator to transpile.

```
npm install [@babel/plugin-proposal-pipeline-](http://twitter.com/babel/plugin-proposal-pipeline-)operator - save-dev
```

And in the .babelrc set the plugin as follows for fsharp.

{% gist 5ae1d2d3eda72926c13771d9317ca89b %}

**Note**: For other variants, set the proposal property either to minimal or smart.

The [partial application proposal](https://babeljs.io/docs/en/babel-plugin-proposal-partial-application) supports the ? reference, which is similar to the # topic reference in the smart variant, but you can pass it _only_ in function calls.

```
npm install [@babel/plugin-proposal-partial-a](http://twitter.com/babel/plugin-proposal-partial-a)pplication - save-dev
```

Then add it to your babel plugins in .babelrc as follows.

{% gist d9d16ed45a34675be1d5b6aca176c6ed %}

### Examples

Let's see an example with the F# variant. First, you have to download and install the babel plugin as shown above. I am going to use partial function application, so make sure to install and add as plugin the partial application proposal plugin as well. See the gist above about the plugins.

{% gist 15f7830a3e2313423d40f4dec2ba3407 %}

In this example I have a list of order items, which makes an order. I want to calculate the final price for the items in stock and apply a discount of 20%, as the client has a coupon. The operations I mentioned earlier are all performed in sequence thanks to the pipeline operator. Notice the applyCoupon function returns a function which receives a total, but I don't provide the total explicitly in the parameters. This is called partial function application, the value is passed implicitly by the pipeline operator.

Finally, let's see how this works with async/await.

{% gist d0cb5086fcb570bc470c7f7b97c4ea8c %}

In this example I want to fetch a country from a remote API and transform the response to an object that I want to show in console. Because I am running these examples in NodeJS and not in browser, I've installed a package called node-fetch, in order to use the fetch API.

The pipeline and the await operator work very nice together as we can see. Notice the await operator must go _after_ the promise/asynchronous code, which in this example are the fetch and json methods, which both return a promise. So, always remember to put your await _after_ the promise when pipelining. Think of await as a function, which takes one Promise parameter, unwraps it and returns the underlying value.

You can find additional motivating examples at the TC39 Pipeline Operator proposal [page](https://github.com/tc39/proposal-pipeline-operator#motivating-examples) and explore other interesting [use cases](https://github.com/tc39/proposal-pipeline-operator/wiki/Example-Use-Cases). You can use the pipeline operator in several cases in your code, like in a [validation scenario](https://github.com/tc39/proposal-pipeline-operator#validation), where you want to validate specific attributes of an object, or you can use it to [create mixins](https://github.com/tc39/proposal-pipeline-operator#mixins) in a much more concise way, or to [create object decorators](https://github.com/tc39/proposal-pipeline-operator#object-decorators) as well as many other [use cases](https://github.com/tc39/proposal-pipeline-operator/wiki/Example-Use-Cases) that are presented in the official proposal.

## Why nesting function calls is bad?

This is the traditional way, it's the "pipelining before it was cool". With this technique we nest multiple function calls, making something that resembles a Christmas tree.

{% gist 9f4eacdfc0fdd8c299a22a6968eec657 %}

Please admit it. _It's ugly_. That's the reason the _pipe_ function was invented, which does the same as above, a bit more elegantly.

## Why choose this over the pipe() method?

Libraries like [Ramda](https://ramdajs.com/) and [RxJS](https://rxjs-dev.firebaseapp.com/) provide a pipe() function which you can use to pipe operations, very much like the pipeline operator. Techniques such these have been around for few years, as no pipeline operator existed before in JavaScript. But now, with its arrival, it manages to make nested function calls look clumsy, wordy and obsolete. I have prepared an example, one with [RxJS](https://rxjs-dev.firebaseapp.com/) and one with [Ramda](https://ramdajs.com/) and finally, the native approach with the pipeline operator. Let's make it functional all the way.

With this example, I have to first define my functions.

{% gist 651d4d1a7d3a1dceac2985b7b204cd13 %}

Now, here's the pipe function in action.

{% gist aa065ad8e00d210f07678b238f42304e %}

I have to nest the functions in the pipe function, list them in order, so they execute in sequence and finally subscribe to the observable, which is returned by the pipe function, in order to read the raw value.

The same example now in Ramda. I must admit I like Ramda more when working with Functional stuff in JS, mostly for its simplicity. I'll setup my functions first in a similar manner.

{% gist 0ccaa43189576046a8e597e30437fad0 %}

Now, I call the pipe method of Ramda and I provide the phrase as a parameter. In the same fashion, all functions in my pipe will run sequentially.

{% gist e7d23fc461bd2613f548b54dd484d723 %}

Finally, it's time for the pipeline operator. As with previous examples, I will set my functions upfront. I've made them a bit more functional compared to previous examples, plus they use the pipeline operator for internal operations.

{% gist d516f20ecd5a2405fc1a9da805b3f6b4 %}

Now that my functions are ready to use, let's get the result.

{% gist 512007bcd3bedabdb48883e600cfef9e %}

I like the latter most, I don't have to learn a new library and its intricacies nor I need to install it and import it, I am just using native stuff and Babel will transpile it accordingly for me.

Notice also how different this looks from the example which nests multiple function calls. The order is completely opposite. The pipelining example reads naturally, while the nesting example is awkward and hard to understand.

<div class="tenor-gif-embed" data-postid="4402454" data-share-method="host" data-aspect-ratio="2.09231" data-width="80%"><a href="https://tenor.com/view/great-gif-4402454">Great Gatsby - Great GIF</a>from <a href="https://tenor.com/search/great-gifs">Great GIFs</a></div> <script type="text/javascript" async src="https://tenor.com/embed.js"></script>

Another advantage is that you don't have to use third party libraries like RxJS or at least some of their functions like pipe(), which only adds to your bundle size. You'll have to worry for less bandwidth to download for your consumers and less libraries to support in-project, for your developers.

It's awesome how the language has progressed, from being so fragmented and badly implemented, to a better version of itself through tools like Babel and Typescript. Less and less utility libraries are needed currently to develop a web application, with developers finally putting more trust in native code.

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).
