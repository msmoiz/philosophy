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
