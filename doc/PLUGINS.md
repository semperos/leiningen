# Leiningen Plugins

Leiningen tasks are simply functions named $TASK in a leiningen.$TASK
namespace. So writing a Leiningen plugin is just a matter of creating
a project that contains such a function.

Using the plugin is a matter of declaring it in the `:plugins` entry
of the project map. If a plugin is a matter of user convenience rather
than a requirement for running the project, you should place the
plugin declaration in the `:user` profile in `~/.lein/profiles.clj`
instead of directly in the `project.clj` file.

## Writing a Plugin

Start by generating a new project with `lein new plugin
lein-myplugin`, and edit the `myplugin` defn in the
`leiningen.myplugin` namespace. You'll notice the `project.clj` file
has `:eval-in-leiningen true`, which causes all tasks to operate
inside the leiningen process rather than starting a subprocess to
isolate the project's code. Plugins should not declare a dependency on
Clojure itself; in fact
[all of Leiningen's own dependencies](https://github.com/technomancy/leiningen/blob/master/project.clj)
should be considered implied dependencies of every plugin.

See the `lein-pprint` directory
[in the Leiningen source](https://github.com/technomancy/leiningen/tree/master/lein-pprint)
for a sample of a very simple plugin.

The first argument to your task function should be the current
project. It will be a map which is based on the `project.clj` file,
but it also has `:name`, `:group`, `:version`, and `:root` keys added
in, among other things. To see what project maps look like, try using
the `lein-pprint` plugin; then you can run `lein pprint` to examine
any project. If you want it to take parameters from the command-line
invocation, you can make the function take more arguments.

Most tasks may only be run in the context of another project. If your
task can be run outside a project directory, add `^:no-project-needed`
metadata to your task defn to indicate so. Your task should still
accept a project as its first argument, but it will be allowed to be
nil if it's run outside a project directory. If you are inside a
project, Leiningen should change to the root of that project before
launching the JVM, so `(System/getProperty "user.dir")` should be the
project root. The current directory of the JVM cannot be changed once
launched.

TODO: mention accepting :keyword args for certain things

The `lein help` task uses docstrings. A namespace-level docstring will
be used as the short summary if present; if not then it will take the
first line of your function's docstring. Try to keep the summary under
68 characters for formatting purposes. The full docstring can of
course be much longer but should still be wrapped at 80 columns. The
function's arglists will also be shown, so pick argument names that
are clear and descriptive. If you set `:help-arglists` in the
function's metadata, it will be used instead for those cases where
alternate arities exist that aren't intended to be exposed to the
user. Be sure to explain all these arguments in the docstring. Note
that all your arguments will be strings, so it's up to you to call
`read-string` on them if you want keywords, numbers, or symbols.

TODO: document subtasks and subtask help

If your task returns an integer, it will be used as the exit code for
the process. If tasks are chained together, a nonzero integer return
value will halt the chain and exit immediately. Throwing an exception
will also halt execution, but returning an integer will avoid showing
an unsightly stack trace.

## Code Evaluation

Plugin functions run inside Leiningen's process, so they have access
to all the existing Leiningen functions. The public API of Leiningen
should be considered all public functions inside the
`leiningen.core.*` namespaces not labeled with `^:internal` metadata
as well as each individual task functions. Other non-task functions in
task namespaces should be considered internal and may change inside
point releases.

Many tasks need to execute code inside the context of the project
itself. The `leiningen.core.eval/eval-in-project` function is used for
this purpose. It accepts a project argument as well as a form to
evaluate, and the final (optional) argument is another form called
`init` that is evaluated up-front before the main form. This may be
used to require a namespace earlier in order to avoid the
[Gilardi Scenario](http://technomancy.us/143). 

Inside the `eval-in-project` call the project's own classpath will be
active and Leiningen's own internals and plugins will not be
available. However, it's easy to update the project map
that's passed to `eval-in-project` to add in the dependencies you
need. For example, this is done in the `lein-swank` plugin like so:

```clj
(defn swank
  "Launch swank server for Emacs to connect. Optionally takes PORT and HOST."
  ([project port host & opts]
     (eval-in-project (update-in project [:dependencies] 
                                 conj ['swank-clojure "1.4.0"])
                      (swank-form project port host opts))))
```

The code in the `swank-clojure` dependency is needed inside the
project, so it's `conj`ed into the `:dependencies`.

The return value of the `eval-in-project` call is an integer that
represents the exit code of the project's process. Zero indicates
success. Be sure to use this as the return value of your task function
if appropriate.

TODO: mention prep-tasks

## Hooks

You can modify the behaviour of built-in tasks to a degree using
hooks. Hook functionality is provided by the [Robert
Hooke](https://github.com/technomancy/robert-hooke) library. This is an
implied dependency; as long as Leiningen 1.2 or higher is used it will
be available.

Inspired by clojure.test's fixtures functionality, hooks are functions
which wrap other functions (often tasks) and may alter their behaviour
by binding other vars, altering the return value, only running the function
conditionally, etc. The `add-hook` function takes a var of the task it's
meant to apply to and a function to perform the wrapping:

```clj
(ns leiningen.hooks.integration
  (:require [robert.hooke]
            [leiningen.test]))

(defn add-test-var-println [f & args]
  `(binding [~'clojure.test/assert-expr
             (fn [msg# form#]
               (println "Asserting" form#)
               ((.getRawRoot #'clojure.test/assert-expr) msg# form#))]
     ~(apply f args)))

;; Place the body of the activate function at the top-level for
;; compatibility with Leiningen 1.x
(defn activate []
  (robert.hooke/add-hook #'leiningen.test/form-for-testing-namespaces
                         add-test-var-println))
```

Hooks compose, so be aware that your hook may be running inside
another hook. To take advantage of your hooks functionality, projects
must set the `:hooks` key in project.clj to a seq of namespaces to load
that call `add-hook`. You may place calls to `add-hook` at the
top-level of the namespace, but if an `activate` defn is present it
will be called; this is the best place to put `add-hook` invocations.

If you need to use hooks from code that runs inside the project's
process, you may use `leiningen.core.injected/add-hook`, which is an
isolated copy of `robert.hooke/add-hook` injected into the project in
order to support features like test selectors.

See [the documentation for
Hooke](https://github.com/technomancy/robert-hooke/blob/master/README.md)
for more details.

## Clojure Version

Leiningen 2.0.0 uses Clojure 1.3.0. If you need to use a different
version of Clojure from within a Leiningen plugin, you can use
`eval-in-project` with a dummy project argument:

```clj
(eval-in-project {:dependencies '[[org.clojure/clojure "1.4.0-beta1"]]}
                 '(println "hello from" *clojure-version*))
```

## Upgrading Existing Plugins

Earlier versions of Leiningen had a few differences in the way plugins
worked, but upgrading shouldn't be too difficult.

The biggest difference between 1.x and 2.x is that `:dev-dependencies`
have been done away with. There are no longer any dependencies that
exist both in Leiningen's process and the project's process; Leiningen
only sees `:plugins` and the project only sees `:dependencies`, though
both these maps can be affected by the currently-active profiles.

If your project doesn't need to use `eval-in-project` at all, it
should be relatively easy to port; it's just a matter of updating any
references to Leiningen functions which may have moved. All
`leiningen.utils.*` namespaces have gone away, and `leiningen.core`
has become `leiningen.core.main`. For a more thorough overview see the
[published documentation on leiningen-core](http://technomancy.github.com/leiningen/).

Plugins that do use `eval-in-project` should just be aware that the
plugin's own dependencies and source will not be available to the
project. If your plugin currently has code that needs to run in both
contexts it must be split into multiple projects, one for `:plugins`
and one for `:dependencies`. See the example of `lein-swank` above to
see how to inject `:dependencies` in `eval-in-project` calls.

If your plugin may run outside the context of the project entirely,
you should still leave room in the arguments list for a project map;
just expect that it will be nil if there's no project present. Use
`^:no-project-needed` metadata to indicate this is acceptable.

## 1.x Compatibility

Once you've identified the changes necessary to achieve compatibility
with 2.x, you can decide whether you'd like to support 1.x and 2.x in
the same codebase. In some cases it may be easier to simply keep them
in separate branches, but sometimes it's better to support both.
Luckily the strategy of using `:plugins` and adding in `:dependencies`
just for calls to `eval-in-project` works fine in Leiningen 1.7. You
can even get support for profiles using `lein plugin install
lein-profiles 0.1.0`, though this support is experimental.

If you use functions that moved in 2.x, you can try requiring and
resolving at runtime rather than compile time and falling back to the
1.x versions of the function if it's not found. Again the `lein-swank`
plugin provides an example of a compatibility shim:

```clj
(defn eval-in-project
  "Support eval-in-project in both Leiningen 1.x and 2.x."
  [& args]
  (let [eip (or (try (require 'leiningen.core.eval)
                     (resolve 'leiningen.core.eval/eval-in-project)
                     (catch java.io.FileNotFoundException _))
                (try (require 'leiningen.compile)
                     (resolve 'leiningen.compile/eval-in-project)
                     (catch java.io.FileNotFoundException _)))]
    (apply eip args)))
```

Of course if the function has changed arities or has disappeared
entirely this may not be feasible, but it should suffice in most
cases.

Allowing the task to run outside a project directory is tricky to do
in a backwards-compatible way since 1.x is overly-clever and actually
inspects your argument list to figure out if it should pass in a
project argument, while 2.x simply always passes it in and just allows
it to be nil if it's not present. You can try checking the first
argument to see if it's a project map, but if you have more than two
arities this can get very tricky; it may just be better to maintain
separate branches of your codebase in this situation.

## Have Fun

Please add your plugins to [the list on the
wiki](http://wiki.github.com/technomancy/leiningen/plugins).

Hopefully the plugin mechanism is simple and flexible enough to let
you bend Leiningen to your will.
