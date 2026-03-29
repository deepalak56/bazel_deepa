Project: /_project.yaml
Book: /_book.yaml

# The Bazel codebase

{% include "_buttons.html" %}

This document is a description of the codebase and how Bazel is structured. It
is intended for people willing to contribute to Bazel, not for end-users.

## Introduction {:#introduction}

The codebase of Bazel is large (~350KLOC production code and ~260 KLOC test
code) and no one is familiar with the whole landscape: everyone knows their
particular valley very well, but few know what lies over the hills in every
direction.

In order for people midway upon the journey not to find themselves within a
forest dark with the straightforward pathway being lost, this document tries to
give an overview of the codebase so that it's easier to get started with
working on it.

The public version of the source code of Bazel lives on GitHub at
[github.com/bazelbuild/bazel](http://github.com/bazelbuild/bazel). This is not
the "source of truth"; it's derived from a Google-internal source tree that
contains additional functionality that is not useful outside Google. Tests for this Google-specific functionality are also kept internal. The
long-term goal is to make GitHub the source of truth.

Contributions are accepted through the regular GitHub pull request mechanism,
and manually imported by a Googler into the internal source tree, then
re-exported back out to GitHub.

## Client/server architecture {:#client-architecture}

The bulk of Bazel resides in a server process that stays in RAM between builds.
This allows Bazel to maintain state between builds.

This is why the Bazel command line has two kinds of options: startup and
command. In a command line like this:

```
    bazel --host_jvm_args=-Xmx8G build -c opt //foo:bar
```

Some options (`--host_jvm_args=`) are before the name of the command to be run
and some are after (`-c opt`); the former kind is called a "startup option" and
affects the server process as a whole, whereas the latter kind, the "command
option", only affects a single command.

Each server instance has a single associated workspace (collection of source
trees known as "repositories") and each workspace usually has a single active
server instance. This can be circumvented by specifying a custom output base
(see the "Directory layout" section for more information).

Bazel is distributed as a single ELF executable that is also a valid .zip file.
When you type `bazel`, the above ELF executable implemented in C++ (the
"client") gets control. It sets up an appropriate server process using the
following steps:

1.  Checks whether it has already extracted itself. If not, it does that. This
    is where the implementation of the server comes from.
2.  Checks whether there is an active server instance that works: it is running,
    it has the right startup options and uses the right workspace directory. It
    finds the running server by looking at the directory `$OUTPUT_BASE/server`
    where there is a lock file with the port the server is listening on.
3.  If needed, kills the old server process
4.  If needed, starts up a new server process

After a suitable server process is ready, the command that needs to be run is
communicated to it over a gRPC interface, then the output of Bazel is piped back
to the terminal. Only one command can be running at the same time. This is
implemented using an elaborate locking mechanism with parts in C++ and parts in
Java. There is some infrastructure for running multiple commands in parallel,
since the inability to run `bazel version` in parallel with another command
is somewhat embarrassing. The main blocker is the life cycle of `BlazeModule`s
and some state in `BlazeRuntime`.

At the end of a command, the Bazel server transmits the exit code the client
should return. An interesting wrinkle is the implementation of `bazel run`: the
job of this command is to run something Bazel just built, but it can't do that
from the server process because it doesn't have a terminal. So instead it tells
the client what binary it should `exec()` and with what arguments.

When one presses Ctrl-C, the client translates it to a Cancel call on the gRPC
connection, which tries to terminate the command as soon as possible. After the
third Ctrl-C, the client sends a SIGKILL to the server instead.

The source code of the client is under `src/main/cpp` and the protocol used to
communicate with the server is in `src/main/protobuf/command_server.proto` .

The main entry point of the server is `BlazeRuntime.main()` and the gRPC calls
from the client are handled by `CommandServer.serveAndAwaitTermination()`.

## Directory layout {:#directory-layout}

Bazel creates a somewhat complicated set of directories during a build. A full
description is available in [Output directory layout](/remote/output-directories).

The "main repo" is the source tree Bazel is run in. It usually corresponds to
something you checked out from source control. The root of this directory is
known as the "workspace root".

Bazel puts all of its data under the "output user root". This is usually
`$HOME/.cache/bazel/_bazel_${USER}`, but can be overridden using the
`--output_user_root` startup option.

The "install base" is where Bazel is extracted to. This is done automatically
and each Bazel version gets a subdirectory based on its checksum under the
install base. It's at `$OUTPUT_USER_ROOT/install` by default and can be changed
using the `--install_base` command line option.

The "output base" is the place where the Bazel instance attached to a specific
workspace writes to. Each output base has at most one Bazel server instance
running at any time. It's usually at `$OUTPUT_USER_ROOT/<checksum of the path
to the workspace>`. It can be changed using the `--output_base` startup option,
which is, among other things, useful for getting around the limitation that only
one Bazel instance can be running in any workspace at any given time.

The output directory contains, among other things:

*   The fetched external repositories at `$OUTPUT_BASE/external`.
*   The exec root, a directory that contains symlinks to all the source
    code for the current build. It's located at `$OUTPUT_BASE/execroot`. During
    the build, the working directory is `$EXECROOT/<name of main
    repository>`. We are planning to change this to `$EXECROOT`, although it's a
    long term plan because it's a very incompatible change.
*   Files built during the build.

## The process of executing a command {:#command-process}

Once the Bazel server gets control and is informed about a command it needs to
execute, the following sequence of events happens:

1.  `BlazeCommandDispatcher` is informed about the new request. It decides
    whether the command needs a workspace to run in (almost every command except
    for ones that don't have anything to do with source code, such as version or
    help) and whether another command is running.

2.  The right command is found. Each command must implement the interface
    `BlazeCommand` and must have the `@Command` annotation (this is a bit of an
    antipattern, it would be nice if all the metadata a command needs was
    described by methods on `BlazeCommand`)

3.  The command line options are parsed. Each command has different command line
    options, which are described in the `@Command` annotation.

4.  An event bus is created. The event bus is a stream for events that happen
    during the build. Some of these are exported to outside of Bazel under the
    aegis of the Build Event Protocol in order to tell the world how the build
    goes.

5.  The command gets control. The most interesting commands are those that run a
    build: build, test, run, coverage and so on: this functionality is
    implemented by `BuildTool`.

6.  The set of target patterns on the command line is parsed and wildcards like
    `//pkg:all` and `//pkg/...` are resolved. This is implemented in
    `AnalysisPhaseRunner.evaluateTargetPatterns()` and reified in Skyframe as
    `TargetPatternPhaseValue`.

7.  The loading/analysis phase is run to produce the action graph (a directed
    acyclic graph of commands that need to be executed for the build).

8.  The execution phase is run. This means running every action required to
    build the top-level targets that are requested are run.

## Command line options {:#command-options}

The command line options for a Bazel invocation are described in an
`OptionsParsingResult` object, which in turn contains a map from "option
classes" to the values of the options. An "option class" is a subclass of
`OptionsBase` and groups command line options together that are related to each
other. For example:

1.  Options related to a programming language (`CppOptions` or `JavaOptions`).
    These should be a subclass of `FragmentOptions` and are eventually wrapped
    into a `BuildOptions` object.
2.  Options related to the way Bazel executes actions (`ExecutionOptions`)

These options are designed to be consumed in the analysis phase and (either
through `RuleContext.getFragment()` in Java or `ctx.fragments` in Starlark).
Some of them (for example, whether to do C++ include scanning or not) are read
in the execution phase, but that always requires explicit plumbing since
`BuildConfiguration` is not available then. For more information, see the
section "Configurations".

**WARNING:** We like to pretend that `OptionsBase` instances are immutable and
use them that way (such as a part of `SkyKeys`). This is not the case and
modifying them is a really good way to break Bazel in subtle ways that are hard
to debug. Unfortunately, making them actually immutable is a large endeavor.
(Modifying a `FragmentOptions` immediately after construction before anyone else
gets a chance to keep a reference to it and before `equals()` or `hashCode()` is
called on it is okay.)

Bazel learns about option classes in the following ways:

1.  Some are hard-wired into Bazel (`CommonCommandOptions`)
2.  From the `@Command` annotation on each Bazel command
3.  From `ConfiguredRuleClassProvider` (these are command line options related
    to individual programming languages)
4.  Starlark rules can also define their own options (see
    [here](/extending/config))

Each option (excluding Starlark-defined options) is a member variable of a
`FragmentOptions` subclass that has the `@Option` annotation, which specifies
the name and the type of the command line option along with some help text.

The Java type of the value of a command line option is usually something simple
(a string, an integer, a Boolean, a label, etc.). However, we also support
options of more complicated types; in this case, the job of converting from the
command line string to the data type falls to an implementation of
`com.google.devtools.common.options.Converter`.

## The source tree, as seen by Bazel {:#source-tree}

Bazel is in the business of building software, which happens by reading and
interpreting the source code. The totality of the source code Bazel operates on
is called "the workspace" and it is structured into repositories, packages and
rules.

### Repositories {:#repositories}

A "repository" is a source tree on which a developer works; it usually
represents a single project. Bazel's ancestor, Blaze, operated on a monorepo,
that is, a single source tree that contains all source code used to run the build.
Bazel, in contrast, supports projects whose source code spans multiple
repositories. The repository from which Bazel is invoked is called the "main
repository", the others are called "external repositories".

A repository is marked by a repo boundary file (`MODULE.bazel`, `REPO.bazel`, or
in legacy contexts, `WORKSPACE` or `WORKSPACE.bazel`) in its root directory. The
main repo is the source tree where you're invoking Bazel from. External repos
are defined in various ways; see [external dependencies
overview](/external/overview) for more information.

Code of external repositories is symlinked or downloaded under
`$OUTPUT_BASE/external`.

When running the build, the whole source tree needs to be pieced together; this
is done by `SymlinkForest`, which symlinks every package in the main repository
to `$EXECROOT` and every external repository to either `$EXECROOT/external` or
`$EXECROOT/..`.

### Packages {:#packages}

Every repository is composed of packages, a collection of related files and
a specification of the dependencies. These are specified by a file called
`BUILD` or `BUILD.bazel`. If both exist, Bazel prefers `BUILD.bazel`; the reason
why `BUILD` files are still accepted is that Bazel's ancestor, Blaze, used this
file name. However, it turned out to be a commonly used path segment, especially
on Windows, where file names are case-insensitive.

Packages are independent of each other: changes to the `BUILD` file of a package
cannot cause other packages to change. The addition or removal of `BUILD` files
_can _change other packages, since recursive globs stop at package boundaries
and thus the presence of a `BUILD` file stops the recursion.

The evaluation of a `BUILD` file is called "package loading". It's implemented
in the class `PackageFactory`, works by calling the Starlark interpreter and
requires knowledge of the set of available rule classes. The result of package
loading is a `Package` object. It's mostly a map from a string (the name of a
target) to the target itself.

