# Ports Options

**Note: this is the feature as it was initially specified and does not necessarily reflect the current behavior.**

## Motivation

There are a number of ports that require fine-grained configuration, and for which the only current options are abusing features, or requiring most users to make an overlay port only to change a few variables. Both those options are suboptimal, which is what this RFC aims to fix.

### Limitations of features

Conceptually, features are sets of named boolean flags. It has some consequences making them unsuited to some use-cases when it comes to fine-grained configuration:

* Features are boolean options: either a feature is enabled or it isn't. It makes it hard to represent options which can have more than two values.
* Features are additive. If a library depends on both `port[a]` and `port[b]`, those will get merged into `port[a, b]`. This is important because not merging those ports will more often than not result in ODR violations. This might not generally be possible with some mutually exclusive options.

### Limitations of overlays

The most glaring problem of requiring users to use overlays to configure ports is that it creates duplication of almost entire portfiles, only to set a few variables. This exposes what should be implementation details to the user, by asking them to edit internals of the portfile.

It also prevents user from getting future versions of the portfile, unless the manually keep their overlay in sync. This is brittle, and ideally we'd like users to not have to touch the portfile at all.

## Design concerns

### Textual representation

Like features and triplets, we probably want a compact form to express a port with some options, if only to be able to talk about them.

We propose extending the `name[features...]:triplet` form into `name[features...]{options...}:triplet`. For example, the port `some-port` with the options `a` set to `"value-a"` and option `b` set to `"value-b"` would be spelled `some-port{a:"value-a", b:"value-b"}`.

Options not provided this way use their default value.

One thing to note is that braces with commas inside them will be interpreted by various shells, so if this is ever used on a command line, it will requires user to quote it, or they'll get surprising results.

### Types of options

Since we expect users to provide options through json, that gives a few possible types that options could have. If we remove compound types and `null`, it leaves strings, numbers, and booleans, which all sound somewhat reasonable.

Whether it make sense for an option to be a "free field" is also an open question. It's expected that the majority of ports that want to provide options will only allow providing one of a set of known values. Interestingly, if we only allow finite set of values, then numbers can be provided as strings without any substantial loss in ergonomics.

For a first implementation, we feel like allowing only strings and booleans from a specified set of possible values covers most usages, and doesn't prevent extending the set of allowed options in the future.

It's also important that all options end up with a value. It should never be possible for an option to have a "not-provided" value that cannot be explicitly set by the user. We can either mandate all options to have a default value, or mandate that the user provide a value for options without a default one.

### To merge or not to merge ports with different options

For the sake of example, we'll assume a port `p`, with two options `a` and `b`. The possible values for option `a` are `"a1"`, `"a2"`, and `"a3"`, and the default one is `"a1"`. The possible options for option `b` are `"b1"`, `"b2"`, and `"b3"`, and the default one is `"b1"`.

One question we have to answer is "What is the relationship between `p{a="a1"}` and `p{a="a2"}`. We propose treating them the same way we'd currently treat triplets. That is, conceptually, each base triplet + option set would be turned into one special triplet. This way, `p{a="a1"}:x64-linux` would be conceptually equivalent to using some `p:options-a-a1-b-b1-triplet-x64-linux` very precise triplet.

The consequences mergin triplets and options semantics are yet to be explored, but we hope that it will make options more easily teachable.
