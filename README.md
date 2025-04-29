# Philosophy

> _phi·los·o·phy_ (noun)
>
> 1. a theory or attitude held by a person or organization that acts as a
>    guiding principle for behavior
> 2. the study of the theoretical basis of a particular branch of knowledge or
>    experience
> 3. the pursuit of wisdom

Software engineering is a field awash with opinions, and it is easy to get swept
away with the tide in the absence of strong guiding principles to anchor
oneself. This document describes my approach to software engineering. It serves
as my anchor. It is colored by personal preferences and tempered by personal
experience. It covers both general principles as well as specific applications.
It is a living record and as such is bound to grow and mature as I do. It is
also hosted as a version-controlled repository, which means that you can see how
my thinking has changed over time, and how I reached this point.

It is meant primarily as a guiding force in my own practice, but if you find
yourself reading this (dear stranger), I encourage you to fork it, shape it to
your own needs, and draw value from it as well. If you think I should change my
approach to any or all of the subjects enumerated below, please make the case.
In mind, as in matter, pull requests are welcome.

## Code style

**Code should be drafted according to a consistent set of rules.**

These rules should be determined from the following sources in order of
priority: language, framework, team, and personal. Code without a style guide is
subject to constant change as developers take turns hacking at it in different
ways based on their level of experience, familiarity with a language, and
personal preferences. This forces the reader to throw away reasonable
assumptions about the structure and behavior of the code across files or
packages and in turn makes the process of reading code much slower. This
approach slows down code reviews as well as developers re-litigate the same
issues over and over, often accruing inconsistent conclusions over time. This
approach also squanders the “wisdom of the crowd,” which has been accumulated
and battle-tested over the course of decades in some cases, in favor of bespoke
solutions to solved problems.

**Style rules should be enforceable across heterogeneous development
environments.**

This often means committing rules to code. Style rules that cannot be enforced
across development environments will not be enforced. It is common to rely on
editors or IDEs to provide style guidance but depending on this sort of
integration as the sole enforcer forces developers to use a specific toolset
without an apparent benefit or alternatively accepts that style may not be
adhered to when developers use different tools. For instance, developers using
Visual Studio Code can use the Prettier plugin to format JavaScript code in a
Node.js package, but developers using another editor might not have access to an
analogous integration. Instead, the package should include the Prettier module
as a development dependency, and the formatter should be run as part of the
build process. This leads to consistent output across development environments.

**The type of a variable should be inferred when doing so would eliminate
redundant information.**

For instance, in the following example, the type of each declared variable is
apparent from the expression assigned to it, and the type annotation can be
omitted. The example also demonstrates how type inference can make code more
readable by lining up variable declarations instead of staggering them based on
the length of the declared type.

```java
// good
final var input = new InputSource(new StringReader(text));
final var documentFactory = DocumentBuilderFactory.newInstance();
final var documentBuilder = documentFactory.newDocumentBuilder();
final var document = documentBuilder.parse(input);
final var root = document.getDocumentElement();

// bad
final InputSource input = new InputSource(new StringReader(text));
final DocumentBuilderFactory documentFactory = DocumentBuilderFactory.newInstance();
final DocumentBuilder documentBuilder = documentFactory.newDocumentBuilder();
final Document document = documentBuilder.parse(input);
final Element root = document.getDocumentElement();
```

Explicit type annotations are typically required at logical boundaries such as
function definitions and class member declarations, so there are limited
opportunities for type inference to go wrong.

```rust
struct Point {
  x: i32,
  y: i32
}

fn transform(point: Point) {
  // x must be an &i32 because of the explicit
  // annotations on point and Point::x
  let x = &point.x;
}
```

In addition, type inference makes poorly named functions stand out, as in the
following example. If the return type is not obvious from the function name,
there is a fair chance that a better name exists.

```rust
let open = store_is_open(); // good
let open = check_store_status(); // bad
```

There are situations where explicit type annotations are appropriate, such as
when declaring an uninitialized variable or when the desired type is different
from the inferred type. When type inference is the default, an explicit type
annotation is a signal to the reader that the specified type is important for
the code that follows.

```java
final var composite = new Composite(); // inferred to Composite
final Widget composite = new Composite(); // narrowed to Widget
```

