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