A large chunk of complexity during package loading is globbing: Bazel does not
require every source file to be explicitly listed and instead can run globs
(such as `glob(["**/*.java"])`). Unlike the shell, it supports recursive globs that
descend into subdirectories (but not into subpackages). This requires access to
the file system and since that can be slow, we implement all sorts of tricks to
make it run in parallel and as efficiently as possible.

Globbing is implemented in the following classes:

*   `LegacyGlobber`, a fast and blissfully Skyframe-unaware globber
*   `SkyframeHybridGlobber`, a version that uses Skyframe and reverts back to
    the legacy globber in order to avoid "Skyframe restarts" (described below)

The `Package` class itself contains some members that are exclusively used to
parse the "external" package (related to external dependencies) and which do not
make sense for real packages. This is
a design flaw because objects describing regular packages should not contain
fields that describe something else. These include:

*   The repository mappings
*   The registered toolchains
*   The registered execution platforms

Ideally, there would be more separation between parsing the "external" package
from parsing regular packages so that `Package` does not need to cater for the
needs of both. This is unfortunately difficult to do because the two are
intertwined quite deeply.

### Labels, Targets, and Rules {:#labels-targets-rules}

Packages are composed of targets, which have the following types:

1.  **Files:** things that are either the input or the output of the build. In
    Bazel parlance, we call them _artifacts_ (discussed elsewhere). Not all
    files created during the build are targets; it's common for an output of
    Bazel not to have an associated label.
