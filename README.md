[![Travis Status][trav_img]][trav_site]
[![Coverage Status][cov_img]][cov_site]

Builder
=======

Builder takes your `npm` tasks and makes them composable, controllable from
a single point, and flexible.

`npm` is fantastic for controlling dependencies, tasks (via `scripts`) and
general project workflows. But a project-specific `package.json` simply doesn't
scale when you're managing many (say 5-50) very similar repositories.

_Enter Builder._ Builder is "almost" `npm`, but provides for off-the-shelf
"archetypes" to provide central sets of `package.json` `scripts`,
`dependencies` and `devDependencies`. The rest of this page will dive into
the details and machinations of the tool, but first here are a few of the
rough goals and motivations behind the project.

* **Single Point of Control**: A way to define a specific set of tasks /
  configs / etc. for one "type" of project. For example, we have an
  ever-expanding set of related repos for our
  [Victory](https://github.com/FormidableLabs/?utf8=%E2%9C%93&query=victory)
  project which all share a nearly-identical dev / prod / build workflow.
* **Flexibility**: There are a number of meta tools for controlling JavaScript
  workflows / development lifecycles. However, most are of the "buy the farm"
  nature. This works great when everything is within the workflow but falls
  apart once you want to be "just slightly" different. Builder solves this by
  allowing fine grain task overriding by name, where the larger composed tasks
  still stay the same and allow a specific repo's deviation from "completely off
  the shelf" to be painless.
* **You Can Give Up**: One of the main goals of builder is to remain very
  close to a basic `npm` workflow. So much so, that we include a section in this
  guide on how to abandon the use of Builder in a project and revert everything
  from archetypes back to vanilla `npm` `package.json` `scripts`, `dependencies`
  and `devDependencies`.

## Overview

At a high level `builder` is a tool for consuming `package.json` `scripts`
commands, providing sensible / flexible defaults, and supporting various scenarios
("archetypes") for your common use cases across multiple projects.

Builder is not opinionated, although archetypes _are_ and typically dictate
file structure, standard configurations, and dev workflows. Builder supports
this in an agnostic way, providing essentially the following:

* `NODE_PATH`, `PATH` enhancements to run, build, import from archetypes so
  dependencies and configurations don't have to be installed directly in a
  root project.
* A task runner capable of single tasks (`run`) or multiple concurrent tasks
  (`concurrent`).
* An intelligent merging of `package.json` `scripts` tasks.

... and that's about it!

### Usage

To start using builder, install and save `builder` and any archetypes you
intend to use. We'll use the [builder-react-component][] archetype as an
example.

**Note**: Most archetypes have an `ARCHETYPE` package and parallel
`ARCHETYPE-dev` npm package. The `ARCHETYPE` package contains _almost_
everything needed for the archetype (prod dependencies, scripts, etc.) except
for the `devDependencies` which the latter `ARCHETYPE-dev` package is solely
responsible for bringing in.

#### Global Install

For ease of use, one option is to globally install `builder` and locally install
archetypes:

```sh
$ npm install -g builder
$ npm install --save builder-react-component
$ npm install --save-dev builder-react-component-dev
```

Like a global install of _any_ Node.js meta / task runner tool (e.g., `eslint`,
`mocha`, `gulp`, `grunt`) doing a global install is painful because:

* You are tied to _just one_ version of the tool for all projects.
* You must also globally install the tool in CI, on servers, etc.

... so instead, we **strongly recommend** a local install described in the
next section!

To help you keep up with project-specific builder requirements, a globally-installed
`builder` will detect if a locally-installed version of `builder` is
available and switch to that instead:

```
$ /GLOBAL/PATH/TO/builder
[builder:local-detect] Switched to local builder at: ./node_modules/builder/bin/builder-core.js

... now using local builder! ...
```

#### Local Install

To avoid tying yourself to a single, global version of `builder`, the option
that we endorse is locally installing both `builder` and archetypes:

```sh
$ npm install --save builder
$ npm install --save builder-react-component
$ npm install --save-dev builder-react-component-dev
```

However, to call `builder` from the command line you will either need to either
augment `PATH` or call the long form of the command:

##### PATH Augmentation

Our recommended approach is to augment your `PATH` variable with a shell
configuration as follows:

**Mac / Linux**

```sh
# Safer version, but if you _have_ global installs, those come first.
export PATH="${PATH}:./node_modules/.bin"

# (OR) Less safe, but guarantees local node modules come first.
export PATH="./node_modules/.bin:${PATH}"

# Check results with:
echo $PATH
```

To make these changes **permanent**, add the `export` command to your `.bashrc`
or analogous shell configuration file.

**Windows**

```sh
# Safer version, but if you _have_ global installs, those come first.
set PATH=%PATH%;node_modules\.bin

# (OR) Less safe, but guarantees local node modules come first.
set PATH=node_modules\.bin;%PATH%

# Check results with:
echo %PATH%
```

To make these changes **permanent**, please see this multi-OS article on
changing the `PATH` variable: https://www.java.com/en/download/help/path.xml
(the article is targeted for a Java executable, but it's analogous to our
situation). You'll want to paste in `;node_modules\.bin` at the end _or_
`node_modules\.bin;` at the beginning of the PATH field in the gui. If there
is no existing `PATH` then add a user entry with `node_modules\.bin` as a value.
(It is unlikely to be empty because an `npm` installation on Windows sets the
user `PATH` analogously.)

##### Full Path Invocation

Or you can run the complete path to the builder script with:

**Mac / Linux**

```sh
node_modules/.bin/builder <action> <task>
```

**Windows**

```sh
node_modules\.bin\builder <action> <task>
```

#### Configuration

After `builder` is available, you can edit `.builderrc` like:

```yaml
---
archetypes:
  - builder-react-component
```

to bind archetypes.

... and from here you are set for `builder`-controlled meta goodness!

#### Builder Actions

Display general or command-specific help (which shows available specific flags).

```sh
$ builder [-h|--help|help]
$ builder help <action>
$ builder help <archetype>
```

Run `builder help <action>` for all available options. Version information is
available with:

```sh
$ builder [-v|--version]
```

Let's dive a little deeper into the main builder actions:

##### builder run

Run a single task from `script`. Analogous to `npm run <task>`

```sh
$ builder run <task>
```

Flags:

* `--builderrc`: Path to builder config file (default: `.builderrc`)
* `--tries`: Number of times to attempt a task (default: `1`)
* `--setup`: Single task to run for the entirety of `<action>`.

##### builder concurrent

Run multiple tasks from `script` concurrently. Roughly analogous to
`npm run <task1> | npm run <task2> | npm run <task3>`, but kills all processes on
first non-zero exit (which makes it suitable for test tasks), unless `--no-bail`
is provided.

```sh
$ builder concurrent <task1> <task2> <task3>
```

Flags:

* `--builderrc`: Path to builder config file (default: `.builderrc`)
* `--tries`: Number of times to attempt a task (default: `1`)
* `--setup`: Single task to run for the entirety of `<action>`.
* `--queue`: Number of concurrent processes to run (default: unlimited - `0|null`)
* `--[no-]buffer`: Buffer output until process end (default: `false`)
* `--[no-]bail`: End all processes after the first failure (default: `true`)

Note that `tries` will retry _individual_ tasks that are part of the concurrent
group, not the group itself. So, if `builder concurrent --tries=3 foo bar baz`
is run and bar fails twice, then only `bar` would be retried. `foo` and `baz`
would only execute _once_ if successful.

##### builder envs

Run a single task from `script` concurrently for each item in an array of different
environment variables. Roughly analogous to:

```sh
$ FOO=VAL1 npm run <task> | FOO=VAL2 npm run <task> | FOO=VAL3 npm run <task>
```

... but kills all processes on first non-zero exit (which makes it suitable for
test tasks), unless `--no-bail` is provided. Usage:

```sh
$ builder envs <task> <json-array>
$ builder envs <task> --envs-path=<path-to-json-file>
```

Examples:

```sh
$ builder envs <task> '[{ "FOO": "VAL1" }, { "FOO": "VAL2" }, { "FOO": "VAL3" }]'
$ builder envs <task> '[{ "FOO": "VAL1", "BAR": "VAL2" }, { "FOO": "VAL3" }]'
```

Flags:

* `--builderrc`: Path to builder config file (default: `.builderrc`)
* `--tries`: Number of times to attempt a task (default: `1`)
* `--setup`: Single task to run for the entirety of `<action>`.
* `--queue`: Number of concurrent processes to run (default: unlimited - `0|null`)
* `--[no-]buffer`: Buffer output until process end (default: `false`)
* `--[no-]bail`: End all processes after the first failure (default: `true`)
* `--envs-path`: Path to JSON env variable array file (default: `null`)

###### Custom Flags

Just like [`npm run <task> [-- <args>...]`](https://docs.npmjs.com/cli/run-script),
flags after a ` -- ` token in a builder task or from the command line are passed
on to the underlying tasks. This is slightly more complicated for builder in
that composed tasks pass on the flags _all the way down_. So, for tasks like:

```js
"scripts": {
  "down": "echo down",
  "way": "builder run down -- --way",
  "the": "builder run way -- --the",
  "all": "builder run the -- --all"
}
```

We can run some basics (alone and with a user-added flag):

```sh
$ builder run down
down

$ builder run down -- --my-custom-flag
down --my-custom-flag
```

If we run the composed commands, the `--` flags are accumulated:

```sh
$ builder run all
down --way --the --all

$ builder run all -- --my-custom-flag
down --way --the --all --my-custom-flag
```

The rough heuristic here is if we have custom arguments:

1. If a `builder <action>` command, append with ` -- ` to pass through.
2. If a non-`builder` command, then append without ` -- ` token.

## Tasks

The underlying concept here is that `builder` `script` commands simply _are_
npm-friendly `package.json` `script` commands. Pretty much anything that you
can execute with `npm run <task>` can be executed with `builder run <task>`.

Builder can run 1+ tasks based out of `package.json` `scripts`. For a basic
scenario like:

```js
{
  "scripts": {
    "foo": "echo FOO",
    "bar": "echo BAR"
  }
}
```

Builder can run these tasks individually:

```sh
$ builder run foo
$ builder run bar
```

Sequentially via `||` or `&&` shell helpers:

```sh
$ builder run foo && builder run bar
```

Concurrently via the Builder built-in `concurrent` command:

```sh
$ builder concurrent foo bar
```

With `concurrent`, all tasks continue running until they all complete _or_
any task exits with a non-zero exit code, in which case all still alive tasks
are killed and the Builder process exits with the error code.

## Archetypes

Archetypes deal with common scenarios for your projects. Like:

* [builder-react-component][]: A React component
* A React application server
* A Chai / jQuery / VanillaJS widget

Archetypes typically provide:

* A `package.json` with `builder`-friendly `script` tasks.
* Dependencies and dev dependencies to build, test, etc.
* Configuration files for all `script` tasks.

In most cases, you won't need to override anything. But, if you do, pick the
most granular `scripts` command in the archetype you need to override and
define _just that_ in your project's `package.json` `script` section. Copy
any configuration files that you need to tweak and re-define the command.

### Task Resolution

The easiest bet is to just have _one_ archetype per project. But, multiple are
supported. In terms of `scripts` tasks, we end up with the following example:

```
ROOT/package.json
ROOT/node_modules/ARCHETYPE_ONE/package.json
ROOT/node_modules/ARCHETYPE_TWO/package.json
```

Say we have a `.builderrc` like:

```yaml
---
archetypes:
  - ARCHETYPE_ONE
  - ARCHETYPE_TWO
```

The resolution order for a `script` task (say, `foo`) present in all three
`package.json`'s would be the following:

* Look through `ROOT/package.json` then the configured archetypes in _reverse_
  order: `ARCHETYPE_TWO/package.json`, then `ARCHETYPE_ONE/package.json` for
  a matching task `foo`
* If found `foo`, check if it is a "passthrough" task, which means it delegates
  to a later instance -- basically `"foo": "builder run foo"`. If so, then look
  to next instance of task found in order above.

### Special Archetype Tasks

Archetypes use conventional `scripts` task names, except for the following
special cases:

* `"npm:postinstall"`
* `"npm:preversion"`
* `"npm:version"`
* `"npm:test"`

These tasks are specifically actionable during the `npm` lifecycle, and
consequently, the archetype mostly ignores those for installation by default,
offering them up for actual use in _your_ project.

As an **additional restriction**, non-`npm:FOO`-prefixed tasks with the same
name (e.g., `FOO`) _may_ call then `npm:`-prefixed task, but _not_ the other
way around. So

```js
// Good / OK
"npm:test": "builder run test-frontend",
"test": "builder run npm:test",

// Bad
"npm:test": "builder run test",
"test": "builder run test-frontend",
```

### Creating an Archetype

Moving common tasks into an archetype is fairly straightforward and requires
just a few tweaks to the paths defined in configuration and scripts in order
to work correctly.

#### Initializing your project

An archetype is simply a standard npm module with a valid `package.json`. To set
up a new archetype from scratch, make a directory for your new archetype,
initialize NPM and link it for ease of development.

```sh
$ cd path/to/new/archetype
$ npm init
$ npm link
```

From your consuming project, you can now link to the archetype directly for ease
of development after including it in your `dependencies` and creating a
`.builderrc` as outlined above in [configuration](#configuration).

```sh
$ cd path/to/consuming/project
$ npm link new-archetype-name
```

#### Managing the `dev` archetype

Because `builder` archetypes are included as simple npm modules, two separate
npm modules are required for archetypes: one for normal dependencies and one for
dev dependencies. Whereas in a non-builder-archetype project you'd specify dev
dependencies in `devDependencies`, with `builder` all dev dependencies must be
regular `dependencies` on a separate dev npm module.

`builder` is designed so that when defining which archetypes to use in a
consuming project's `.builderrc`, `builder` will look for two modules, one named
appropriately in `dependencies` (ex: `my-archetype`) and one in
`devDependencies` but with `-dev` appended to the name (ex: `my-archetype-dev`).

To help with managing these while building a builder archetype, install
[`builder-support`](https://github.com/FormidableLabs/builder-support)
to create and manage a `dev/` directory within your archetype project with it's
own `package.json` which can be published as a separate npm module.
`builder-support` will not only create a `dev/package.json` with an appropriate
package name, but will also keep all the other information from your archetype's
primary `package.json` up to date as well as keep `README.md` and `.gitignore`
in parity for hosting the project as a separate npm module.

Get started by installing and running `builder-support gen-dev`:

```sh
$ npm install builder-support --save-dev
$ ./node_modules/.bin/builder-support gen-dev
```

_TIP: Create a task called `"builder:gen-dev": "builder-support gen-dev"` in
your archetype to avoid having to type out the full path each time you update
your project's details._

For ease of development, `npm link` the dev dependency separately:

```sh
$ cd dev
$ npm link
```

Then from your consuming project, you can link to the dev package.

```sh
$ cd path/to/consuming/project
$ npm link new-archetype-name-dev
```

Read the [`builder-support` docs](https://github.com/FormidableLabs/builder-support)
to learn more about how dev archetypes are easily managed with
`builder-support gen-dev`.

#### Workflow for moving dependencies and scripts to your new archetype

Once everything is configured and `npm link`'d, it should be easy to move
scripts to your archetype and quickly test them out from a consuming project.

##### Moving `dependencies` and `devDependencies` from an existing `package.json`

* copy `dependencies` to `package.json` `dependencies`.
* copy `devDependencies` to `dev/package.json` `dependencies`.

##### Moving scripts and config files

All scripts defined in archetypes will be run from the root of the project
consuming the archetype. This means you have to change all paths in your scripts
to reference their new location within the archetype.

An example script and config you may be moving to an archetype would look like:

```js
"test-server-unit": "mocha --opts test/server/mocha.opts test/server/spec"
```

When moving this script to an archetype, we'd also move the config from
`test/server/mocha.opts` within the original project to within the
archetype such as `config/mocha/server/mocha.opts`.

For this example script, we'd need to update the path to `mocha.opts` as so:

```js
"test-server-unit": "mocha --opts node_modules/new-archetype-name/config/mocha/server/mocha.opts test/server/spec"
```

Any paths that reference files expected in the consuming app (in this example
`test/server/spec`) do not need to change.

##### Updating path and module references within your script configs

Any JavaScript files run from within an archetype (such as config files) require
a few changes related to paths now that the files are being run from within
an npm module. This includes all `require()` calls referencing npm modules and
all paths to files that aren't relative.

For example, `karma.conf.js`:

```js
module.exports = function (config) {
  require("./karma.conf.dev")(config);

  config.set({
    preprocessors: {
      "test/client/main.js": ["webpack"]
    },
    files: [
      "sinon/pkg/sinon",
      "test/client/main.js"
    ],
  });
};
```

All non-relative paths to files and npm modules need to be full paths, even ones
not in the archetype directory. For files expected to be in the consuming
project, this can be achieved by prepending `process.cwd()` to all paths. For
npm modules, full paths can be achieved by using
[`require.resolve()`](https://nodejs.org/api/globals.html#globals_require_resolve).

An updated config might look like:

```js
var path = require("path");
var ROOT = process.cwd();
var MAIN_PATH = path.join(ROOT, "test/client/main.js");

module.exports = function (config) {
  require("./karma.conf.dev")(config);

  config.set({
    preprocessors: {
      [MAIN_PATH]: ["webpack"]
    },
    files: [
      require.resolve("sinon/pkg/sinon"),
      MAIN_PATH
    ],
  });
};
```

#### Example `builder` archetype project structure

```
.
├── CONTRIBUTING.md
├── HISTORY.md
├── LICENSE.txt
├── README.md
├── config
│   ├── eslint
│   ├── karma
│   ├── mocha
│   │   ├── func
│   │   │   ├── mocha.dev.opts
│   │   │   └── mocha.opts
│   │   └── server
│   │       └── mocha.opts
│   └── webpack
│       ├── webpack.config.coverage.js
│       ├── webpack.config.dev.js
│       ├── webpack.config.hot.js
│       ├── webpack.config.js
│       └── webpack.config.test.js
├── dev
│   └── package.json
└── package.json
```

## Tips, Tricks, & Notes

### Project Root

Builder uses some magic to enhance `NODE_PATH` to look in the root of your
project (normal) and in the installed modules of builder archetypes. This
latter path enhancement sometimes throws tools / libraries for a loop. We
recommend using `require.resolve("LIBRARY_OR_REQUIRE_PATH")` to get the
appropriate installed file path to a dependency.

This comes up in situations including:

* Webpack loaders
* Karma included files

The other thing that comes up in our Archetype configuration file is the
general _requirement_ that builder is running from the project root, not
relative to an archetype. However, some libraries / tools will interpret
`"./"` as relative to the _configuration file_ which may be in an archetype.

So, for these instances and instances where you typically use `__dirname`,
an archetype may need to use `process.cwd()` and be constrained to **only**
ever running from the project root. Some scenarios where the `process.cwd()`
path base is necessary include:

* Webpack entry points, aliases
* Karma included files (that cannot be `require.resolve`-ed)

### Other Process Execution

The execution of tasks generally must _originate_ from Builder, because of all
of the environment enhancements it adds. So, for things that themselves exec
or spawn processes, like `concurrently`, this can be a problem. Typically, you
will need to have the actual command line processes invoked _by_ Builder.

### Terminal Color

Builder uses `exec` under the hood with piped `stdout` and `stderr`. Programs
typically interpret the piped environment as "doesn't support color" and
disable color. Consequently, you typically need to set a "**force color**"
option on your executables in `scripts` commands if they exist.

### Why Exec?

So, why `exec` and not `spawn` or something similar that has a lot more process
control and flexibility? The answer lies in the fact that most of what Builder
consumes is shell strings to execute, like `script --foo --bar "Hi there"`.
_Parsing_ these arguments into something easily consumable by `spawn` and always
correct is quite challenging. `exec` works easily with straight strings, and
since that is the target of `scripts` commands, that is what we use for Builder.

### I Give Up. How Do I Abandon Builder?

Builder is designed to be as close to vanilla npm as possible. So, if for
example you were using the `builder-react-component` archetype with a project
`package.json` like:

```js
"scripts": {
  "postinstall": "builder run npm:postinstall",
  "preversion": "builder run npm:preversion",
  "version": "builder run npm:version",
  "test": "builder run npm:test",
  /* other deps */
},
"dependencies": {
  "builder": "v2.0.0",
  "builder-react-component": "v0.0.5",
  /* other deps */
},
"devDependencies": {
  "builder-react-component-dev": "v0.0.5",
  /* other deps */
}
```

and decided to _no longer_ use Builder, here is a rough set of steps to unpack
the archetype into your project and remove all Builder dependencies:

* Copy all `ARCHETYPE/package.json:dependencies` to your
  `PROJECT/package.json:dependencies` (e.g., from `builder-react-component`).
  You _do not_ need to copy over `ARCHETYPE/package.json:devDependencies`.
* Copy all `ARCHETYPE/package.json:scripts` to your
  `PROJECT/package.json:scripts` that do not begin with the `builder:` prefix.
  Remove the `npm:` prefix from any `scripts` tasks and note that you may have
  to manually resolve tasks of the same name within the archetype and also with
  your project.
* Copy all `ARCHETYPE-dev/package.json:dependencies` to your
  `PROJECT/package.json:devDependencies`
  (e.g., from `builder-react-component-dev`)
* Copy all configuration files used in your `ARCHETYPE` into the root project.
  For example, for `builder-react-component` you would need to copy the
  `builder-react-component/config` directory to `PROJECT/config` (or a renamed
  directory).
* Review all of the combined `scripts` tasks and:
    * resolve duplicate task names
    * revise configuration file paths for the moved files
    * replace instances of `builder run <task>` with `npm run <task>`
    * for `builder concurrent <task1> <task2>` tasks, first install the
      `concurrently` package and then rewrite to:
      `concurrent 'npm run <task1>' 'npm run <task2>'`

... and (with assuredly a few minor hiccups) that's about it! You are
Builder-free and back to a normal `npm`-controlled project.

### Versions v1, v2, v3

The `builder` project effectively starts at `v2.x.x`. Prior to that Builder was
a small DOM utility that fell into disuse, so we repurposed it for a new
wonderful destiny! But, because we follow semver, that means everything starts
at `v2` and as a helpful tip / warning:

> Treat `v2.x` as a `v0.x` release

We'll try hard to keep it tight, but at our current velocity there are likely
to be some bumps and API changes that won't adhere strictly to semver until
things settle down in `v3.x`-on.

[builder-react-component]: https://github.com/FormidableLabs/builder-react-component
[trav_img]: https://api.travis-ci.org/FormidableLabs/builder.svg
[trav_site]: https://travis-ci.org/FormidableLabs/builder
[cov]: https://coveralls.io
[cov_img]: https://img.shields.io/coveralls/FormidableLabs/builder.svg
[cov_site]: https://coveralls.io/r/FormidableLabs/builder
