---
title: "Getting Started"
layout: article
---

## About this Guide

This guide will explain:

  * how to create a simple address book application using Gizmo
  * how to add new handlers, snippets and widgets
  * core concepts and conventions


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

## Adding new HTTP actions

In order to create an HTML HTTP resopnse, you should create:

  * a route that will trigger a call to handler
  * a handler, that will specify widgets for page contents
  * widget and snippet and a HTML template to tie them all together

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

Handler is a function that returns a hash with following keys:

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


prepares the response and returns HTTP response code, response
body and content type and hands it over to responder. Depending on
response content type, an appropriate renderer is invoked (for
exmaple, HTML or JSON).


Handlers are usually split logically per-entity. There's no strict
convention, but this is quite an intutive way, and we suggest to stick
to that convention.

Now, we can create an `index` handler in `my-webapp.handlers.people`:

<p class="text-muted">./src/my_webapp/handlers/home.clj</p>
```clj
(ns my-webapp.handlers.people)

(defn index
  [request]
  {:render :html
   :widgets {:main-content 'my-webapp.widgets.people/index-content}})
```


### Adding Handler

<p class="text-muted">./src/my_webapp/handlers/people.clj</p>
```clj
(ns my-webapp.snippets.people
  (:require [net.cgrand.enlive-html :as html]
            [clojurewerkz.gizmo.enlive :refer [defsnippet within]]))

(defsnippet index-snippet "templates/people/index.html"
  [*index-content]
  [env])

```

<p class="text-muted">./resources/templates/people/index.html</p>
```html
<table snippet="people-list">
  <tr class="person">
    <td>%{name}</td>
    <td>%{address}</td>
    <td>%{phone}</td>
  </tr>
</table>
```


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
