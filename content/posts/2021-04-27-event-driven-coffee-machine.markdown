---
title:  "An Event-Driven REST Coffee Machine API with Clojure"
date: 2021-04-17
categories:
  - blog
tags:
  - software
  - architecture
  - clojure
---

Today I'm going to show you how to build a REST API for a coffee machine using:
- Event-Driven communication between components
- Clojure, a language for writing correct and robust software with less ceremony
- Integrant for application state management
- reitit for routing http requests

You can find the [source code](https://github.com/tiagodalloca/coffee-machine-rest-api) for this project in my github.

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

Each component will depend on some dependencies, which will be explicit in our code as we inject the dependencies with Integrant.

I don't want to spend your time showing the regular stuff. So lets jump right into the core concept, that's why I'm writing this post.

### Event emitter: laying the wires for async communication between components

As I briefly aluded about in the [Introduction](#introduction), event handling and dispatching are going to be the core constructs in which we'll build our app's communication between components.

The event shapes is: `[event-t & args]`, where `event-t` is the event type/identifier (which is any hashable object but I highly recommend using Clojure's namespaced keywords) followed by the arguments that will be passed on for the handlers and observers.

Let's see some code.

```clojure
(use 'coffee-machine-rest-api.events)

(def emitter (create-emitter {:immediately-start? false}))

(start-listening emitter)

(add-handler
   emitter :hi
   (fn [[msg] handler-promise]
     (Thread/sleep 1000)
     (when handler-promise (deliver handler-promise "hello there!"))))
     
(add-observer
   emitter :hi :logger
   (fn [[msg]] (println (str "logging> " msg))))

(deref (dispatch-event emitter [:hi "hi"]))
;; prints "logging> hi"
;; Thread sleeps for 1000ms
;;=> "hello there"
```

The `emitter` is a regular Clojure map that holds all the state necessary for routing events to registered handlers and observers, which are added via `add-handler` and `add-observer`. Handlers must be associated with an `event-t`, as `:hi` in the example above, and observers must be associated with an id (`:logger`).

The component that is firing an event should be completely agnostic regarding any observer, as an observer should do just that: observe (logging could be a good use case).

A handler, on the other hand, consists of a function which receives the event's args and can return something using the `handler-promise`, which is a plain `promise` that is returned after `dispatch-event`. The pairing between `event-t`s and handlers is 1:1, so an event type can be associated with only one handler. 

Let's see this bad boy in action.

### Handling http requests with reitit

```clojure
(ns coffee-machine-rest-api.rest-api.handler
  (:require [coffee-machine-rest-api.events :as events] ;; more requires...
            ))

(defn- get-routes [{:keys [emitter] :as deps}]
  ["/api"
   ["/brew-coffee"
    {:post
     {:parameters {:body [:map [:coffee-id keyword?] [:money double?]]}

      :handler (fn [{{{:keys [coffee-id money]} :body} :parameters :as request}]
                 (let [event-ret @(events/dispatch-event
                                   emitter
                                   [::brew-coffee coffee-id money]
                                   {:enforce-handler true})]
                   (if (instance? Exception event-ret)
                     (throw event-ret)
                     {:body event-ret})))}}]])
```

I chose to start this section with some code right away because that's pretty much it. We are going to receive `coffee-id` and `money` ('cause we ain't no charity) as parameters in the requests `body` and dispatch `[::brew-coffee coffee-id money]`. Also, note the options `{:enforce-handler true}` map we're passing after the event. This will enforce that the event is handled, otherwise we get an exception as result of the `promise` `event-ret`.

If everything is ok, we simply return the a map with its `body` containing what is returned by the handler of the event `::brew-coffee`.

Who shall be the one to handle such an important task as brewing coffee for the people?

### Brewing some coffee

```clojure
(use 'coffee-machine-rest-api.coffee-machine)

;; ...

(def coffee-machine (create-coffee-machine
                       {:coffees {"Affogato" 1.00
                                  "Caffè Latte" 1.50
                                  "Caffè Mocha" 2.00}
                        :available-coins [0.50 1.00 0.10 0.25]}))
(request-coffee coffee-machine :caffe-latte 2.10)
;;=>
;;{:coffee-instance
;;  {:name "Caffè Latte",
;;   :price 1.5,
;;   :created-at "2021-04-27T15:46:12.928Z"},
;;  :change {1.0 0, 0.5 1, 0.25 0, 0.1 1},
;;  :change-value 0.6}
```

From a `coffee-machine` instance we will `request-coffee`.

What about handling the event of brewing?

That's when we put it together.

### System config

```clojure
(ns coffee-machine-rest-api.system
  (:require [coffee-machine-rest-api.coffee-machine :as coffee-machine]
            [coffee-machine-rest-api.events :as events]
            [coffee-machine-rest-api.rest-api :as api]
            [coffee-machine-rest-api.rest-api.handler :as api-handler]

            [integrant.core :as ig]))

(def config
  {::emitter  {:opts {:pool-size 4
                      :chan-buf-size 10
                      :immediately-start? true}}

   ::coffee-machine {:opts {:coffees {"Affogato" 1.00
                                      "Caffè Latte" 1.50
                                      "Caffè Mocha" 2.00}
                            :available-coins [0.50 1.00 0.10 0.25]}}
   
   ::server {:opts {:port 6942}
             :handler (ig/ref ::handler)}

   ::handler {:emitter (ig/ref ::emitter)}

   ::api-events-handlers
   {:emitter (ig/ref ::emitter)
    :coffee-machine-instance (ig/ref ::coffee-machine)
    :opts
    {::api-handler/brew-coffee
     (fn [{:keys [coffee-machine-instance]}]
       (fn [[coffee-id money] p]
         (->> (coffee-machine/request-coffee coffee-machine-instance coffee-id money)
              (deliver p))))}}})

;; ...

(defmethod ig/init-key ::api-events-handlers [_ {:keys [emitter
                                                        coffee-machine-instance
                                                        opts] :as args}]
  (let [handlers-deps {:coffee-machine-instance coffee-machine-instance}]
    (doseq [[event-t f] opts]
      (events/add-handler emitter event-t (f handlers-deps))))
  
  args)

;; ...

```

This is the config of our system that will be initilized by Integrant. I'd like you to pay close attention to the `::api-events-handlers` component and how we are initilizing it. The goal is to populate the `emitter` with the required handlers already injected with the dependencies to handle the events. How? Passing the dependencies to a function which in return gives us a handler ready to be added into `emitter`. 

## Wrapping up

Holly molly, this was longer to write than I expected. Between redundancies and crypt Clojure code, I hope you learned something useful! I think event-driven communication between components and clean state management (as with Integrant) makes for a very pleasent developing experience and easy to reason about architecture.

Also as for next steps, we could add events handlers validations, as in to explicitly enforce and describe which handlers a component depends on.

That's it for today's post!

![Lawrence](/images/and_send_it_to_the_internet.gif)
