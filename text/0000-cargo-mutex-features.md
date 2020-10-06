- Feature Name: `cargo-mutex-features`
- Start Date: 2020-10-06
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/cargo#2980](https://github.com/rust-lang/cargo/issues/2980)

**NOTE**: This is **work in progress**!

Based on the idea from: https://github.com/rust-lang/cargo/issues/2980#issuecomment-700687022

# Summary
[summary]: #summary

This RFC proposes a way to implement "mutually exclusive" features in Cargo, by introducing the concept of
"capabilities" of a feature.

# Motivation
[motivation]: #motivation

Cargo currently supports features, which allow one to use conditional compilation. A common use case is to
provide some kind of capability, backed by different implementations, based on the feature selected by the user.

In group of cases, these implementations are mutually exclusive. While custom `build.rs` scripts can be used to
implement this behavior partly, it is not a great experience. Neither from the implementor's side, nor from the
user's side.

## Examples

One example is the crate [`k8s-openapi`](https://github.com/Arnavion/k8s-openapi). The crate compiles for different
Kubernetes versions. However, at most one API version may be active.

To achieve this, the [Cargo.toml](https://github.com/Arnavion/k8s-openapi/blob/master/Cargo.toml) currently contains
features (`v1_11` to `v1_19`), and has a [build.rs](https://github.com/Arnavion/k8s-openapi/blob/master/build.rs),
which uses custom code to validate this.

Another example is the crate [`stm32f7xx-hal`](https://github.com/stm32-rs/stm32f7xx-hal/) (and its sibling crates),
which require exactly one hardware configuration to be enabled. Again, the `Cargo.toml` contains a list of features,
one for each hardware configuration. It also introduces a meta feature named `device-selected`, which gets set to
simplify checks in the code if a platform was selected. 

It would also make sense to have this feature on a build tree (global) level. A common example in the embedded space
is the "panic handler" or "allocator", which must be selected, but unique on the whole application. Currently, people
simple comment out alternatives (https://github.com/rust-embedded/cortex-m-quickstart/blob/master/Cargo.toml).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In addition to an identifier, features can specify one or more capabilities that they provide or require, when the
feature is enabled.

## Basic example

The following example shows the basic idea:

~~~toml
[features]
json = []
yaml = []
json_foo = [ "foo_json" ]
json_bar = [ "bar_json" ]

[features.json]
requires = ["json_backend"]

[features.json_foo]
provides = ["json_backend"]

[features.json_bar]
provides = ["json_backend"]
~~~

In the above example there are two standard features `json` and `yaml`, which can be activated as needed. The `json`
feature additionally declares, that it requires the capability `json_backend`. `json_backend` is a simple identifier,
which is local to the crate. The two additional features `json_foo` and `json_bar` both provide the functionality of
a JSON backend. Due to some limitation, they are mutually exclusive, and the user has to choose one.

If the user requests the feature `json`, but does not select a feature, providing the capability `json_backend`, then
cargo will fail the build:

~~~
$ cargo build --features json
error: Package `example v0.0.0 (/path)` requires the capability `json_backed`, but no feature which could provide
the capability is enabled. Possible candiates are:
  - json_foo
  - json_bar
~~~

At the same time, if a user select more than one feature that provides `json_backend`, cargo will fail the build as well:

~~~
$ cargo build --features json,json_foo,json_bar
error: Package `example v0.0.0 (/path)` has multiple features enabled that provide the mutually exclusive capability
`json_backend`. Enabled features providing this capability are:
  - json_foo
  - json_bar 
~~~ 

## Another example

In the area of embedded development, it may be interested to let the user provide buffer sizes during compile time to
tweak the memory footprint of an application. The following example shows how this can be achieved using features and
capabilities:

~~~toml
requires = ["buffer-size"]

[features]
default = ["1k"]
1k=[]
2k=[]
4k=[]

[features.1k]
provides = ["buffer-size"]

[features.2k]
provides = ["buffer-size"]

[features.4k]
provides = ["buffer-size"]
~~~

Using the `requires` field, the build requests the user to select a feature providing the capability `buffer-size`, and
at the same time, defaults to the feature `1k`, which will provide an implementation of a 1k buffer.

The user has the ability to choose a different buffer size, using the different features. At the same time, cargo will
ensure that exactly one provider of `buffer_size` is enabled.

## Global example

It is possible to let a feature declare a global capability, which takes effect on the full build tree. For example is
it possible to define a "panic handler". However, only one panic handler may be defined for an application.

Consider the application having the following `Cargo.toml`

~~~toml
requires = ["global:panic-handler"]

[dependencies]
panic-rtt-target = { version = "0.1.1", features = ["cortex-m"] }
panic-halt = "0.2.0"
panic-itm = "0.4.1"

[features]
panic-by-rtt = ["panic-rtt-target"]
panic-by-halt = ["panic-halt"]
panic-by-itm = ["panic-itm"]
panic-by-mock = []

[features.panic-by-mock]
provides = ["global:panic-handler"]
~~~

Additionally, assume the dependencies `panic-rtt-target`, `panic-halt` and `panic-itm` each have a `Cargo.toml` containing:

~~~toml
provides = ["global:panic-handler"]
~~~

A `provides` declaration on the global section defines that a crate will always provide this capability.

With this configuration, Cargo will now ensure that exactly one implementation of a `panic-handler` is enabled in the
build. It can also provide guidance on features to enable or disable. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

**NOTE:** to be written …

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

An argument to not do this, could be that using `build.rs`, you can already achieve similar results. Without the need
to introduce additional concepts. And while this is partially true, it is not a very user-friendly experience.

Additionally, if different people create different ways of processing situations like this in Rust code, contained
in `build.rs`, it may be complicated over time as different concepts and ways of handling similar situations evolve.

Also is it problematic (or even impossible) to crate proper tooling on logic contained in `build.rs` files.   

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

**NOTE:** to be written …

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

## Package manages like RPM, deb, … 

Packages managers like RPM and others, already have the concept of capabilities. It is possible to define dependencies
on specific other packages, or on "capabilities", which may be provided by different packages.

For example, an RPM of `sendmail` may declare that it provides the capability of `MTA` (mail transfer agent).
http://rpmfind.net/linux/RPM/fedora/32/x86_64/s/sendmail-8.15.2-43.fc32.x86_64.html. At the same time,
other packages can provide the same capability: http://rpmfind.net/linux/rpm2html/search.php?query=MTA

A crate can be seen a "software package" and Cargo as a package manager.

A key difference to packages managers like RPM is, that RPM doesn't consider a capability "mutually exclusive". This
can however be achieved by the use of additional rules, like "Conflicts". Also do such package managers support
weak dependencies, like "Suggest" or "Recommends". However, adding such concepts makes things more complex, harder to
implement and harder to understand for the user. 

## OSGi

The Java programming language itself did not have module concept before Java 9. However, OSGi did provide a "bundle"
concept on top of Java before that. Dependencies between bundles can be declared explicitly by referencing another
bundle, or using generic capabilities. See: https://blog.osgi.org/2015/12/using-requirements-and-capabilities.html

"Bundles" in OSGi would map to "crates" in Rust.

While the capabilities concepts have been re-used in OSGi package managers (like P2 or BND), OSGi itself enforces the
capabilities during runtime. Which Rust/Cargo would not do.

For OSGi it is also possible to use much more complex definitions, providing actual values in the capabilities, and
enforcing requirements using LDAP queries. While this allows one to crate rather complex constructs, it also makes
things harder to understand and maintain. Both from an implementation as well from a user perspective.

If such complex scenerios are required, this proposal still allows for the use of custom code in the `build.rs` script
to implement more complex requirements.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

**NOTE:** To be written … 

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
