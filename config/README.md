# Configuration and Installation Reconsidered

Any kind of software installation, update or generally lifecycle management
involves a setup and/or update procedure and the parameterization of this
(installation) procedure.
There are plenty of such environments like [*Terraform*](https://www.terraform.io), [*Chef*](https://www.chef.io), [*Ansible*](https://ansible.com),
package managers like [*APT*](https://wiki.ubuntuusers.de/APT), [*Crossplane*](https://www.crossplane.io) or even [*Kubernetes*](https://kubernetes.io) (we will see later why
this appears in this list).
Configuration is used to adapt the installation process to the needs of
particular applications or application installations.

There are several approaches to simplify the description used to control the
installation process and simplify the creation of installers at all,
from templating used to apply patterns or rules to avoid
duplicate information to complete general or product specific DSLs used to
express complex configuration descriptions closer to the problem domain than
simple value structures. Those DSLs can be generalized by frameworks
to shift the installer development from a general purpose language to 
the composition of DSL elements.

In the following we will describe how configurations could look like and how it
interacts with or relates to different concepts for installation procedures.
And we argue that the optimization of configurations and configuration DSLs
does not solve the problem with complex installations. Instead, the crucial elements
are the abstraction level between the configuration elements focusing on
the problem domain and the finally maintained elements in the target environment
and a dynamic description layer.

### The Problem

Basically the installation problem decomposes into two similar problems, the
initial setup of a 
product or application and its update. These parts look very similar, especially
because typically an attempt is made to describe solutions for both problem flavor
in a uniform manner. But there is a large
difference in complexity. Setting up a new installation is relatively simple compared 
with an update to a newer version. During the setup no existing state of an
installation has to be considered, which just required to describe and finally create
the required implementation element in a target environment. In contrast to this, an 
update in its general form has to handle the migration of an existing implementation
structure towards a potentially different one arising from revised or evolved
design decisions. This not only could require the migration of data structures but also
to preserve abstract state distributed over multiple implementation elements.

The term *installation* will be used in the following as a synonym for both *setup* and 
*update*. If required, the term update will explicitly be used.

### General Layout

All installation approaches finally follow the same general layout.
There is an installation or update code in some general purpose programming
language whose task is to provide or update a dedicated installation of some
particular product or component in a target environment. For this it uses an
API provided by the target environment to manage elements offered this
environment. The procedure is product specific. To apply to constraints
for a particular installation instance it consumes some installation
configuration, which is used to specify information used to concretize
variation points supported by the installation procedure valid for this
particular installation instance.

This configuration is typically always data centric, it does not contain 
code used to define the installation process.

<center>
<img src="./media/layout.png" style="width:50%" />
</center>

Optionally the installation procedure may keep some state about the actual
installation state, which is used to complete configuration data when
updating or deleting an existing installation.

Let's consider the configuration description, first.

### Configurations Reconsidered

In the most simple case configuration is described by some general possibly structured
data definition format, like [YAML](https://yaml.org), [JSON](https://www.json.org), [XML](https://www.w3.org/TR/xml) or [INI](/https://docs.fileformat.com/system/ini).
The installation code looks for dedicated fields in the configuration to assign 
a particular meaning. This could be single fields or complete structures whose meaning
is identified by some kind of special type field or its path in the document structure.

By this interpretation by the installer code even simple data formats are used to
effectively describe a DSL, a logical DSL simply defined by structure and type fields.
This shows that it is not necessarily required to define DSL with an own syntax to
describe configuration tailored for a particular problem domain.

<center>
<img src="./media/logical-dsl.png" style="width:50%" />
</center>

Formats for purely defining structures values, like JSON typically lack the feature 
of supporting values derived from other settings or rules for common value layouts. But this is common problem for mre complex scenarios,
values of particular attributes should either follow some rules, depend on
other settings or should be set according to some relations in a particular
scenario. The consequence is, that the same or related values must be
configured independently at different structural locations in the
configuration description.

The description follows the WET approach, a
backronym commonly taken to stand for *write everything twice* (alternatively
*write every time*, *we enjoy typing* or *waste everyone's time*
(wikipedia)). Instead, it would be desired to better follow the DRY
approach, *Don't repeat yourself* or *duplication is evil*.
YAML, for example, feature value references as part of its syntax to be able
to refer to values or complete data structures defined somewhere else in
the document. But this is only of limited value, because it does not allow
expression rules or calculations.

At this point in time typically templating engines enter the scene to close
this gap.
Here, often standardized tools like [Go-templates](https://pkg.go.dev/text/template), [ytt](https://carvel.dev/ytt), [cue](https://cuelang.org)
or [spiff++](https://github.com/mandelsoft/spiff) are used to preprocess a
configuration description to keep the input (and the parsing) for the
installation procedure as simple as before.

<center>
<img src="./media/templates.png" style="width:50%" />
</center>

Templates typically support expressions to solve the problem of derived
values, but they can also be used to introduce explicitly named and
parameterized elements, which can act as syntactical elements in the
finally maintained description. It provides the possibility to offer formal
DSLs on top of simple structured value formats by supporting functions or
named and parameterized templates.

This kind of preprocessing might be done by a general purpose templating
engine or an explicit DSL implementation. In general, the mapping is
achieved by providing libraries usable for every particular
installation parameterization. Such a library can be separated from the
configuration description maintained by the human operator.

<center>
<img src="./media/templated-dsl.png" style="width:50%" />
</center>

The library describes parameterized functions, templates or explicit DSL
elements, which
evaluate to more complex descriptive structures of the underlying data
format. The result is some kind of *intermediate* DSL usable by the 
human operator maintaining the final configuration description.

<center>
<img src="./media/cascade.png" style="width:50%" />
</center>*

With the introduction of libraries we open the scenario for providing
basic installers applicable to a wide range of applications. The concrete
product installer is provided by a library or module set with a product
specific composition of elements offered by the underlying basic installer.
We can clearly distinguish between the basis installer, its
specialization for a dedicated application or product, and its application
for a particular installation instance.
There are two basic possibilities where such libraries can be located:
- It can be located on the right hand side together with the technical
  basic installer itself to support generic application installations.
  This way, it is used to offer a higher-level abstraction or comfort for
  the development of particular product installers. The underlying 
  installer itself is generalized to an installer framework applicable for
  installer development for a particular application domain. Its basic
  installation capabilities are defined by the core functions of the
  underlying technical installer code and the abstractions provided by
  the library.

- Or it can be located on the left hand side as part of the installed product
  to compose elements of the DSL provided by the installer code to meet the
  installation needs for the particular product.

An example for such an environment can be [Crossplane](https://www.crossplane.io)
(providing an extensible core framework) and compositions used to describe
dedicated  application scenarios. Another example is [Terraform](https://www.terraform.io),
which provides core installation features used to create installation
descriptions by orchestrating those basic elements to describe a particular
implementation scenario.

Typically, the first case is always combined with the second case.
The product specific library is implemented with some kind of
*intermediate* DSL, a library extending the basic installer. The product 
library composes elements of the intermediate DSL to describe an
installation of the product. It is then used to describe the DSL
available for the configuration values finally maintained by a product
operator for a dedicated installation of the product.

The concrete installation procedure is now
formulated in the DSL of the installer framework, not in a general purpose
programming language. The final product installer is the combination of the
original installer (framework) and its product specific configuration or
better instrumentation. It is some cascading scenario for creating 
installers*.

For example *Terraform* as an installer framework is used by 
an application to orchestrate the installation logic by terraform resources.
Both, the terraform executable and the orchestration description is
delivered. On the installation side a configuration file is provided by
the human operator to specify the values required to parameterize the
installation procedure. 

In the next step the DSL used to describe the product specific element
composition is not handled by a templating engine and library but is a
builtin DSL directly supported by the basic installer by
integrating the DSL parsing into the installer code. The result is an
integrated installer framework for a dedicated application domain.
Examples are
[Terraform](https://www.terraform.io)
with [HCL](https://github.com/hashicorp/hcl) or [KCL](https://www.kcl-lang.io).
<center>
<img src="./media/integrated-dsl.png" style="width:50%" />
</center>

This typically comes with an extensibility framework allowing to extend the
core installation functionality under the hood of the provided DSL. We
will come back to this in the next section.

All the previous scenarios are always based on some kind of effective 
configuration value set evaluable prior to an installation. The effective
input for the technical installation procedure is a simple value set
according to the very first scenario.

Such integrated DSLs now offer the possibility for a feedback cycle from the
concrete installation process. It is possible to describe a value flow
incorporating results from other installation steps or even inside the
parameterization of a single step as long as a value for an attribute is 
required only after some partial step has been executed to provide the
described effective value.

<center>
<img src="./media/feedback-dsl.png" style="width:50%" />
</center>

An example for such a configuration DSL is *Terraform* with *HCL* or even the *Pod* manifest
used by the *Kubernetes* *kubelet* to configure *Pods* with referential field expressions.

With this last scenario we leave the pure configuration description, and we
have to have a closer look at the installation procedure itself.

### Installer Reconsidered

The installer is coding responsible to map a configuration description
to an appropriate set of elements in the target environment for the
concrete installation described by the configuration. The installer itself is
product or application specific, while the configuration describes
information intended for the actual installation instance.

The last scenario in the previous section already shows a dedicated
variant of an installer, which can combine information from installation
steps with provided configuration. This always requires an appropriate
functional interpretation of expressions provided by the configuration
description (e.g. an expression describing such a
value reference). As such, it is a hybrid concept combining installation
functionality with configuration descriptions. How such a concept can be 
generalized will be shown in the next section.

Orthogonally to this interwoven behaviour between installation and
configuration, it is possible to distinguish two different execution flavors:

- An installer can explicitly be called to execute an installation or an installation update, either manually or as part of a CI/CD pipeline.
- It can be an ongoing process aligning changes in the configuration with changes in the target environment. This kind of drift-control is called reconcilation or reconcilation-loop and is well-known from the *Kubernetes* ecosystem.

<center>
<img src="./media/procedure.png" style="width:50%" />
</center>

The next flavors describing an own orthogonal dimension arise when
considering the internal structure of the installer.
The most simple one consists of explicit coding specialized for the intended
installation scenario, including the evaluation of the configuration
description.

<center>
<img src="./media/explicit.png" style="width:50%" />
</center>

This is basically a direct implementation of the general layout shown
previously. The typical application of this pattern are ad-hoc installers
using scripts to provide a quick tool to set up something before it is
remastered to achieve a productively usable solution.

In a next iteration an installation framework is used to embed the
application specific installation/update procedure. The framework typically 
covers three functionalities:
- it provides access to the configuration values in a formalized way by evaluating the syntactical elements of the configuration format.
- it controls the execution of the application specific code.
- if multiple installers are involved it handles the dependency management.

<center>
<img src="./media/framework.png" style="width:50%" />
</center>

Typical applications of this pattern are operating system level packages
handled by a package manager, like [APT](https://wiki.ubuntuusers.de/APT).
But also [Kubernetes](https://kubernetes.io) basically follows this
pattern, the configuration description is the resource manifest, the
specialized code is the controller or operator. Kubernetes takes the role
of the framework by handling the configuration descriptions, coordinating 
the requests to the controller to execute updates and even to run the
controllers.

<center>
<img src="./media/extensible-framework.png" style="width:50%" />
</center>

In many cases this framework is able to
handle multiple specialized installers for different kinds of installation
steps. The configuration description may describe multiple,
now typed, elements according to those types. Every type has assigned
specialized code, which is responsible to map the described elements of this
type to elements in the target environment.
The data plane/control plane concept of *Kubernetes* is an example for this
pattern.

But the most common use-case of this pattern in the domain of installers is
to provide some kind of generalized DSL, which can be used to compose
multiple interdependent installation steps to describe the
installation of a product. The installer for a particular application is
developed with the DSL provided by the framework by composing instances
of the predefined installation types. To still offer the possibility to
describe particular installation instances for the described application,
the framework again provides the support for a separated specification of 
configuration values, which can be used to parameterize the elements
described by the DSL.

<center>
<img src="./media/framework-values.png" style="width:55%" />
</center>

Now, the DSL part provided by the framework and used to describe a
composition of the supported installation elements is used to *implement*
the installer. And the installation instance specific value configuration
is used to concretize the variation points offered by the installer. 

As a result, the final installer consists of a dedicated product specific
DSL composition and the product independent generic installer framework.
It is some kind of cascaded application of the initial installer layout
as shown in the previous section. The DSL interpretation if typically
integral part of the basic installer code.

The idea behind this concept is to simplify and standardize the development
of installers. The installation problem is decomposed into the description
of low-level fine granular elements offered by the (potentially extensible)
installation framework. Instead of writing general purpose code, now the
composition of those elements is described on the DSL level. The coordination
and the mapping of the descriptive elements to the target environment is
handled by the basic installer framework and its type specific mapping
implementations.

A typical example for such an environment is *Terraform* with *HCL* as 
configuration DSL or even *Crossplane* enriched by extensions for various
kinds of target environments offering the management for the elements
provided by those environments
(e.g. IaaS layers offering elements like VMs, networks, VPCs or volumes).

This approach has typically four constraints for the design of the DSL:
- to be as flexible as possible for describing installations and to avoid the
  creation of own general purpose code, the elements
  offered on the DSL level have to be as close as possible to the elements
  of the target environment.
- therefore the extensible installer framework code and its extensions
  provide a fixed 1:1, or 1:n (where n is very small) mapping to elements
  of the target environment.
- the framework must also handle the deletion of elements if they disappear
  from the concrete DSL manifestation.
- to provide higher level abstractions, elements like compositions or
  modules are supported, offering an own parameterization.

Unfortunately, this has the consequence, that the installer on the DSL
level looses the control over the coordination of element creation and
deletion. And it looses the possibility to handle
migrations which could avoid loss of information
(e.g. deleting a database and creating a new one is never
a good idea, if the database contains data).

For the setup part of an installation this is not really a problem. 
It typically only requires some basic
ordering and value dependencies among the created implementation elements.
But it will cause problems for day-2 update procedures, because migrations
must be considered to avoid fixing the implementation structure with the
first shot of the installer.

This becomes even more evident, if the framework keeps state about the
mapping of described elements, which is bound to or identified by
the structuring of those elements in the DSL. A structural migration, and
therefore the evolution of the installation or implementation (of the
application) structure, is hardly possible, because the overall abstract
state must be preserved. But instead an implementation related state
structure is held.

A typical telltale sign here can be seen for *Terraform*: it is highly
recommended to carefully examine the output of the plan statement, before
really applying a change. It allows examining the change operations,
which would be done if the configuration is applied.

All approaches to circumvent those problems are typically
- highly specialized for particular problem flavors
- or require the embedding of general purpose code combined with complex synchronization mechanism to connect it with the implicit handling by the framework.

All this makes it very complicated, confusing and error-prone
to work with such installer DSLs when evolving the implementation
structure, if it is possible at all.

### Target Environment Reconsidered

A new quality can be achieved, if the configuration or the DSL-based
composition is not a static input to the complete installation process,
anymore. A first step towards this direction
has been shown in the last section, where it is possible to
refer to information from the target environment by using special
static expressions in the static configuration description. This a
special case for a feedback loop, which can be generalized to whole
element configuration descriptions.

<center>
<img src="./media/recursion.png" style="width:55%" />
</center>

In such a scenario, the static description is replaced by a completely
dynamic one, by considering the configuration itself as part of the target
environment. Configuration descriptions can be created, manipulated or
deleted on demand, even during the execution of an installation step.

The result is 
a recursion, which enables particular installer code to implement its
configuration elements by other configuration elements. The decision for
the mapping is not a static one, but a free decision of the implementation 
of the configuration elements.

This feature is tightly coupled with the decomposition of the overall
installer into separately processed configuration element types,
in best case combined with the extensibility of the set of element types,
and a reconcilation approach. The last feature is required to be able
to react on changes of the set of configuration elements and their
attribution appearing after the initial process has been started.
Therefore, it combines most of the elements shown before to drastically
increase the expressive power of the overall system as well as the
abstraction available for the installer implementation. It can
asynchronously decompose into other configuration elements and
combine this with own synchronization and ordering logic.


### Git Ops versus Configuration/Installation

A popular operational model to handle installations is
[*GitOps*](https://www.gitops.tech). You might wonder why
it has not been covered yet as part of the installation process.

The reason is that basically *GitOps* is not part of the technical
installation process.
It is a separate process orchestrated around the pure installation process. 
Its task is to handle the process of bringing together configurations
stored in
versioning repositories with the execution of the installation steps based on
those configurations, but it does not describe the installation process itself.

<center>
<img src="./media/gitops.png" style="width:50%" />
</center>

So, *GitOps* finally just embeds and wraps a configuration and installation
process as described by the previous sections.

Nevertheless, *GitOps* systems like [Flux](https://fluxcd.io/) or
[ArgoCD](https://argoproj.github.io/cd/) might also enrich such a
process by tooling generating the configurations used, e.g. using
[Kustomize](https://kustomize.io/) 
to template and modify *Kubernetes* manifests (as resource configurations)
prior to handing them over to the execution environment (the *Kubernetes*
data plane).

In such a case the templating shown in the previous sections is either
completely externalized or extended by the GitOps system.
Because of the nature of the configuration representation in a versioning
system, GitOps also take the role of describing compositions of
configurations elements represented as different files. But finally those
descriptive elements are always transformed and transferred
into a representation and location from where they can be consumed by
involved technical installers.

### Conclusion

First of all, we see that DSLs are present everywhere, where configuration is interpreted.
Even with simple data formats it is possible to express rules, commonalities and derived values.
The expressive power comes from the evaluation logic as part of the installation code.
Explicit specialized DSL can be seen as syntactical sugar to improve the readability.

A tiny step forward is the possibility to incorporate reflection of the target environment
into the parameterization of a configuration description.
But a completely new quality only comes with the possibility to describe reactions
on the actual target state compared with the described desired state.
This not only means the evaluation of configuration values derived from installation steps
or the actual target state, but the synchronization of installations
steps and even deciding on
installations steps because of configuration changes or, even more important, because of
changed implementation decisions. This means that the installer might choose
a changed mapping of the installation to elements into the target environment.
This might cause additional actions, like data migrations, which must be synchronized
with the other installation steps.

All this requires code provided by the product or application, but the human operator
still has to use his basic (simple) configuration setting for the particular
installation instance.
Even with the described scenario separating configuration values from configuration element 
settings (as done by *Terraform* or *Crossplane* compositions) using an intermediate DSL, this
kind of coding is required, because the set of mapped elements in the target environment might
completely change after an update. An *intermediate* DSL able to express all those cases is 
typically again a turing complete language, with synchronization, steps and explicit calls
to the API of the target environment.

Because of this, in most cases, it is more convenient for an installation developer to implement
everything in the programming language he is used to. The only support by an installation
framework should cover the access to the configuration values and potentially
the execution and configuration of the installation process implementation itself.

So, we are finally
back to the very first picture by reducing the configuration description to the
demands of the semantic of the installation, but never the concrete layout or the concretely 
maintained elements in the target environment. We can see that the flexibility for
designing update procedures can only be increased by also increasing the abstraction
level between  the configuration description and the elements required in the target domain.

<center>
<img src="./media/abstract.png" style="width:50%" />
</center>

The conclusion therefore must be, that the flexibility really required to support
necessary update steps and to decouple them from the
configuration expected from the human operator in the long term can only be achieved
by a very high abstraction between the configuration description and its final
mapping to a composition of elements in the target environment. The description purely
focuses on *what* should be achieved (the intent), but not *how* it could
be achieved (the required implementation elements). Only this can assure, that the 
implementation can freely be chosen by the installer.

When bringing all together, ongoing-drift control, extensibility, abstract
configuration and full control of the implementation mapping, we end up in
a framework, which looks as follows:

<center>
<img src="./media/operator-framework.png" style="width:50%" />
</center>

It is able to handle multiple, potentially different installations by 
explicit free coding able to execute any manipulation of a target
environment.

To make this kind of abstraction available again for the implementation of
particular installer code a dynamic configuration description should
be chosen. It allows combining the expressive power of such a configuration
model with the necessary flexibility on the implementation level.

An example for such a framework can be the [Kubernetes Resource Model](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md) architecture,
which provides a framework for running operators written in arbitrary general
purpose programming languages extending an extensible data plane hosting arbitrary
abstract resource configurations in a simple configuration format for pure data
and responsible for mapping those configured abstract resources to any
implementation in possibly any target environment.

Because all operators have access to the data plane they can evaluate resource
references as part of their resource manifests to access appropriate manifests
of other (used) resources, which reflect additional information about required 
runtime attributes. Following the reconcilation approach dynamic dataflow can be 
achieved among different kinds of resources.

In contrast to a static description layer as implemented by traditional installation
systems, the *KRM* Data Plane as description
layer is very dynamic. New elements can arbitrarily be created, updated or deleted
via API during the *overall* installation process. This enables a flexible cascading
of resource implementations backed by other (potentially more low level)
configuration elements. This is new quality for the feedback between the target environment
and configuration description: the description layer can be used as target environment.
It is not only possible to describe value relations,
but complete configuration elements can be maintained during the installation process.
This cascading allows implementing completely dynamic setup and update flows
by falling back to other descriptive elements, combining general purpose code to
provide the required descriptive power and descriptive implementation objects. No other
static description formalism described by a DSL can achieve this new quality.

<center>
<img src="./media/kubernetes.png" style="width:50%" />
</center>

Because of the nature of those resource declarations only fixed values are
possible in the manifests, so it does not support the DRY approach. But there
is a rich tool ecosystem handling such mechanism at the level of *GitOps*.

This typically supports basic expression and template functionality but no
feedback loop based on expressions. If this is required in special cases the
resource manifest format can be used to implement an appropriate DSL by providing
values recognizable as expressions.
The particular operator then has to implement this DSL. This way any kind of
feedback loop, including the access to attributes of other resource kinds can be
implemented.

But because of the dynamic behavior of the description layers and drift control
implemented by operators it is possible
for operators to avoid all this and fill configuration attributes of resources
on-the-fly with values arising from the mapping processing.

This demonstrates the executive and descriptive power of a KRM-based ecosystem,
which makes it highly suitable for installation scenarios.

