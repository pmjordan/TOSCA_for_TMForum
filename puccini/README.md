Puccini
=======

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Latest Release](https://img.shields.io/github/release/tliron/puccini.svg)](https://github.com/tliron/puccini/releases/latest)
[![Go Report Card](https://goreportcard.com/badge/github.com/tliron/puccini)](https://goreportcard.com/report/github.com/tliron/puccini)

Deliberately stateless cloud topology management and deployment tools based on
[TOSCA](https://www.oasis-open.org/committees/tosca/).

Each tool is a self-contained executable file, allowing them to be easily distributed and easily
embedded in tool chains, orchestration, and development environments. They are coded in 100%
[Go](https://golang.org/) and are very portable, even runnable on
[WebAssembly](https://webassembly.org/). 

[![Download](assets/media/download.png "Download")](https://github.com/tliron/puccini/releases)

Welcome, first timers! Check out the [quickstart guide](QUICKSTART.md).

To build Puccini yourself see the [build guide](scripts/).


puccini-tosca
-------------

* [**puccini-tosca** documentation](puccini-tosca/)
* [TOSCA parser documentation](tosca/parser/)

Clout frontend for TOSCA. Parses a TOSCA service template and compiles it to Clout (see below).

Why TOSCA? It's a high-level language made for modeling and validating cloud topologies using
reusable and extensible objects. It allows architects to focus on application design and
requirements without being bogged down by the ever-changing specificities of the infrastructure.

Puccini can ingest several popular TOSCA and TOSCA-like dialects:
[TOSCA 1.3](http://docs.oasis-open.org/tosca/TOSCA-Simple-Profile-YAML/v1.3/TOSCA-Simple-Profile-YAML-v1.3.html) (in progress),
[TOSCA 1.2](http://docs.oasis-open.org/tosca/TOSCA-Simple-Profile-YAML/v1.2/TOSCA-Simple-Profile-YAML-v1.2.html),
[TOSCA 1.1](http://docs.oasis-open.org/tosca/TOSCA-Simple-Profile-YAML/v1.1/TOSCA-Simple-Profile-YAML-v1.1.html),
as well as the more limited grammars of
[Cloudify DSL 1.3](https://docs.cloudify.co/latest/developer/blueprints/),
and [OpenStack Heat HOT 2018-08-31](https://docs.openstack.org/heat/stein/template_guide/hot_guide.html).

TOSCA is supported as straightforward YAML files, in the file system or hosted on URLs, as well as packaged
[CSAR files](http://docs.oasis-open.org/tosca/TOSCA-Simple-Profile-YAML/v1.1/os/TOSCA-Simple-Profile-YAML-v1.1-os.html#_Toc489606742).
Puccini also includes a simple CSAR creation tool, **puccini-csar**.

**puccini-tosca** comes with TOSCA profiles for the
[Kubernetes](assets/tosca/profiles/kubernetes/1.0/) and
[OpenStack](assets/tosca/profiles/openstack/1.0/) cloud infrastructures, as well as
[BPMN processes](assets/tosca/profiles/bpmn/1.0/).
Profiles include node, capability, relationship, policy, and other types. Also included are detailed
[examples](examples/) using these profiles to get you started. These profiles would work with any
TOSCA-compliant product.

What's special to Puccini is the inclusion of JavaScript code as an option for self-contained
orchestration integration. How do TOSCA, Clout, JavaScript, and cloud infrastructures all fit
together in Puccini? Consider this: with a single command line you can take a TOSCA service template,
compile it with **puccini-tosca**, pipe the Clout through the **puccini-js** processor, which will
run JavaScript to generate Kubernetes specifications, then pipe those to
[kubectl](https://kubernetes.io/docs/reference/kubectl/overview/),
which will upload the specifications to a running Kubernetes cluster in order to be scheduled.
Like so:

     puccini-tosca compile my-app.yaml | puccini-js exec kubernetes.generate | kubectl apply -f -

Et voilà, your abstract architectural design became a running deployment.

### Standalone Parser

Puccini's [TOSCA parser](tosca/parser/) is available as an independent Go library. Its 5 phase
do normalization, validation, inheritance, and assignment of TOSCA's many types and templates,
resulting in a [flat, serializable data structure](tosca/normal/) that can easily be consumed by
your program. Validation error messages are precise and useful. It's a very, very fast multi-threaded
parser, fast enough that it can be usefully embedded in editors and IDEs for validating TOSCA while
typing.

TOSCA is a complex object-oriented language. We put considerable effort into adhering to every
aspect of the grammar, especially in regards to value type checking and type inheritance contracts,
which are key to delivering the object-oriented promise of extensibility while maintaining reliable
base type compatibility. Unfortunately, the TOSCA specification is famously inconsistent and imprecise.
For this reason, the Puccini parser also supports [quirk modes](tosca/parser/QUIRKS.md) that enable
alternative behaviors based on differing interpretations of the spec.

### Compiler

The TOSCA-to-Clout compiler's main role is to take the parsed data structure and dump it into
Clout. The Clout includes any JavaScript required to process the data. Thusly Clout functions as an
"intermediate representation" (IR) for TOSCA.

By default the compiler also performs [topology resolution](tosca/compiler/RESOLUTION.md), which
attempts to satisfy requirements with capabilities, thus creating the relationships (Clout edges)
between node templates. This feature can be turned off in order to add more processing phases
before final resolution. Resolution is handled via the embedded **tosca.resolve** JavaScript.

### Visualization

You can graphically visualize the compiled TOSCA in a dynamic web page. A one-line example:

    puccini-tosca compile examples/tosca/requirements-and-capabilities.yaml | puccini-js exec assets/tosca/profiles/common/1.0/js/visualize.js > /tmp/puccini.html && xdg-open /tmp/puccini.html

The visualization JavaScript is not embedded by default into the Clout, but can be manually
added via `puccini-js put` (see below).


puccini-js
----------

* [**puccini-js** documentation](puccini-js/)

Clout processor for JavaScript. Executes existing JavaScript in a Clout file, and can also be
used to add/remove JavaScript. For example, it can execute the Kubernetes specification generation
code inserted by **puccini-tosca**, as well as TOSCA functions and value constraints.

Also supported are implementation-specific JavaScript "plugins" that allow you to extend existing
functionality. For example, you can add a plugin for Kubernetes to handle custom application needs,
such as adding sidecars, routers, loadbalancers, etc. Indeed, Istio support is implemented as a
plugin. You can also use **puccini-js** to add plugins to the Clout file, either storing them
permanently or piping through to add and execute them on-the-fly.

### TOSCA Functions and Constraints

These are implemented in JavaScript so that they can be embedded into the Clout and then be
executed by **puccini-js**, allowing a compiled-from-TOSCA Clout file to be entirely independent
of its TOSCA source. The Clout lives on its own.

To call these functions we provide the **tosca.coerce** JavaScript, which calls all functions and
replaces the call stubs with the evaluated values:

    puccini-js exec tosca.coerce my-clout.yaml > coerced-clout.yaml

For convenience, this functionality is included in **puccini-tosca** as the `--coerce` switch:

    puccini-tosca compile --coerce my-clout.yaml

A useful side benefit of this implementation is we allow you to easily extend TOSCA by
[adding your own functions/constraints](examples/javascript/). Obviously, such custom functions
are not part of the TOSCA specification and may be incompatible with other TOSCA implementations.

### TOSCA Attributes

TOSCA attributes (as opposed to properties) represent live data in a running deployment. And the
function `get_attribute` allows other values to make use of this live data. The implication is that
some values in the Clout should change as these attributes change. But also, attribute definitions
in TOSCA allow you to define constraints on the value, so we must also make sure that the new data
complies with them.

Our solution has two steps. As an example, let's look at Kubernetes. First, we have JavaScript
(**kubernetes.update**) that extracts these attributes from a Kubernetes cluster (by calling
**kubectl**) and updates the Clout. Second, we run **tosca.coerce**, which not only calls TOSCA
functions but also applies the constraints.

Putting it all together, let's refresh a Clout:

    puccini-js exec kubernetes.update my-clout.yaml | puccini-js exec tosca.coerce > coerced-clout.yaml

### TOSCA Interfaces, Operations, Workflows, Notifications, and Policy Triggers

*WORK IN PROGRESS*

TOSCA workflows are an abstraction of task graphs that are tightly coupled with the topology. They
represent the "classical" orchestration paradigm, which procedurally (in serial and/or in parallel)
executes individual self-contained operations that when successful achieve a total state for an
application.

TOSCA profiles in Puccini may come with built-in domain-specific "normative" workflows. For example,
OpenStack has workflows to provision and remove its resources. TOSCA 1.1 introduced custom workflows,
which can be used for changing the state of existing deployments. Notifications and policy triggers
are a related feauture, as they specify an event or condition that could launch a workflow or an
individual operation.

Puccini provides three different implementations of these features:

1) For OpenStack, Puccini can generate [Ansible](https://www.ansible.com/) playbooks that rely on
the
[Ansible OpenStack roles](https://docs.ansible.com/ansible/latest/modules/list_of_cloud_modules.html#openstack).
Custom operation artifacts, if included, are deployed to the virtual machines and executed.
Effectively, the combination of TOSCA + Ansible provides an equivalent set of features to
[Heat](https://docs.openstack.org/heat/stein/)/[Mistral](https://docs.openstack.org/mistral/rocky/).
However, Ansible is a general-purpose orchestrator that can do a lot more than Heat. The generated
playbooks comprise roles that can be imported and used in other playbooks, allowing for custom
orchestration integrations. Also note that although Puccini can compile HOT directly, we recommend
TOSCA because of its much richer grammar and features.  

2) Puccini's BPMN profile lets you generate [BPMN2](https://www.omg.org/spec/BPMN/) processes from
TOSCA workflows and policy triggers. These allow for tight integration with enterprise process
management (called [OSS/BSS](https://en.wikipedia.org/wiki/OSS/BSS) in the telecommunications
industry). The generated processes can also be included as sub-processes within larger business
processes. 

3) Kubernetes doesn't normally require workflows: its "scheduling" paradigm is a declarative
alternative to the classical procedural orchestration paradigm. As it provides a truly
cloud-native environment, Kubernetes applications are better off orchestrating themselves, for
example by relying on [operators](https://github.com/operator-framework/operator-sdk) to do the
heavy lifting. Still, it could make sense to use workflows for certain externally triggered
features. Puccini's solution is straightforward: it can generate an Ansible playbook that deploys
TOSCA artifacts with `kubectl cp` and executes them with `kubectl exec`.


Clout
-----

* [Clout documentation](clout/)

Introducing the **clou**d **t**opology ("clou" + "t") representation language, which represents a
simple graph database in YAML/JSON/XML.

Clout is an intermediary format for your deployments. As an analogy, consider a program written in
the C language. First, you must *compile* the C source into machine code for your hardware
architecture. Then, you *link* the compiled object, together with various libraries, into a
deployable executable for a specific target platform. Clout here is the compiled object. If you only
care about the final result then you won't see the Clout at all. However, this decoupling allows for
a more powerful tool chain. For example, some tools might change your Clout after the initial
compilation (to scale out, to optimize, to add platform hooks, debugging features, etc.) and then
you just need to "re-link" in order to update your deployment. This can happen without requiring
you to update your original source design. It may also possible to "de-compile" some cloud
deployments so that you can generate a Clout without "source code".

Clout is essentially a big, unopinionated, implementation-specific dump of vertexes and the edges
between them with un-typed, non-validated properties. Rule #1 of Clout is that everything and the
kitchen sink should be in one Clout file. Really, anything goes: specifications, configurations,
metadata, annotations, source code, documentation, and even text-encoded binaries. (The only
possible exception might be that you would want to store security certificates and keys
elsewhere.)

In itself Clout is an unremarkable format. Think of it as a way to gather various deployment
specifications for disparate technologies in one place while allowing for the *relationships*
(edges) between entities to be specified and annotated. That's the topology.

Clout is not supposed to be human-readable or human-manageable. The idea is to use tools (Clout
frontends and processors) to deal with its complexity. We have some great ones for you here. For
example, with Puccini you can use just a little bit of TOSCA to generate a single big Clout file
that describes a complex Kubernetes service mesh.

If Clout's file size is an issue, it's good to know that Clout is usually eminently compressible,
comprising just text with quite a lot of repetition.

### Storage

Orchestrators may choose to store Clout opaquely, as is, in a key-value database or filesystem.
This could work well because cloud deployments change infrequently: often all that's needed is to
retrieve a Clout, parse and lookup data, and possibly update a TOSCA attribute and store it again.
Iterating many Clouts in sequence this way could be done quickly enough even for large
environments. Simple solutions are often best.

That said, it could also make sense to store Clout data in a graph database. This would allow for
sophisticated queries, using languages such [GraphQL](https://graphql.org/) and
[Gremlin](https://tinkerpop.apache.org/gremlin.html), as well as localized transactional updates.
This approach could be especially useful for highly composable and dynamic environments in which
Clouts combine together to form larger topologies and even relate to data coming from other systems.

Graph databases are quite diverse in features and Clout is very flexible, so one schema will not
fit all. Puccini instead comes with examples: see [storing in Neo4j](examples/neo4j/) and
[storing in Dgraph](examples/dgraph/).


FAQ
---

### Can Puccini deploy my existing TOSCA to Kubernetes or OpenStack?

If you didn't plan it that way, then: no. Our TOSCA Kubernetes/OpenStack profiles do *not* make use
of TOSCA's
[Simple Profile](http://docs.oasis-open.org/tosca/TOSCA-Simple-Profile-YAML/v1.1/TOSCA-Simple-Profile-YAML-v1.1.html)
or [Simple Profile for NFV](http://docs.oasis-open.org/tosca/tosca-nfv/v1.0/tosca-nfv-v1.0.html)
types (Compute, BlockStorage, VDU, etc.). Still, if you find these so-called "normative" types
useful, they are included in Puccini and will be compiled into Clout. You may bring in your
own orchestration to deploy them to your cloud environments. But, we encourage you to consider
carefully whether this is a good idea. We think it's a dead end.

The notion that a single set of normative types could be used for all the various cloud and container
platforms out there is a pipe dream. There may be superficial similarities between them, but the devil
is in the details and the amount of detail needed for integrated, scalable, cloud-native deployments
keeps growing and diversifying. Thus every platform needs and deserves its own concepts, models, data
points, and thus its own profile of interrelated TOSCA types. However, by bringing all these
tiny-but-important implementation details into one place we can at least have a lingua franca and
common tool chain for all platforms. That's the value proposition of TOSCA and the gist of Puccini.

### Why Go?

[Go](https://golang.org/) is fast becoming the language of choice for cloud-native solutions.
It has the advantage of producing very deployable executables that make it easy to containerize
and integrate. Go features garbage collection and easy multi-threading (via lightweight
goroutines), but unlike Python, Ruby, and Perl it is a strictly typed language, which
encourages good programming practices and reduces the chance for bugs.

### Why JavaScript?

The decision to use an embedded interpreted programming language is intentional and important.
Unlike some Kubernetes tools ([Helm](https://helm.sh/)), we do not treat YAML files as plain text
to be manipulated by an anemic text templating language, where just working around YAML's strict
indentation requirements becomes a cumbersome nightmare.

JavaScript lets you manipulate data structures directly using a full-blown, conventional language.
It's probably
[not anyone's favorite language](https://archive.org/details/wat_destroyallsoftware), but it's
familiar, mature, standardized (as [ECMAScript](https://en.wikipedia.org/wiki/ECMAScript)), and does
the job. From a certain angle it's essentially the Scheme language (because it has powerful closures
and functions are first class citizens) but with a crusty C syntax.

And because JavaScript is self-contained text, it's trivial to store it in a Clout file, which can
then be interpreted and run almost anywhere.

Our chosen ECMAScript engine is [goja](https://github.com/dop251/goja), which is 100% Go and does
not require any external dependencies.

### Can I use simple text templating instead of TOSCA functions and JavaScript?

Nothing is stopping you. You can pipe the input and output to and from the text translator of your
choice at any point in the tool chain. Here's an example using
[gomplate](https://github.com/hairyhenderson/gomplate):

    puccini-tosca compile my-app.yaml | gomplate | puccini-js exec kubernetes.generate

Your TOSCA can then inject expressions into values, such as:

	username: "{{strings.ReplaceAll "\"" "\\\"" .Env.USER}}"

Just make sure that your templating engine can emit valid YAML where appropriate. For example, it
should be able to escape quotation marks, like we did above.

A useful convention for your toolchain could be to add a file extension to mark a file for text
template processing. For example, `.yaml.j2` could be recognized as requiring Jinja2 template
processing, after which the `.j2` extension would be stripped.

### Can I compose a single service from several interrelated Clout files?

TOSCA has a feature called "substitution mapping", which is useful for modeling service composition.
It is, however, a *design* feature. The implementation is up to your orchestration tool chain. See our
examples
[here](examples/tosca/substitution-mapping.yaml) and
[here](examples/tosca/substitution-mapping-client.yaml).

Puccini intentionally does *not* support service composition. Each Clout file is its own universe.
If you need to create edges between vertexes in one Clout file and vertexes in other Clout files,
then it's up to you and your tools to design and implement that integration. The solution could be
very elaborate indeed: the two Clouts might represent services with very different lifecycles, that
run in different clouds, that are handled by different orchestrators. And the connection might
require complex networking to achieve. There's simply no one-size-fits-all way Puccini could do
it—namespaces? proxies? catalogs? repositories?—so it insists on not having an opinion.

### TOSCA is so complicated! Help!

I know, right? Now imagine writing a parser for it... Not only is it a complex language, but the
[specification itself](http://docs.oasis-open.org/tosca/TOSCA-Simple-Profile-YAML/v1.2/TOSCA-Simple-Profile-YAML-v1.2.html)
(as of version 1.2) has many contradictions, errors, and gaps.

Please join [OASIS's TOSCA community](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=tosca)
to help improve the language!

Meanwhile, we've included [examples](examples/tosca/) of TOSCA core grammatical features,
with some running commentary. Treat them as your playground. Also, if you have 4 hours to spare,
grab some snacks, get comfortable, and watch the author's free online course for TOSCA 1.0:
[part 1](https://www.youtube.com/watch?v=aMkqLI6o-58),
[part 2](https://www.youtube.com/watch?v=6xGmpi--7-A).

(Author's note: This is my second take at writing a TOSCA parser. The first was
[AriaTosca](https://github.com/apache/incubator-ariatosca), an
incubation project under the Apache Software Foundation. I am grateful to
[Cloudify](https://cloudify.co/) for funding much of the AriaTosca project. Note, however, that
Puccini is a fresh start initiated by myself with no commercial backing. It does not use
AriaTosca code and has a radically different architecture as well as very different goals.)

### Why is it called "Puccini"?

[Giacomo Puccini](https://en.wikipedia.org/wiki/Giacomo_Puccini) was the composer of the
[*Tosca*](https://en.wikipedia.org/wiki/Tosca) opera (based on Victorien Sardou's play,
[*La Tosca*](https://en.wikipedia.org/wiki/La_Tosca)), as well as *La bohème*, *Madama Butterfly*,
and other famous works. The theme here is orchestration, orchestras, composition, and thus operas.
Capiche?

### How to pronounce "Puccini"?

For a demonstration of its authentic 19th-century Italian pronunciation see
[this clip](https://www.youtube.com/watch?v=dQw4w9WgXcQ).
