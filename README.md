# Philosophy

> *phi·los·o·phy* (noun)
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

**The type of a variable should be inferred when doing so would eliminate
redundant information.** For instance, in the following example, the type of
each declared variable is apparent from the expression assigned to it, and the
type annotation can be omitted. The example also demonstrates how type inference
can make code more readable by lining up variable declarations instead of
staggering them based on the length of the declared type.

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

**Branches that have been merged into the main branch should be deleted.** The
longer a branch is left around, the more difficult it becomes to determine if
the branch is needed. This determination is easiest to make when a branch is
merged into the main branch; at this point, the branch is no longer necessary.
Dead branches increase the clutter in a package and in turn increase the time
required to find relevant code.

### Commit management

**Each published commit should be atomic.** Commits that focus on a single task
are easier to understand than commits that cover multiple tasks. Atomic commits
also provide more precision when rolling back. In addition, atomic commits help
readers narrow the range of relevant commits when searching for bugs. In
practice, a good rule of thumb is that if the commit body contains a statement
that is not reasonably related to the commit subject line, the commit should be
split up further.

**Each published commit should be stable.** This means that every commit should
build and pass tests. Once published, it is reasonable to expect that a
developer may need to jump back to an arbitrary revision in the history at some
point, either to study or to revert to that revision. If the project no longer
builds or passes tests at that revision, the developer is forced to solve two
problems (figure out how to build and the original issue) instead of one problem
(the original issue). This standard accelerates development velocity as well,
allowing developers to make quick deployments with the confidence that earlier
revisions are safe to revert to at need.

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

**The body for a commit should wrap lines at 72 characters.** Commit messages
should be readable from the command line, and Git does not wrap commit messages
automatically. Lines that extend beyond this limit force the reader to use code
editors or other graphical tools to make sense of commit messages.

**The body for a commit should focus on the nature and purpose of a commit
rather than its implementation.** The implementation of a change is apparent
from the code itself, but the intention of the change is not. For instance,
consider a change that makes a button blue instead of red. This change is
apparent on the face of the code, but it is not obvious that the change was made
because customers were more likely to click the blue button than the red one.
This context helps the reader understand how the code came to be in its present
state and also helps them determine whether the assumptions that underlie the
present state still hold (e.g., red buttons are back in vogue). Exposition on
the nature of the commit is also useful to point out a whole that may not be
apparent from its parts.

**The footer for a commit should include a link to the tracking issue that
relates to that commit, if applicable.** A commit message is fixed in time. It
is also unilateral in that it is written by one person. Issue tracking software
is better situated to provide dynamic context in the form of dialogue and
associated media. Linking to the tracking issue in a commit message allows the
reader to see the ongoing justification for the change alongside its original
motivation.

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

## Pre-production environments

**Pre-production environments should depend on production services.** The
primary purpose of a pre-production environment is to reduce the risk of
introducing bugs to production. The closer a pre-production environment is to
the production environment&mdash;sometimes called its fidelity&mdash;the more
confidence it provides that something that works in the lesser environment will
work in production. To this end, pre-production tests should run against
production dependencies, for a few reasons:

* Integration behavior often gets out of sync with production behavior. This
  reduces environment fidelity.
* Integration endpoints are often less reliable than their production
  counterparts and may not have the same support guarantees.
* Many services do not offer integration endpoints at all. This forces the
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
transport protocol.** HTTP is advertised as an application layer protocol, but
in practice it is a poor tool for modeling application logic, for a number of
reasons:

* *The protocol blurs the line between message transmission and substance.* It
  offers methods (e.g., `CONNECT`, `OPTIONS`, and `TRACE`), status codes (e.g.,
  `3xx`, `502`, `504`), and headers (e.g., `Connection`, `Forwarded`,
  `Access-Control-Allow-Origin`) that address routing, redirection, proxy
  servers, cross-origin access, and other matters that have no bearing on
  application logic.
* *The protocol was not designed for service-to-service communication.* It was
  initially designed to make it easy to request, serve, and render arbitrary
  content in the context of a browser, and that legacy continues to play a
  significant role in the features it provides. For instance, the `Content-Type:
  application/json` header helps a browser render a response from an arbitrary
  server in a reasonable manner. The same header means little for most backend
  applications, which support exactly one serialization format (typically JSON),
  and their clients, which are written to work with a single format.
* *The protocol is ambiguous with respect to application logic.* Over the years,
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
* *The protocol lacks the granularity needed for application logic.* For
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
