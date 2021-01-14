---
layout: post
title: "JavaScript is Great"
post_ref: javascript_is_great
lang: en
categories:
tags:
  - javascript
  - programming languages
---

JavaScript is great.

```javascript
console.log("I'm pretty good!")
```

I'm not joking.

Being critical about JavaScript is commonplace between developers.
This patchy and ugly language, with a name that has been leaving HR people confunded for decades.
This language turned against basic aristotelian logic and that we only use because
is the only alternative on the web browser.
Let's forget one moment about the bad to see the good.

## Functional Programming

JavaScript is special between the most popular programming languages
for having functions-as-values from Day One, 1995.
Many other languages have incorporated similar features(Java 8 lambdas, C++11, C# 3.0, PHP 5)

Functional programming, a clear and elegant paradigm, has been the idiomatic JS way always.
[The most dependend upon package on npm](https://gist.github.com/anvaka/8e8fa57c7ee1350e3491#file-01-most-dependent-upon-md)
is [lodash](https://lodash.com/), a functional programming utilities library with [predecesors](https://underscorejs.org/) whose [earliest apperances](http://prototypejs.org/) where long ago.

## Asychronous Programming

"Callbacks?! Those suck".

Everybody agrees, including the comitee responsible for evolving JavaScript, which
is why they have introduced Promises and async/await.
In fact, JavaScript is also special in being asychronous from birth. This has had
a profound impact and paved the way for Node.js as a fast async server(no need for Apache's mpm_prefork).

```javascript
// Boring and nested
function doThing(callback) {
    talkToBackend(function (resultA) {
        talkTo3rdParty(resultA, function (resultB) {
            callback(resultA + resultB)
        })
    })
}

// Clean and slick
async function doThing() {
    const resultA = await talkToBackend()
    const resultB = await talkTo3rdParty()

    return resultA + resultB
}

// Parallel yeah.
// Don't even bother writing this with callbacks.
async function doThingParallel() {
    const [resultA, resultB] = await Promise.all([
        talkToBackend,
        talkTo3rdParty,
    ])

    return resultA + resultB
}
```

## About the funny behaviour...

"OK. But it really does [some stupid shit](https://www.destroyallsoftware.com/talks/wat). Equality is sometimes not transitive and sums are sometimes not commutative"

Yeah, some of those things are somewhat embarrasing, others are even funny.
Basically all those behaviours come from implicit type conversion to string and boolean.
What was a reasonable desing decision, the web is not strongly typed after all, the DOM is full of strings
and creating an ergonomic web language pushes for a loose type conversion criteria.

But most of these pains are avoidable, just use "===" instead of "==". Most linters will even force you to do so.
We can do even better though

## JavaScript is good, TypeScript is genius

TypeScript is a superset of JavaScript with optional typing that compiles to JavaScript

..., and it rocks.

With minimum effort it solves are pain associated with unintended type conversions, allows better autocompletion,
supports classes, modules, enums on every browser. I feel myself faster and safer on TypeScript than in almost any other language.

## The ecosystem is great

JavaScript ecosystem is gigantic, anyone could would be hard pressed to find a use case without a JS library,
many times of great quality and actively mantained.
Modern frameworks allow for great productivity and ease of coding. Nowadays JS is the only real technological stack
that reaches the web, mobile, server and desktop.

It is true that there are some problems, tool churn, dependency explosion, accidental complexity. Even then, these
are problems caused by the success of the JS world. Many programmers on less used langs would prefer to have these problems.

## JavaScript to the moon

JavaScript has realized the Java promise "Write Once, Run Everywhere" (maybe the JavaScript name wasn't even a bad one after all). And the language keeps moving: asm.js, WebAssembly, ES.Next, WebGL, LocalStorage, ... It is incredible that
a language designed just for providing a little bit of interactivity to the web has been proven to be so versatile.
Some even think it may [reach Ring 0](https://www.destroyallsoftware.com/talks/the-birth-and-death-of-javascript)(same author as the [Wat](https://www.destroyallsoftware.com/talks/wat) talk).

## Conclusion

JavaScript has some design pain points. It is easy to find those: null vs undefinied, monkey patching, "[Object object]",
memory and resource leaks, anidated callbacks, old and unconsistent language libraries, etc.

But I think we are lucky that JavaScript was the language of the web, and not, for example VBScript.

Maybe JavaScript doesn't love us very much, but it also doesn't hate us, I tend to think we get along pretty well
and I've gotten like it earnestly.