2.  **Rules:** these describe steps to derive its outputs from its inputs. They
    are generally associated with a programming language (such as `cc_library`,
    `java_library` or `py_library`), but there are some language-agnostic ones
    (such as `genrule` or `filegroup`)
3.  **Package groups:** discussed in the [Visibility](#visibility) section.

The name of a target is called a _Label_. The syntax of labels is
`@repo//pac/kage:name`, where `repo` is the name of the repository the Label is
in, `pac/kage` is the directory its `BUILD` file is in and `name` is the path of
the file (if the label refers to a source file) relative to the directory of the
package. When referring to a target on the command line, some parts of the label
can be omitted:

1.  If the repository is omitted, the label is taken to be in the main
    repository.
2.  If the package part is omitted (such as `name` or `:name`), the label is taken
    to be in the package of the current working directory (relative paths
    containing uplevel references (..) are not allowed)

A kind of a rule (such as "C++ library") is called a "rule class". Rule classes may
be implemented either in Starlark (the `rule()` function) or in Java (so called
"native rules", type `RuleClass`). In the long term, every language-specific
rule will be implemented in Starlark, but some legacy rule families (such as Java
or C++) are still in Java for the time being.

Starlark rule classes need to be imported at the beginning of `BUILD` files
using the `load()` statement, whereas Java rule classes are "innately" known by
Bazel, by virtue of being registered with the `ConfiguredRuleClassProvider`.

Rule classes contain information such as:

1.  Its attributes (such as `srcs`, `deps`): their types, default values,
    constraints, etc.
2.  The configuration transitions and aspects attached to each attribute, if any
3.  The implementation of the rule
4.  The transitive info providers the rule "usually" creates

**Terminology note:** In the codebase, we often use "Rule" to mean the target
created by a rule class. But in Starlark and in user-facing documentation,
"Rule" should be used exclusively to refer to the rule class itself; the target
is just a "target". Also note that despite `RuleClass` having "class" in its
name, there is no Java inheritance relationship between a rule class and