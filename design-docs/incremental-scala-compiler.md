Due to the power and complexity of the language, Scala compilation is slow, and will stay slow in the future.

We want to speed up Scala compilation by:

 * Integrating with sbt's/Zinc's incremental Scala compiler
   Based upon the official Scala compiler, this compiler only recompiles code that has changed or is affected by a change.
   Incremental compilation is particularly interesting for developer builds and builds with few source changes (e.g. CI commit builds).
   It it known to lead to increased compilation times for full builds.

 * Reusing the Scala compiler between compilation tasks and Gradle builds
   Keeping the compiler code warm has shown to lead to big improvements in compilation time.

[Zinc and Incremental Compilation](http://blog.typesafe.com/zinc-and-incremental-compilation) has more information on the sbt/Zinc incremental compiler.
Zinc is a wrapper for sbt's incremental compiler that provides a scalac-like command-line compiler interface. As an option, it can keep the compiler
running in a daemon process (similar to fsc). Zinc also offers a compiler API, which might be easier to integrate with than sbt
and/or might have a more stable API (investigation pending). The Zinc API appears to be a fairly small wrapper around the sbt API,
and some types appear to leak through. Recent versions of the [scala-maven-plugin](https://github.com/davidB/scala-maven-plugin) exclusively use Zinc.

# Implementation plan

## Make ScalaCompile task support sbt's incremental compiler

Figure out whether to integrate directly with sbt or go via Zinc. In both cases, we will integrate via a compiler API, and no external process will be involved.

### User visible changes

New switch to enable the incremental compiler: ScalaCompileOptions.useIncrementalCompiler = true|false.

### Sad day cases

If for some reason the incremental compiler can't compile some code, user can always fall back to the regular compiler.

### Test coverage

All existing Scala compilation tests should keep working when turning on incremental compilation.

### Implementation approach

Implement a new Compiler<ScalaCompileSpec> that leverages the incremental compiler.
Use scala-maven-plugin as an example for how to use the incremental compiler API.

## Integrate ScalaCompile task with Gradle compiler daemon

This allows compilation to be performed in an external (Gradle compiler daemon) process. That process also caches the Scala compiler between compile tasks.

### User Visible Changes

Introduce ScalaCompileOptions.forkOptions (similar to Java and Groovy case). Forked compilation will always use the Gradle compiler daemon.

### Sad day cases

If for some reason the forked compiler doesn't work in some cases, user can always fall back to in-process compilation.

### Test coverage

All existing Scala compilation tests should keep working when turning on forked compilation. Add additional tests for compilation
with fork options and selection of compatible compiler daemon.

### Implementation approach

Implement a new DaemonScalaCompiler that leverages the Gradle compiler daemon. Might require some enhancements to the compiler daemon
so that it can cache class loader(s) that load the Scala compiler. Or maybe we get that automatically by starting the daemon with
a "fully" loaded class path. Might also require some enhancements with regard to selecting an appropriate daemon.

## Reuse Scala compiler between Gradle builds

Options are to extend our own compiler daemon so that it can stay alive and get reused between builds, or to integrate with the Zinc daemon.

### User visible changes

Probably needs some ways to configure the compiler daemon (how long does it stay, how many daemons at a time, etc.)

### Sad day cases

Can always go back to a per-build daemon if necessary.

### Test coverage

All existing Scala compilation tests should keep working when compiler daemon gets reused between builds.

### Implementation approach

Conceivable approaches:

 * Create common daemon infrastructure used by Gradle Daemon and compiler daemon
 * Create daemon infrastructure separate from Gradle Daemon that can be used whenever a Gradle build needs to run an activity in a different process
 * Implement a compiler daemon specific solution

# Open issues

Probably a good idea to deprecate fsc integration (ScalaCompileOptions.useCompileDaemon) as we go. From what I know, it never worked that well anyway,
especially for multi-project builds.

When performing Scala/Java joint compilation, Zinc's compiler apparently not only reads Java sources but also compiles them. Can we avoid that? If not,
can we at least reroute Java compilation to use our own Java compiler integration (like we do for Groovy)? Is this behavior specific to Zinc,
or does it also apply to sbt's incremental compiler?

The incremental compiler stores some metadata on disk. When incremental compilation is flipped on and off on successive compilations, can this lead to
incorrect compilation results, or does it, in the worst case, lead to more files being recompiled than necessary?

Judging from my experiments, sbt/Zinc not only require the scala-library Jar (7MB) on their own class path, but also the scala-compiler Jar (15MB).
This is although they can be configured with the Scala compiler (Jar) to be used for compilation. Since we can't ship such big Jars with the Gradle
distribution, questions arise around how to load sbt/Zinc dynamically, which versions to use at runtime (vs. compile time), etc.