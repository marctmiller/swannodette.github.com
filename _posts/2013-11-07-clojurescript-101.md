---
layout: post
title: "ClojureScript 101"
description: ""
category: 
tags: ["clojurescript" "core.async"]
---
{% include JB/setup %}

While few of the ideas in the core.async are new, understanding how to
solve problems with [CSP]() is simply not as well documented as using
plain callbacks or [Promises](). My previous posts have mostly
explored fairly sophisticated uses of [core.async], this post instead
takes the form of a very basic tutorial.

We're going to demonstrate all the steps required to build a simple search
interface and we'll see how core.async provides some unique solutions
to problems common to client side user interface programming.

You can follow along by using
[mies](http://github.com/swannodette/mies). I recommend use Google
Chrome so that you can get good [source map]() support. You don't need
Emacs to have fun with Lisp. SublimeText 2 is pretty nice these days,
I recommend installing the [paredit](http://github.com/odyssomay/paredit) and [lispindent](http://github.com/odyssomay/sublime-lispindent) packages via
[Sublime Package Control](http://sublime.wbond.net/installation).

If you have
[Leiningen](http://github.com/technomancy/leiningen) installed you can
run the following at the command line:

```
lein new mies async-tut1
```

Change the `:dependencies` in `project.clj` file to look like the following:

```
:dependencies [[org.clojure/clojure "1.5.1"]
               [org.clojure/clojurescript "0.0-2030"]
               [org.clojure/core.async "0.1.256.0-1bf8cf-alpha"]] ;; ADD
```

In the project directory run the following to start the auto compile
process:

```
lein cljsbuild auto async-tut1
```

We're going to demonstrate all the steps required to build a simple search
interface.

First off we want to add the following markup to `index.html` before
the first script tag which loads `goog/base.js`:

```
<input id="query" type="text"></input>
<button id="search">Search</button>
<p id="results"></p>
```

Open `index.html` and make sure you see a input field and a text
button.

Now we want to write some code so that we can interact with the
DOM. We want our code to be resilient to browser differences so we'll
use Google Closure to abstract this stuff away as we might with jQuery.

So we want to require `goog.dom` and give it a less annoying alias:

```
(ns async-tut1.core
  (:require [goog.dom :as dom]))
```

We want to confirm that this will work so let's change
`src/async_tut1/core.cljs` so that the `console.log` looks this
instead:

```
(. js/console (log (dom/getElement "query")))
```

Save the file and it should be recompiled instantly. We should be able
refresh the browser and see that a DOM element got printed in the
JavaScript console (**View > Developer > JavaScript Console**). Remove
this little test snippet after you've confirmed it works.

So far so good.

Now we want a way to deal with the user clicking the mouse. Instead of
just setting up a call back on the button directly we're going to make
the button put the click event onto a core.async channel.

Let's write a little helper called `listen` that will already return a
channel of any DOM element's events. First we need to require
core.async macros and functions, our `ns` should look like the following now:

```
(ns async-tut1.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [goog.dom :as dom]
            [goog.events :as events]
            [cljs.core.async :refer [put! chan >! <!]]))
```

Again we want to abstract away browser quirks so we use `goog.events`
for dealing with that. We include only the `core.async` macros and
functions that we intend to use.

Now we can write our `listen` fn, it looks like this:

```
(defn listen [el type]
  (let [out (chan)]
    (events/listen el type
      (fn [e] (put! out e)))
    out))
```

We want very out function works as advertised we can check it with
following bit of code:

```
(let [clicks (listen (dom/getElement "search") "click")]
  (go (while true
        (.log js/console (<! clicks)))))
```

Note that we've created what appears to be an infinite loop here, but
actually it's a little state machine. If there are no events to read
from the click channel, the go block will be suspended!

Let's search Wikipedia. Lets define the basic URL we are going to hit
via [JSONP](). Put this right after the `ns` form.

```
(def wiki-search-url
  "http://en.wikipedia.org/w/api.php?action=opensearch&format=json&search=")
```

Now we want to make a function that returns a channel for JSONP
results.

We again reach for Google Closure to avoid browser quirks. Make your
`ns` form looking like the following:

```
(ns async-tut1.core
  (:require-macros [cljs.core.async.macros :refer [go go-loop]])
  (:require [goog.dom :as dom]
            [goog.events :as events]
            [cljs.core.async :refer [>! <! put! chan]])
  (:import [goog.net Jsonp]
           [goog Uri]))
```

Here we use `:import` so that we can use short names for the
constructors.

Our JSONP helper looks like the following (put it after `listen` in
the file):

```
(defn jsonp [uri]
  (let [out (chan)
        req (Jsonp. (Uri. uri))]
    (.send req nil (fn [res] (put! out res)))
    out))
```

This looks pretty straight forward, very similar to `listen`. Let
write a simple function for constructing a query url:

```
(defn query-url [q]
  (str wiki-search-url q))
```

Again lets test this by writing a snippet of code at the bottom of the file.

```
(go (.log js/console (<! (jsonp (query-url "cats")))))
```

In the JavaScript Console we should see we got an array of JSON data
back from Wikipedia. Success!

It's time to hook everything together. Remove the test snippet and
replace it with the following:

```
(defn user-query []
  (.-value (dom/getElement "query")))

(defn init []
  (let [clicks (listen (dom/getElement "search") "click")]
    (go (while true
          (<! clicks)
          (.log js/console (<! (jsonp (query-url (user-query)))))))))

(init)
```

Try it now, you should be able to write a query in the input field,
click "Search", and see results in the JavaScript Console.

If you've done any JavaScript programming this way of writing the code
should be somewhat surprising - we don't need a callback to work with
button clicks!

Think a bit how this works, when the page loads, `init` will run, the
`go` block will try to read from `clicks`, but there will be nothing
to read so, so the `go` block becomes suspended. Only you click on the
button can it proceed at which point we'll run the query and loop. The
code reads exactly how it would if you didn't have to consider
asynchrony!

Instead of printing to the console we would like to render the results
to the page. Let's do that now add the following before `init`:

```
(defn render-query [results]
  (str
    "<ul>"
    (apply str
      (for [result results]
        (str "<li>" result "</li>")))
    "</ul>"))
```

The usually string concatenation stuff, we use a list comprehension
here just for fun.

Now change `init` to look like the following:

```
(defn init []
  (let [clicks (listen (dom/getElement "search") "click")
        results-view (dom/getElement "results")]
    (go (while true
          (<! clicks)
          (let [[_ results] (<! (jsonp (query-url (user-query))))]
            (set! (.-innerHTML results-view) (render-query results)))))))
```

Hopefully this code at this point just makes sense. Notice how we can
use destructuring on the JSON array of Wikipedia results.

A beautiful succinct program!

```
(ns async-tut1.core
  (:require-macros [cljs.core.async.macros :refer [go go-loop]])
  (:require [goog.dom :as dom]
            [goog.events :as events]
            [cljs.core.async :refer [>! <! put! chan]])
  (:import [goog.net Jsonp]
           [goog Uri]))

(def wiki-search-url
  "http://en.wikipedia.org/w/api.php?action=opensearch&format=json&search=")

(defn listen [el type]
  (let [out (chan)]
    (events/listen el type
      (fn [e] (put! out e)))
    out))

(defn jsonp [uri]
  (let [out (chan)
        req (Jsonp. (Uri. uri))]
    (.send req nil (fn [res] (put! out res)))
    out))

(defn query-url [q]
  (str wiki-search-url q))

(defn user-query []
  (.-value (dom/getElement "query")))

(defn render-query [results]
  (str
    "<ul>"
    (apply str
      (for [result results]
        (str "<li>" result "</li>")))
    "</ul>"))

(defn init []
  (let [clicks (listen (dom/getElement "search") "click")
        results-view (dom/getElement "results")]
    (go (while true
          (<! clicks)
          (let [[_ results] (<! (jsonp (query-url (user-query))))]
            (set! (.-innerHTML results-view) (render-query results)))))))

(init)
```