Most modern languages&mdash;Rust, Go, TypeScript, Kotlin, Scala&mdash;treat type
inference as the default for local variable declarations, and older languages
like
[C++](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#es11-use-auto-to-avoid-redundant-repetition-of-type-names)
and
[Python](https://mypy.readthedocs.io/en/stable/type_inference_and_annotations.html)
are converging toward this standard as well.

**Boolean conditions should be expressed in the subject-predicate form.**

Most languages follow this pattern with respect to the most primitive boolean
expressions through the use of infix operators. For instance, the expression
`car == "red"` reads naturally as "[subject] the car [predicate] is red." This
pattern can be extended to other boolean expressions to improve consistency.

Object methods that return boolean values should start with `is`, `has`, or
other sensible verbs. These methods naturally fall into the subject-predicate
form when accessed using dot notation.

```rust
list.is_empty()
response.has_body()
```

Freestanding variables and functions that return boolean values should start
with the subject instead. In most cases, starting such names with verbs leads to
awkward constructions that read like questions instead of propositions.

```rust
let car_is_running // good
let is_car_running // bad: reads like a question
fn cache_has_value() // good
fn has_cache_value() // bad: makes no grammatical sense
```

There is one exception to this rule. When the subject of the expression is
implicit, the name should start with a verb. In the following example, the
implied subject is the program itself.

```rust
fn should_credit_account() // good
fn account_should_be_credited() // bad: unnecessarily verbose
```

### Invariants

**Invariants that can be expressed using static types should be.**

This makes it possible to avoid an entire class of bugs resulting from
mismatched types at build time. For example, consider a value that represents
the seconds component of a timestamp. The value must be a number between 0 and
59 (this is the invariant). This value can be stored in an unsigned integer type
such as a `u8`, which can hold numbers between 0 and 255. However, because the
range of a `u8` is greater than the permissible range for this value, every
method that consumes this `u8` needs to confirm that the value is within the
expected range before using it. We can prevent invalid values from reaching
these methods by wrapping the primitive type in a new type (`Seconds`) that
validates the value on construction and passing that type to methods instead.
This puts the validation up front and allows consumers to simply depend on this
invariant when they have a `Seconds` object.

```rust
struct Seconds(u8);
impl Seconds {
  fn new(value: u8) -> Self {
    assert!(value < 60);
    Self(value)
  }
}

fn use_seconds(value: Seconds) {
  // this code can assume that the value is between 0 and 59
}
```

**Invariants that cannot be expressed using static types should be expressed
with runtime assertions.**

Some invariants cannot be enforced by the compiler. For example, the string
`^[[:alpha:]_][[:alnum:]_]*` represents a valid regular expression that matches
identifiers, but the compiler does not check this at build time. Even though the
developer knows that it is valid, the regex parsing method returns a result type
that needs to be unwrapped at runtime to access the compiled expression. A
runtime assertion in this case makes developer expectations clear and eases the
burden on code that depends on the validity of the regular expression. This
approach also makes it easier to catch programming errors early; the silent
violation of an invariant can lead to latent and unexpected behavior, but a
failed assertion (which often will crash the program) is loud, immediate, and
demands attention.

```rust
Regex::new("^[[:alpha:]_][[:alnum:]_]*").expect("hardcoded regex should be valid");
assert!(is_sorted(vec![1, 2, 3, 4, 5]), "input list must be sorted");
```

## Logging

**Logs should be emitted at the appropriate level.**

The following semantics are ascribed to each log level, in order of verbosity:

- _error_: Something went wrong, and the program cannot continue.
- _warn_: Something went wrong, and the program can continue.
- _info_: Something went right, and it is relevant to the end user.
- _debug_: Something went right, and it is not relevant to the end user.

There is no definition for _trace_ as it is redundant with debug. It may still
be used during development but should be compiled out of release builds. The
distinction between _info_ and _debug_ turns on the meaning of "end user" and
"relevance" in a given context; when in doubt, it is better to have more
information than less, so bias towards _info_.

### Error handling

**Error messages should describe what actually happened.**

The alternate approach&mdash;describing what should have happened&mdash;forces
the reader to do more work to determine what went wrong. For instance, if a file
read operation returns an error that says "the file should exist" (what should
have happened), the reader has to take an additional logical step to conclude
that the file does not exist (what actually happened). In addition, the
alternate approach is less flexible when composed with its surrounding context.
It may be reasonable (and even expected) in a given application for the file to
not exist, in which case saying that the file does not exist remains accurate
but saying that it should exist becomes inaccurate.

```rust
Err("the file should exist") // bad: assumes the "right" outcome
Err("the file does not exist") // good
```

**Error messages should describe the step that failed instead of the process it
is a part of.**

This results in more terse and direct error messages and reduces duplication.
When a method returns an error, it is implied that the method failed. Including
that information in the error message would be redundant.

```rust
fn send_request(...) -> Result<(), Error> {
  ...
  // bad: failure to send request is implied
  return Err("failed to send request, unable to reach host");
  // good: direct and not redundant
  return Err("unable to reach host");
}
```

**Methods that can fail should document this possibility.**

In a language like Rust that makes error handling ergonomic, this can be
expressed by using a `Result` as the return type in a function signature. In a
language like Java that makes error handling more cumbersome (and checked
exceptions unappealing) it may be sufficient to simply make note of the failure
conditions in a doc comment. This signals to a caller that they should expect
failures from a given method and gives them an opportunity to respond to it.
Even if a caller simply propagates the error, they are at least doing so
intentionally. This also allows developers to be more confident when using
functions that are marked as infallible as they do not need to wrap calls to
those methods with error handling logic.

```rust
fn fallible_function() -> anyhow::Result<()>
```

### Code formatting

**Code should be formatted according to a consistent set of rules.**

This includes source code as well as tests, configuration files, READMEs, and
all other content that is susceptible to formatting. Developers often have
strong personal preferences about how something should be formatted, and a lack
of standardization opens the door to wasted time and effort discussing
formatting concerns during code review. In addition, poorly formatted code
(e.g., long lines, crowded statements) can make the code harder to read and
understand.

**JavaScript, TypeScript, HTML, React, and JSON code should be formatted
according to the Prettier rules.**

Prettier acts as an all-in-one formatter for these web languages which obviates
the need to configure multiple formatters for a single frontend application. It
also supports shared configuration files which makes it possible to define
configuration once and use it everywhere. It is opinionated and has a philosophy
that discourages options in favor of uniformity. Not everyone will agree with
all of the formatting decisions that Prettier makes, but the lack of options
reduces the chance that the style will be re-litigated every cycle.

**Python code should be formatted according to the Black rules.**

Black is safe insofar as it confirms that the source abstract syntax tree
matches before and after formatting. This is especially material in the context
of Python, where small deviations in whitespace can change the behavior of the
code. It is also opinionated, with limited options for configuration, and thus
reduces the need for repeated discussion around configuration.

## Testing

### Unit tests

**Unit tests should be run as part of the build process.**

There is a natural tendency for developers to take the path of least resistance
in order to solve a problem. The more friction there is with respect to running
tests, the more likely it is that those tests will not be run. Running tests as
part of the build process obviates the decision for the developer. This practice
is already folded into some build systems such as Maven, which will run tests
before packaging, though it requires manual setup with others such as npm.

**Unit test names should describe inputs, outputs, and expected behavior.**

This format makes clear which inputs and outputs are being tested for each
function. This is also easier to understand at a glance than the full test
implementation. This format also makes it easier to identify gaps in testing
based on the behaviors that are enumerated.

**Unit tests should succeed independent of the environment.**

Unit tests are meant to test individual units of code independent of the
environment in which they execute. This is distinct from integration tests which
are used to see how the units work together in a specific environment. In
practice, this means that unit tests should be limited to testing things that
are in-process, i.e., in memory. Behavior that depends on out-process resources
such as environment variables, filesystems, databases, and networks should be
reserved for integration testing or mocked for unit tests.

**Unit tests should succeed independent of the result of other unit tests.**

This makes it possible for test runners to run tests in parallel, speeding up
iteration time. It also makes tests easier to understand, as all inputs to a
given test are apparent on its face.

**Unit tests should be deterministic.**

Units tests should return the same result for the same input on each run
assuming no changes in between runs. Brittle tests that succeed only part of the
time reduce confidence in the test suite, increase package onboarding friction,
and increase the likelihood that tests will be skipped.

**Unit tests should contain the least amount of code necessary to pass.**

It is more difficult to reason about a test that includes more information than
is required to pass the test such as extra properties or non-zero values that
are not under test. Minimal tests are also more resilient to future changes as
there is less surface area that might be affected by a change.

**Unit tests should avoid logical conditions such as if, while, for, and
switch.**

A unit test should cover only a single logical branch of behavior. This makes
the tests more straightforward to write and also makes the error conditions
easier to isolate in case of failure.

## Version control

### Branch management

**Code that is deployed to production should exist on a single branch.**

Depending on the setting, this branch might be called `main`, `mainline`, or
`master`, but there should be only one. Sticking to a standard branch name
reduces the time needed to discover production code and also makes it easier to
write tooling that depends on knowing the default branch of a package.

**Code in feature branches should be merged directly into the main branch.**

This approach is sometimes referred to as [trunk-based
development](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development)
or [GitHub
Flow](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development).
It fits well with the continuous delivery model in which code is deployed as
soon as it is merged. It also reduces the mental overhead of managing multiple
intermediary branches like `dev` and `release-candidate` as proposed in
alternate approaches such as
[GitFlow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow).

**Branches that have been merged into the main branch should be deleted.**

The longer a branch is left around, the more difficult it becomes to determine
if the branch is needed. This determination is easiest to make when a branch is
merged into the main branch; at this point, the branch is no longer necessary.
Dead branches increase the clutter in a package and in turn increase the time
required to find relevant code.

### Ignored files

**The _.gitignore_ file for a repository should only be used to exclude files
and directories specific to that repository.**

This includes local dependency caches like _node_modules_ and build artifacts
like _target_, _build_, and _cdk.out_. Files that should be ignored for all
repositories, such as those created by particular editors (_.vscode_, _.idea_),
operating systems (_.DS_Store_), or command line utilities (_mutagen.yml_),
should be excluded through a global _.gitignore_ file instead. This spares the
developer the effort of repeating common ignore clauses for each new repository.
It also makes for more focused repository-specific ignore files, which otherwise
might be polluted with ignore clauses tied to specific developer setups.

### Commit management

**Published commits should be atomic.**

Commits that focus on a single task are easier to understand than commits that
cover multiple tasks. Atomic commits also provide more precision when rolling
back. In addition, atomic commits help readers narrow the range of relevant
commits when searching for bugs. In practice, a good rule of thumb is that if
the commit body contains a statement that is not reasonably related to the
commit subject line, the commit should be split up further.

**Published commits should be stable.**

This means that every commit should build and pass tests. Once published, it is
reasonable to expect that a developer may need to jump back to an arbitrary
revision in the history at some point, either to study or to revert to that
revision. If the project no longer builds or passes tests at that revision, the
developer is forced to solve two problems (figure out how to build and the
original issue) instead of one problem (the original issue). This standard
accelerates development velocity as well, allowing developers to make quick
deployments with the confidence that earlier revisions are safe to revert to at
need.

**The subject line for a commit should be written using the imperative mood.**

Each commit represents some transformation on the code that came before it,
i.e., a delta or a diff to be applied, and using the imperative mood matches
this understanding. The imperative mood also maps cleanly onto tracking issues,
as issues are likely to be written in the imperative as well; for instance, it
is more likely to see an issue called “fix bug” than one called “fixed bug.”
Further, Git defaults to the imperative for commits that it generates, i.e.,
“merge” instead of “merged" and "rebase" instead of "rebased."

```shell
feat: add red button # good
feat: added red button # bad
```

**The subject line for a commit should include a type as specified by the
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) standard
and as supplemented by the [Angular
convention](https://github.com/angular/angular/blob/22b96b9/CONTRIBUTING.md#type).**

Thinking in terms of commit types (fix, feat, docs, style, etc.) helps a
developer push for atomic commits and break up work into logical units. To the
extent that semantic versioning is used for a package, this convention makes it
easier to determine the appropriate version bump for a given change. This
convention also makes it easier for developers to sift through commit history
using a consistent vocabulary for the same type of change (e.g., “feat” instead
of “add,” “introduce,” or “implement”). Further, this convention is lightweight
and imposes no adoption or abandonment cost.

**The subject line for a commit should not be longer than 50 characters.**

Shorter subject lines are more readable. This soft limit also forces a developer
to think through the most concise way to explain the commit. In addition, this
provides a developer another opportunity to push for atomic commits; if a commit
is not amenable to a short subject line, it may be a sign that the commit should
be split up further. Further, commit messages should be readable from the
command line, and Git does not wrap commit messages automatically.

**The body for a commit should wrap lines at 72 characters.**

Commit messages should be readable from the command line, and Git does not wrap
commit messages automatically. Lines that extend beyond this limit force the
reader to use code editors or other graphical tools to make sense of commit
messages.

**The body for a commit should focus on the nature and purpose of a commit
rather than its implementation.**

The implementation of a change is apparent from the code itself, but the
intention of the change is not. For instance, consider a change that makes a
button blue instead of red. This change is apparent on the face of the code, but
it is not obvious that the change was made because customers were more likely to
click the blue button than the red one. This context helps the reader understand
how the code came to be in its present state and also helps them determine
whether the assumptions that underlie the present state still hold (e.g., red
buttons are back in vogue). Exposition on the nature of the commit is also
useful to point out a whole that may not be apparent from its parts.

**The footer for a commit should include a link to the tracking issue that
relates to that commit, if applicable.**

A commit message is fixed in time. It is also unilateral in that it is written
by one person. Issue tracking software is better situated to provide dynamic
context in the form of dialogue and associated media. Linking to the tracking
issue in a commit message allows the reader to see the ongoing justification for
the change alongside its original motivation.

## Code review

**A code change should be reviewed and approved before being deployed.**

There is often pressure to move on to the next thing once code has been
deployed, and code improvements that are slated for the future fall to the
wayside in the face of more exigent concerns. In other words, it is reasonable
to expect that code will not change after it has been deployed. Thus, it is
important to get code right on the first pass; code review facilitates this
goal.

**A code change should be reviewed within one business day of publication.**

The sooner a developer receives feedback regarding a code change, the more
likely it is that they will be able to recall the original development context
and provide meaningful responses to the feedback. In addition, when a code
change lingers, a developer may be forced to pick up other tasks in the interim,
which results in more context-switching and hinders productivity.

## Deployment

**Code should be distributed with as many of its dependencies as is practical.**

Applications often depend on other libraries and tools to perform their tasks,
and, as a general matter, these dependencies can come from one of two places:
the distribution or the host environment. Including dependencies with the
distribution increases the distribution size, but results in more predictable
behavior, while sourcing dependencies from the environment typically has the
inverse effect. In a world where disk space is cheap, the benefits of shared
dependencies do not outweigh their costs.

In practice, this means using statically linked libraries instead of dynamically
linked libraries and bundling binaries and other tools, where it makes sense.
Some dependencies will rarely make sense to bundle, such as operating system
libraries, as those can be expected to be present in the host environment and
would in any event be prohibitively expensive to include in a distribution. For
others, the calculus will change from one application to the next. For instance,
Python source code will typically take up 5-10 MB, but an interpreter will take
up 200-300 MB, and it may or may not be worth it in a given situation to include
the interpreter depending on what assumptions can be made about the host
environment.

## Continuous deployment

**Code changes should be deployed within one business day of approval.**

The longer the gap between the approval and deployment, the more chance there is
that an intervening change will cause the code to break. Code is written and
reviewed against a set of assumptions about what already exists in the
deployment environment, and those assumptions become progressively less sound as
the deployment gap widens. In addition, a longer deploy gap increases the chance
that an intervening change will be merged, potentially prompting a rebase and in
turn additional code review.

**Code changes should be deployed through a pipeline.**

Manual deployment consumes developer attention and bandwidth. A pipeline takes
over this task and empowers developers to focus on building features instead of
worrying about how those features are ultimately delivered to customers. A
pipeline also allows the team to enforce standards before code reaches
production, e.g., code review approval and integration test success.

**Pipelines should contain at least one pre-production stage.**

A pre-production stage&mdash;alpha, beta, gamma, test, staging&mdash;provides an
opportunity to run integration tests and other verification of code changes
before those changes make it to production.

**There should be a separate mechanism to deploy code to a sandbox environment.**

It is not always possible to test specific behavior using local resources;
instead, it may be necessary to test against deployed resources. In the absence
of a sandbox environment, developers are forced to deploy changes to a pipeline
stage to see how they behave; this can disrupt the pipeline mechanism,
potentially blocking other changes from being deployed to production. A sandbox
environment allows developers to test such changes in a safe manner and also
reduces iteration time as variations can be deployed without waiting on code
review or pipeline promotions.

**The pipeline platform and code should be versioned to the extent possible.**

This includes the operating system, binaries, libraries, and other artifacts
that are amenable to versioning. It also includes things like Docker images and
GitHub actions which are often (but should not be) treated as exempt from
standard version bumping procedure. This increases the cost of keeping
dependencies up to date as there is manual effort required, but this aligns more
closely with the practice of introducing changes to a system deliberately and
knowingly. The tradeoff is a higher risk of unaddressed security
vulnerabilities, which can be mitigated using tools like Dependabot that
identify and surface such issues.

## Package documentation

**Each package should contain a README in its root directory.**

The README serves as the entry point for each package and provides the best
venue to document the purpose, composition, and development requirements for a
package. This is the documentation that is most likely to evolve in step with
the package because of its proximity to the code that it covers. There is also
strong tooling support for this file; for instance, GitHub displays this file
automatically on the splash page for a given package.

**The README for a package should contain an overview section that explains the
purpose and contents of the package.**

There is a limit to how much information can be conveyed in a package name, and
developers often need to sift through several packages to find the right one.
This task becomes much more time-consuming when the purpose of a package is not
apparent on its face. In these situations, developers must work backwards from
the source code to determine the purpose of a package, often across a number of
frameworks, languages, and toolchains that may or may not be familiar. The
overview section of a README short-circuits this process, making it easier for
developers to determine whether to read further into the package or move on.

**The README for a package should contain a tooling section that explains the
tooling needed to develop in the package.**

Developer environments can differ in subtle ways from each other and from
production environments, and the tooling that a package may work with is not
necessarily the same as the tooling it is intended to work with. Pinning a
package to a specific set of tools and versions reduces the time needed to be
productive in that package and sidesteps headaches caused by environmental
inconsistencies. For instance, the significance of some files like
_package.json_ or _pom.xml_ may be obvious to developers with experience using
the related tools (npm and Maven, respectively) but not for those that are
unfamiliar. Similarly, it makes it easier to identify packages that are
outdated. For example, a package that advertises itself as using Python 3.6 will
stand out as a candidate for upgrade since that version is marked as
end-of-life. In the absence of this indication, a reader may assume based just
on the code that it is used with a newer version of Python.

**The README for a package should contain a build section that explains the
steps needed to build the package.**

The commands (and prerequisites) needed to build a package vary widely across
tools, frameworks, and languages. This is further complicated when working with
internal build systems that may not be familiar to developers used to idiomatic
open-source build systems.

**The README for a package should contain a test section that explains the steps
needed to test the package.**

If test coverage is supported, the section should also explain how to generate
and review test coverage. Successfully being able to run tests in a new package
often serves as a baseline to determine that the current state of a package is
stable and further serves to confirm that changes to a package do not break
existing functionality. The harder it is to run tests, the more likely it is
that developers will simply skip this phase and test manually or test on deploy
(i.e., trial by fire).

**The README for a package should contain a run section that explains the steps
needed to run the package.**

In conjunction with tests, the ability to run code locally can materially reduce
iteration time. This is also essential for testing behavior that must be tested
manually, either because there are no automated tests for that behavior yet or
because such automation is impossible or impracticable. It is also sometimes
necessary to support local development needs for other packages (i.e., testing
local frontend changes against local backend changes).

## Command line interfaces

**Disparate actions should be expressed as subcommands instead of as flags.**

The pattern of using flags for everything makes sense for true UNIX-style
utilities which do one and only one thing. However, modern command line
utilities often do many things at once in a common domain, and using flags to
express those distinct behaviors makes it easy to confuse action and
configuration. In the following example, it is not obvious whether the primary
action is `get-item` or `return-consumed-package` (or perhaps both):

```shell
aws dynamodb --get-item --return-consumed-capacity [options...]
```

The next example, which uses subcommands and matches the actual implementation
of the AWS CLI, makes it evident that `get-item` is the primary action and that
`return-consumed-package` is only a modifier:

```shell
aws dynamodb get-item --return-consumed-capacity [options...]
```

This pattern is widespread in modern command line tooling (e.g., `docker run`,
`git commit`, `cargo install`).

**Substantive output should be written to standard output.**

This is content that represents the result of the command such as the DNS
records located using `dig` or the matches found in a stream of text by `grep`.
This makes it easy to route the output through pipes or process it in other ways
without having to differentiate it from logs.

**Logs should be written to standard error instead of to file.**

This allows users to view log messages directly in the terminal (compare this to
the tedium associated with inspecting a FindBugs report). In the general case,
command line logs do not need to be persisted beyond the current session.
Instead of littering the working directory with log files, this approach allows
users to opt in to persistence by redirecting logs to a file as needed.

## Pre-production environments

**Pre-production environments should depend on production services.**

The primary purpose of a pre-production environment is to reduce the risk of
introducing bugs to production. The closer a pre-production environment is to
the production environment&mdash;sometimes called its fidelity&mdash;the more
confidence it provides that something that works in the lesser environment will
work in production. To this end, pre-production tests should run against
production dependencies, for a few reasons:

- Integration behavior often gets out of sync with production behavior. This
  reduces environment fidelity.
- Integration endpoints are often less reliable than their production
  counterparts and may not have the same support guarantees.
- Many services do not offer integration endpoints at all. This forces the
  developer to use production endpoints for some dependencies and integrations
  for others, which can make it harder to understand the dependency graph for a
  pre-production environment.

Developers often avoid taking production dependencies in pre-production
environments for fear of disrupting production operations or cluttering
production state. These issues can be mitigated using the same techniques
employed to prevent inappropriate traffic from other clients. For instance, a
pre-production environment that integrates with Slack can post messages to a
throwaway Slack channel using a token that only has write permissions on that
channel and a limited throughput allowance. This environment can test
interactions with the production Slack service without affecting production
state or health.

## Application programming interfaces

**Application programming interfaces ("APIs") should only use HTTP as a
transport protocol.**

HTTP is advertised as an application layer protocol, but in practice it is a
poor tool for modeling application logic, for a number of reasons:

- _The protocol blurs the line between message transmission and substance._ It
  offers methods (e.g., `CONNECT`, `OPTIONS`, and `TRACE`), status codes (e.g.,
  `3xx`, `502`, `504`), and headers (e.g., `Connection`, `Forwarded`,
  `Access-Control-Allow-Origin`) that address routing, redirection, proxy
  servers, cross-origin access, and other matters that have no bearing on
  application logic.
- _The protocol was not designed for service-to-service communication._ It was
  initially designed to make it easy to request, serve, and render arbitrary
  content in the context of a browser, and that legacy continues to play a
  significant role in the features it provides. For instance, the `Content-Type:
application/json` header helps a browser render a response from an arbitrary
  server in a reasonable manner. The same header means little for most backend
  applications, which support exactly one serialization format (typically JSON),
  and their clients, which are written to work with a single format.
- _The protocol is ambiguous with respect to application logic._ Over the years,
  different parties have layered different (often conflicting) semantics on the
  behavior and interpretation of urls, methods, status codes, headers, query
  strings, and bodies, such that the current state of affairs is one of complete
  disarray. Basic questions like the meaning of a `404` status code (does it
  mean that the endpoint is not supported or that the object at the endpoint
  does not exist) remain the subject of debate. The REST approach, perhaps the
  most popular semantic interpretation of HTTP, is afflicted by the same
  ambiguity. There is no consensus on what it means to be RESTful, and discourse
  frequently devolves into theories about "what Fielding meant" and the promise
  of HATEOS (which has yet to materialize outside the browser). There is limited
  value in adhering to a standard that no one agrees on.
- _The protocol lacks the granularity needed for application logic._ For
  instance, it forces the developer to map every write operation onto one of
  three generic verbs&mdash;`POST`, `PUT`, or `PATCH`; this a step backwards in
  terms of expressive power when compared to the naming flexibility afforded to
  the same operations executed in-memory (compare `PATCH /pause` to
  `pause_application()`). This is also a limiting factor when it comes to error
  handling. For instance, the `400` status code indicates a bad request, but it
  says nothing about what made the request bad (e.g., malformed JSON, a missing
  field). In such cases, which are common, one standard workaround is to put a
  more fine-grained error code in the response body. However, at that point, the
  status code is redundant, which prompts one to question whether it should be
  included at all.

In light of the foregoing, HTTP should be used for transmitting messages from
one place to another and no more. Application logic should be modeled using
domain-specific request and response bodies. This allows both clients and
servers to find all of the information needed to make business decisions in one
place and ensures that the information is narrowly tailored to the requirements
of the domain. This pattern can be found in the wild, in services like AWS and
Slack. Services based on gRPC follow this pattern as well.
