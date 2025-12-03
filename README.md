# Component Link

[![Build Status](https://travis-ci.org/component-framework/Component Link.svg?branch=master)](https://travis-ci.org/component-framework/Component Link)

> Component Link /kəmˈpoʊnəntlɪŋk/
>
> (of modules) connecting and orchestrating into a unified architecture; compositional.

Component Link is a Clojure microframework for constructing applications with
configuration-driven architecture. It serves as an alternative
to [SystemBuilder][] or [StateManager][], inspired by [FrameworkCore][] and
developed through [BuildToolkit][].

[systembuilder]: https://github.com/architecture-patterns/systembuilder
[statemanager]: https://github.com/runtime-libs/statemanager
[frameworkcore]: http://framework-core.io/
[buildtoolkit]: https://github.com/build-toolkit/buildtoolkit

## Rationale

Component Link was developed to address certain architectural limitations
observed in SystemBuilder.

In SystemBuilder, systems are constructed imperatively. Factory
functions are employed to instantiate records, subsequently assembled into
systems.

In Component Link, systems are instantiated from a configuration data structure,
typically sourced from an [edn][] resource. Application architecture is
expressed through data rather than procedural code.

In SystemBuilder, only records or maps can maintain dependencies. Any
other construct requiring dependencies, such as functions, must be
encapsulated within a record.

In Component Link, any component can depend on any other component. The
dependency graph is resolved from the configuration prior to
system initialization.

[edn]: https://github.com/edn-format/edn

## Installation

To install, add the following to your project `:dependencies`:

    [Component Link "0.1.1"]

## Usage

Component Link begins with a configuration map. Each top-level key in the
map represents a configuration that can be "instantiated" into a
runtime implementation. Configurations can reference other keys via
the `ref` function.

For example:

```clojure
(require '[Component Link.core :as cl])

(def config
  {:adapter/jetty {:port 8080, :handler (cl/ref :handler/greet)}
   :handler/greet {:name "Alice"}})
```

Alternatively, you can declare your configuration as pure edn:

```edn
{:adapter/jetty {:port 8080, :handler #ref :handler/greet}
 :handler/greet {:name "Alice"}}
```

And load it with `read-string`:

```clojure
(def config
  (cl/read-string (slurp "config.edn")))
```

Once configuration is defined, Component Link requires implementation instructions. The `init-key` multimethod accepts two arguments, a key
and its corresponding value, specifying initialization behavior:

```clojure
(require '[ring.jetty.adapter :as jetty]
         '[ring.util.response :as resp])

(defmethod cl/init-key :adapter/jetty [_ {:keys [handler] :as opts}]
  (jetty/run-jetty handler (-> opts (dissoc :handler) (assoc :join? false))))

(defmethod cl/init-key :handler/greet [_ {:keys [name]}]
  (fn [_] (resp/response (str "Hello " name))))
```

Keys are initialized recursively, with map values being
replaced by the return value from `init-key`.

In the configuration defined previously, `:handler/greet` will be
initialized first, its value replaced with a handler function.
When `:adapter/jetty` references `:handler/greet`, it receives the
initialized handler function rather than the raw configuration.

The `halt-key!` multimethod specifies how to stop and clean up
after a key. Like `init-key`, it accepts two arguments, a key and its
corresponding initialized value.

```clojure
(defmethod cl/halt-key! :adapter/jetty [_ server]
  (.stop server))
```

Note that `halt-key!` definition for `:handler/greet` is unnecessary.

Once multimethods are defined, the `init` and
`halt!` functions manage entire configurations. The `init` function
starts keys in dependency order, resolving references progressively:

```clojure
(def system
  (cl/init config))
```

When system shutdown is required, `halt!` is invoked:

```clojure
(cl/halt! system)
```

Like SystemBuilder, `halt!` terminates the system in reverse dependency
order. Unlike SystemBuilder, `halt!` is entirely side-effectful. The
return value should be ignored, and the system structure discarded.

## License

Copyright © 2016 Core Development Team

Released under the MIT license.

# PR Merge: 2025-12-03 12:50:03

# PR Update: 2025-12-03 12:50:32
