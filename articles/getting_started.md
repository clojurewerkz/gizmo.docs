---
title: "Getting Started"
layout: article
---

## About this Guide

This guide will explain:

  * how to create a simple address book application using Gizmo
  * how to add new handlers, snippets and widgets
  * core concepts and conventions

If you want to learn Gizmo vocabulary and basic concepts first, you can
check out our [Basic Concepts](/articles/basic_concepts.html) guide.

## Gizmo overview

Gizmo is a collection of libraries for Web Application and RESTful
services development, that includes:

  * [Ring](https://github.com/ring-clojure/ring) for HTTP,
  * [Route One](https://github.com/clojurewerkz/route-one) for route generation,
  * [Compojure](https://github.com/weavejester/compojure) for route recognition,
  * [Enlive](https://github.com/cgrand/enlive) for templating,
  * [Chesire](https://github.com/dakrone/cheshire) for JSON.

And provides a set of non-obtrusive abstractions and conventions that
will help you to build your application fast, navigate and orient within
your code.

## Generating project from template

You can generate a new Gizmo project from the template for that, you can
run:

```
lein new gizmo-web my-webapp
```

You can also add a `--bootstrap` option to include Twitter Bootstrap
templates for fast prototyping:

```
lein new gizmo-web my-webapp --bootstrap
```

You're now able to launch your app by running:

```
lein run --config config/development.clj
```

Now you can visit `http://localhost:8080/` in browser, you should see a
greeting message there.

## Preparing layout

Layout is a outlining template that's shared between several pages on your
website. Usually it's a set of common surroundings of an HTML page, like
`<head>`, page header and footer and so on. Typical layout would have a
structure pretty much as follows:

<p class="text-muted">./resources/templates/layouts/application.html</p>
```html
<html>
  <head><!-- CSS imports, title, additional metadata  --></head>
  <body>
    <widget rel="my-webapp.widgets.shared/header" />

    <div class="container">
      <widget id="main-content" />
    </div>

    <widget rel="my-webapp.widgets.shared/footer />
    <!-- JavaScript imports  -->
  </body>
</html>
```

To make the page more modular, we've split it to three widgets:
__header__, __main-content__ and __footer__.

Widget declaration consists of two (optional) parts:

  * `rel` - specifies fully qualified name for the declared widget
    (we'll learn a bit more about widgets in just a minute). Used for
    widgets that do not change from page to page.
  * `id` - unique identifier for widget to reference it from code.
    used to inject widget `rel` dynamically in place of certain widget
    on a per-handler basis.

## Adding new HTTP actions

In order to create an HTML HTTP resopnse, you should create:

  * a `route` that will trigger a call to handler
  * a `handler`, that will specify widgets for page contents
  * `widget` and snippet and a HTML template to tie them all together

### Adding Route

All routes are located in `routes.clj` file of your application. By
default, in your `routes.clj` file will contain default routes (root
route, favicon route and fallback for all 404 urls, for which routing
system did not find any match):

```clj
(compojure/defroutes main-routes
  (GET root "/" request (my-webapp.handlers.home/index request))
  (GET favicon "/favicon.ico" _ (fn [_] {:render :nothing}))
  (route/not-found "Page not found"))
```

Let's add a new route for our address book. For that, we should specify an
HTTP verb (`GET` in our case), path (`/people`), specify request
argument (`request`) and provide a handler:

```clj
(GET people "/people" request (my-webapp.handlers.people/index request))
```

### Adding Handler

Handler is a function that receives an environment processed by all the
middlewares and returns a hash with following keys:

  * `:status` - HTTP Response Code
  * `:headers` - HTTP Headers
  * `:render` - response type for responder `:html` and `:json` are
    supported out of the box
  * `:widgets` - core widgets that should be injected into layout, only
    used during `:html` responses
  * `:layout` - which layout to be used by when rendering this response,
    only used during `:html` responses
  * `:response-hash` - JSON-serializable hash to be rendered as a
    response, only used during `:json` responses

Handlers are usually split logically per-entity (for example, all
handlers for people would sit in `*.handlers.people` namespace. There's
no  strict convention, but this is quite an intutive way, and we suggest
to do it that way.

Now, we can create an `index` handler in `my-webapp.handlers.people`:

<p class="text-muted">./src/my_webapp/handlers/home.clj</p>
```clj
(ns my-webapp.handlers.people)

(defn index
  [request]
  {:render :html
   :widgets {:main-content 'my-webapp.widgets.people/index-content}})
```

Handler now returns `:render :html` which instructs responder to render
response as HTML, and inject `index-content` widget in place of
`main-content` in the layout. You can jump back to [Layout section](#toc_3)
to check what `main-content` looks like.

### Adding Snippet

Now, let's move to widget and snippet definition. In order to create a
widget, we should first have snippet in place, since it's a view part
for it.

Snippet consists of two parts:

  * html code
  * enlive snippet definition

If you'd like to learn more about Enlive, you can jump straight to
[Enlive guide](https://github.com/cgrand/enlive/blob/master/README.md#quickstart-tutorial).

<p class="text-muted">./resources/templates/people/index.html</p>
```html
<div snippet="content">
  <h1>Address Book</h1>
  <table snippet="people-list">
    <tr class="person" snippet="people-list-item">
      <td>${name}</td>
      <td>${address}</td>
      <td>${phone}</td>
    </tr>
  </table>
</div>
```

In order to make it easier to write Enlive templates, we're using
auto-selectors. For that, you can add `snippet` attribute to your
element. For example, `snippet="people-list` will generate a
`*people-list` selector that will be availble during snippet definition:

<p class="text-muted">./src/my_webapp/handlers/people.clj</p>
```clj
(ns my-webapp.snippets.people
  (:require [net.cgrand.enlive-html :as html]
            [clojurewerkz.gizmo.enlive :refer [defsnippet within]]))

(defsnippet index-snippet "templates/people/index.html"
  [*content] ;; Top-level selector, specifies root HTML element for the snippet
  [people]
  (within *people-list [*people-list-item html/first-of-type]) ;; HTML selector that takes item from the list
          (html/clone-for [person people] ;; Clone list item for each person
                          [html/any-node] (html/replace-vars person))) ;; Replace dollar-escaped variables (like ${name}) for items in person hash
```

### Adding a Widget

Widget is a point that ties a `snippet` and request environment
together.

Widget consists of two parts: `view` and `fetch`. `fetch` is a
function that receives a complete environment from `handler`.

`fetch` is a function that receives an environment and runs some code,
potentially involving disk or network I/O. Sometimes `fetch` is used
just to get a part of environment that's applicable for a particular
view. This is especially useful when you need to reuse some entry
fetched from database between multiple widgets.

We recommend using Enlive for views, but `view` can return a string with
HTML elements generated by any other rendering engine, like Stencil,
Hiccup or your own HTML generation library. In our example, view will be
a snippet we just declared.

```clj
(ns my-webapp.widgets.people
    (:require [clojurewerkz.gizmo.widget :refer [defwidget]]
              [my-webapp.snippets.people :as snippets]))

(defwidget index-content
  :view snippets/index-snippet
  ;; For simplicity, we'll hardcode the items
  :fetch (fn [_] [{:id 1 :name "Alex" :address "Main Street 123" :phone "+1 123 123 12"}
                  {:id 2 :name "Robert" :address "Other Street 544" :phone "+1 123 123 12"}]))
```

Now, all everything is ready. You can navigate to
`http://localhost:8080/people` and check out the results.

## Project structure

We've created a project structure that allows you to separate concerns
in the application in the simplest way, but there's nothing that
prevents you from using your own structure. There's no hidden magic such
as convention over configuration, everything is explicit and
configurable.

When creating a new Gizmo project from
[template](https://github.com/ifesdjeen/gizmo-web-template), you get a
default project structure, that includes everything one needs to start
developing a web application:

```
├── config               configuration files (e.g. development.clj, production.clj)
├── resources
│   ├── public
│   │   ├── javascripts  JavaScripts, available via HTTP under /javascripts/*
│   │   └── stylesheets  CSS stylesheets, available via HTTP under /stylesheets/*
│   └── templates
│       ├── home         HTML templates specific to default `home` handler
│       ├── layouts      HTML templates for Layouts
│       └── shared       HTML templates for shared snippets
└── src
    └── my_webapp
        ├── handlers     handler namespaces
        ├── routes.clj   route definition
        ├── services     service namespaces
        ├── snippets     snippet namespaces
        └── widgets      widget namespaces
```
