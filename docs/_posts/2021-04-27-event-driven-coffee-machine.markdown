---
title:  "An Event-Driven REST Coffee Machine API with Clojure"
date: 2021-04-17 11:39:51 -0300
categories:
  - blog
---

## Introduction

[Skip introduction](#getting-our-hands-dirty)

In recent years developing software for an enterprise software, I had the experience to work with the antonym of a clean architecture. Business logic mixed with presentation and tightly coupled with IO. That made for an almost impossible to unit-test software without some herculean refactoring that was getting postponed _ad infitum_. It was on those bleak days that I missed writing FP-based projects. They are easy to reason about. The code is clean, functions are predictable and the state... and that is where I blanked. 

State management and component-glueing were very tricky to me. That was usually where most of the complexity of my projects with [Clojure](https://clojure.org/) were, even if incidental. That was until recently when I read about application state management in idiomatic Clojure. As usual, the language's community tend to have excellent taste when providing pragmatic libraries to solve non-trivial problems, and that is exactly what I think [Integrant](https://github.com/weavejester/integrant) is.  Integrant is a framework that provide a way of managing applications that are made out of smaller, dependent components in a data-driven fashion.

Right. So now we have solved the components state management problem. There's one missing piece of the puzzle though. How do we get our components to talk to each other in the most decoupled yet organized manner? Well, I got a hint when I stumbled upon [clojurewerkz's eep](https://github.com/clojurewerkz/eep). I derived my own little [event processing library](https://github.com/tiagodalloca/coffee-machine-rest-api/blob/master/src/coffee_machine_rest_api/events.clj) from some of its ideas. The implementation I ended up with is way simpler yet pretty ok I'd say. No fancy stuff, just `handler`s and `observer`s, which I will define precisely what they mean later in this post.

With these problems out our way, nothing can stop us from assembling a glorious coffee machine, that is easy to understand and to maintain (hopefully).

Enough talking. Let's get our hands dirty.

## Getting our hands dirty

As the big letters in the title implies, our goal is to provide access to a coffee machine using via a REST API. The main components will be:
- a **server** to listen to http requests for coffee (`POST: /api/brew-coffee`)
- an **event emitter** to dispatch and coordinate events handling, which is essentialy the *communication enabler* of our system
- a **coffee machine instance** from which to retrieve coffee and operate the core *business logic*


 
