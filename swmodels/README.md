# From Development to System Landscapes - A Hierarchy of Software Models

Modern software systems are built from a growing number of components, assembled into delivery artifacts, delivered to fenced environments, deployed across complex landscapes, and operated as interdependent runtime elements. Yet the descriptions accompanying these artifacts rarely follow them all the way through — deployment configurations drift from their origins, and runtime environments become difficult to trace back to what was originally delivered and what should intentionally be deployed.

Various description models exist for each level of the software lifecycle, each well-established within its own domain:

**Building artifacts** is covered by package formats and build metadata such as SPDX, CycloneDX, or OCI image manifests, which describe what went into a build and its composition.

**Describing deliveries** is addressed by package managers and release formats — Helm charts, OCI artifacts, or Debian/RPM packages — which describe what is being shipped and in what version.

**Deriving vulnerabilities** relies on formats like CVE, VEX (Vulnerability Exploitability eXchange), or OSV, which describe known weaknesses against software components.

**Describing the landscape topology** is addressed by models like C4, ArchiMate, or TOSCA, which describe the logical structure of a landscape — its components, their relationships, and dependencies — expressing the intent of what should exist and how it fits together, independent of deployment mechanics.

**Describing the orchestration of landscape deployments** is addressed by Infrastructure-as-Code tools such as Terraform, Pulumi, or CloudFormation for provisioning the underlying infrastructure, and by deployment orchestration tools such as Helm, Kustomize, or ArgoCD application definitions for describing what should be deployed where — together expressing the desired deployment state of a landscape in terms of concrete artifacts, their configuration, and their target environments.

**Describing runtime environments** is the domain of orchestration platforms like Kubernetes manifests, systemd units, or Docker Compose files, which describe how software is actually deployed and operated.

Each of these models is self-contained and optimized for its own purpose. A CycloneDX SBOM does not know which landscape it will be deployed into. A Helm chart does not reference the SBOM of the image it ships. A Kubernetes manifest does not carry the vulnerability context of its running containers. A Terraform description does not know what will eventually run on the infrastructure it provisions.
The result is a set of isolated snapshots — accurate within their own scope, but disconnected from each other, making end-to-end traceability across the lifecycle effectively impossible without significant manual effort. Typically, descriptions like components, artifacts or package are used, but in order to describe elements of a cloud or systems that need to be delivered/installed/maintained on prefabricated hardware or given infrastructure/platforms, we need to describe the elements not in their logical function (components) or transport envelope (artifact or package, along with metadata such as SBoMs) but with their use when provisioned as "Service".

This document describes a closed, end-to-end description methodology that connects delivery artifacts through orchestrated deployments to the effective runtime landscape, maintaining full traceability across the entire chain.
This will be achieved by a hierarchy of separate but connected description models.

## The Model Hierarchy

Everything starts with the development of software artifacts, like libraries,
programs, or container images. There are various heterogeneous tools and environments supporting this development step. But all of them typically end up in some build
process providing artifacts intended to be delivered and/or deployed, including
installation procedures or descriptions. Here, we start our journey omitting all
those sneaky details arising from programming ecosystems.

1) Describing Sets of Delivery Artifacts (Software Bill of Delivery)

   As part of the build process, the generated artifacts are described by a
uniform and technology-agnostic model capable to name sets of artifacts and
particular artifacts in those sets. Additionally, the model comes with a
repository specification, which describes storage
formats to store and lookup model element descriptions as well as described
artifacts either local to the description itself
or in external software repositories like OCI registries. This is then used
to attach access information to the artifact description to technically bind
the description to the reality, the technical artifacts. This way, described artifacts 
can directly be accessed following the formal access methods
 being part of the description. 

2) Describing Services (Service Model)

   The service model is a federated model used to describe services with their
API, their installation and the required deployment artifacts. Therefore,
it refers to the identities. The underlying model is not only used for the software 
artifacts, but for the model element description itself. They are
described and stored as part of the Software Bill of Material model and also
use identities provided by this model. Based on registries for the underlying model
tools can directly access the model element descriptions based on their
identities in a location-agnostic way.

   As such, model element descriptions are, like software artifacts development
artifacts maintained and stored tightly coupled with the software itself.
If the higher-level tooling is used to create to finally create deployment
instances this assures the technical relevance of such descriptions. to avoid
fantasy creations on this level, which do not mirror the reality anymore.

3) Describing intended Orchestrations (Landscape Design Time Model)

   The design time model defines which services should be available in 
a landscape and how they are wired. The services hereby are described by the 
underlying service model. The design elements and their wiring can be
validated by evaluating the requirements and constraints described here.
Tools can be provided working on a software repository (model level 1) to extract
available services and their features to guide the designer of a landscape
to achieve a consistent and complete landscape with all the required versions and
dependencies.

   The same way upgrades can be planned or even automatically determined.
  
   To be valuable this model must be accompanied by some tooling using the 
deployment and installation information from lavel 2 together with the 
described artifacts exploiting identities and access information from level 1.

4) Describing Runtime Elements (Landscape Runtime Model)

  A runtime model describes runtimes and elements deployed into runtimes.
Hereby, runtimes are typically nested; for example, a container is running in Pod running on a node, which is part of a Kubernetes cluster. The identified 
elements are assigned to elements of the design model from level 2. This can be achieved with the help of the deployment system, which annotates deployed elements
according to the information received from level 2, which is its client. 
Or correlating technical deployments annotated during the production process by informations from model 1 (for example by adding annotations to an image manifest)
with deployments initiated by model 3.

### The Software Model

Describing software artifacts with their features is the base functionality for the complete model hierarchy. It has to provide everything required to link together all the following models and
to provide information and content required by those models and related tools.

Such a model must therefore meet some requirements:
- It must be able to describe aggregated software units at any granularity
- It must be applicable to any kind of artifact to be content technology-agnostic.
- It must provide a globally unique naming scheme to offer a common language understandable to all related tools and models to be able to talk about the same things (lingua france).
- It must relate to the reality, the effective artifacts provided by build processes, delivered to customers and finally used to install software or complete landscapes.
- It must provide information needed to ensure the authenticity of described content.
- It must combine a location-agnostic description with the possibility to keep track of artifact locations along with transport chains to always describe valid information also for private and fenced environments 
- It should be able to express related artifact structures across multiple versions.

To be technically relevant, describing software is not enough. It must also offer all information required by tools working on it 
to technically access the described content. And a standard tooling is required to work with content and descriptions.

#### The Model

Typically, software is organized by components. A component is an entity with a particular meaning, for example, single programs as tools, or complete systems, like Kubernetes. Such a core model is the
[*Open Component Model*](https://ocm.software). The base element ere is the component.

A *Component* as such does not describe any software artifacts, just a common meaning for the elements it aggregates, the versions of software. 

An important feature for such a model is to provide a globally unique and location-agnostic naming scheme. So, descriptions following this model must be valid in any context, regardless, whether software has been copied into local environments, or not.

Components are uniquely identified by a globally unique identifier, consisting of a FQDN followed by a hierarchically organized name (the path). Hereby, a providing organization must own the FQDN in combination with a path prefix to avoid conflicts.
This prefix then opens a namespace which can be maintained locally by the owning organization. For example, the owner of a FQDN can 
manage its namespace directly under the DNS name. Or it delegates
ownership by assigning one or more levels to other (sub) organizations. For example, `github.com` delegates one project name level to another organization or person who owns the github project. This way, the prefix `github.com/<your-org>`can be used to span a local namespace for your components without conflicting with names used by other projects. 

Below a component now an element is required carrying real content.
Those elements are called *Component Versions*.
A component version describes particular versions of a software whose meaning is defined by the component they are related to.
A component may have any number of versions (even none, if it is
just in planning). A component version is a set of software *Artifacts*, which together build a version of a component.
Such a set might be hierarchically described.

<p align="center">
<img src="./media/ocm1.png" style="width:50%" />
</p>

It may be described by listing directly the artifacts and/or by referring to versions of other components. In such a case the component version is composed by aggregating other component versions maintained independently. Hereby, it is possible to refer to multiple versions of a used component. For example, The Gardener
(a managed Kubernetes service) includes multiple Kubernetes versions, always all the versions directly supported by a Gardener version. 

For this, all elements of a component version have a local name, which identifies a particular entry in the context of the component version. THose names reflect the meaning of a nested
element in the context of a component. Therefore, successive versions should stick to this meaning and use the same names for elements with the same meaning as before. This enables tools
acting on the model to find artifacts across a range of versions
by referring to the persistent meaning of an artifact in the context of a component.

This way any artifact can be addressed either globally or locally. Globally any artifact is addressed by an identity
given by the component id, the component version and the name of the artifact as defined in this version. Relative to a component version, there is also a possibility to refer to any artifact. This is possible by either using directly its local name or by specifying a path of component version reference names to navigate down the nesting hierarchy followed by the artifact name as used in the containing component version (*relative artifact names*).

<p align="center">
<img src="./media/ocm1a.png" style="width:50%" />
</p>

Besides the pure model element definitions, a description format for component versions is defined. It can be persisted in some content repository (see below). 

This description format is the basis for signing the content to ensure and validate the authenticity of provided content.
To ensure the validity of the described information even along a transport path of software as required to support private or even fenced environments, this must be reflected in the way artifacts are described.
One requirement was to provide information required to access the
technical artifacts behind the description. To technology-agnostic, such artifacts may live in any kind of technical repository. For example, OCI registries are simple blob store. This is formalized by introducing access types defining a set of attributes (the access specification) required to technically locate and access (not required credentials) the artifact in any repository. This information needs to be shared, but can be modified by tools that transfer content from one repository landscape to another. Signing a component version must omit such environment-specific information but include information identifying the technical content. This is typically a hash, either a binary hash, or a logical hash for content, which might require modification during a transport step.

To support this, a normalization of a component version description is defined and  a formal API to gain access to the technical artifact blob according to the actual access specification.

Accessing a component version content just by using a component version identity requires some kind of repository able to store 
component version descriptions and to lookup those descriptions
just be using their identity.

The model does not define an own technical repository specification for this, like OCI did for images. Instead,
this is covered by including specifications for a mapping of the OCM model to existing technical repository types. Like the access specification this is typed and can be extended by new types.

To support a transport of software, it must be possible to store all content, the descriptions plus described artifacts in some 
archive, which may act as OCM repository. But because it must also be able to host the artifacts without referring to external repositories, the repository specification as well as
the access specification must include the possibility to store artifacts along with the component version description (called *local access*).

All this requires some core tooling as described by the next section. But the model is basically an enabler for many more usecases and specialized tooling. Because it combines an environment and technology-independent descriptions of software, the technical access to described content and a globally unique
identification, independent tools can operate on the described content and maintain own information bases. The information is always bound together by using the common identity sheme.
This enables tools to easily correlate information found in multiple such information bases. For example, the *OCM Gear*
provides a scanning, reporting and triaging framework for vulerabilities in the context of component version aggregation.
It is based on inbound identities, uses the artifact access feature to feed the scanners and stored the found information under the identities found in the component version descriptions.

Another example can be installers directly working on a component version to extract the required installation procedures delivered as part of the software and retrieving the required deploy artifacts or their locations in the actual repository landscape.

### The Tooling

The technology independence of the model requires a number of extension points, which need implementations to be provided for, to finally work with the model in a concreate environment. For example, different types for access specifications, signing algorithms and tools or OCM repository types.

To achieve all the goals defined at the beginning the model and description format must be accompanied by a basic core-tooling.
based on an extensible library able to handle all those cases. It allows transparent access to component versions with all the described content stored in any supported OCM repository and external content repository, including a file-based (transport archive) one.

On-top an *ocm-cli* is provided able to do all the standard tasks 
required to work with the model:
- composing new component versions and storing the content in
  a given repository landscape
- accessing any content adhering to the component model just be using the appropriate identities.
- signing and validation component versions
- transporting from one repository landscape to another (or into a archive file and vice versa)  adapting the access information by uploading the artifacts into an intended repository landscape.

<p align="center">
<img src="./media/ocm2.png" style="width:50%" />
</p>

### The Service Model

The component model described software artifacts grouped
by components representing a dedicated meaning or purpose.
But this is not enough to gain information what runtime entities
are required to establish an orchestrated service mesh and service landscape. A more high-level model is required dealing with elements relevant for landscape operators, a service model. 

#### The Service Term

The term service is highly overloaded. Here, it is used in a very general way.

A *service* in its most general form is just an entity, which offers some API to its consumers.

<p align="center">
<img src="./media/service.png" style="width:10%" />
</p>

This spans a wide range of implementations. It might be a software module like a Go package offering some API used as part of an application program, it can be some services offering a REST API to be used over a public or private network like an OCI registry, or it is a runtime environment like Kubernetes or Cloud Foundry used to run other services.

A service may be an *Instance* of some service kind.

A *Service Kind* describes the common features shared by all its instances.

Basically, the features of a service can always be described in an abstract manner by a service kind. For example, the API specification must be identical, but every instance may have another formal instance endpoint to reach the service. The term service, service instance and service kind are often used interchangeable. If a dedicated meaning is intended, which is not evident from the usage context
the explicit terms are used.

A service typically lives on a service runtime, for example a VM lives on a hypervisor, a Kubernetes Pod on a Kubernetes cluster, or even a database scheme living on a database.
A database scheme is a service in the sense of the definition of a service used here, because it is an entity with dedicated users, permissions and maintained elements (like tables) separated from elements in other schemes. It has an own API and API endpoint. A scheme has formally an own endpoint although it is typically reachable over a common API of the database. This is comparable to objects and classes in an object-oriented environment. Technically, the methods and their implementations are bound to the classes. But formally they belong to the objects, the access point is not the class (like for abstract data types), but the object identity.

<p align="center">
<img src="./media/service-runtime.png" style="width:10%" />
</p>

A *service runtime* is a service used to embed other services and their API.

A service may be an *Instance* of some service kind. A *Service Kind*  describes the common features shared by all its instances.

Basically, the features of a service can always be described in an abstract manner by a service kind. For example, the API specification must be identical, but every instance may have another formal instance endpoint to reach the service.

If we take a closer look at such a service, it can be decomposed into multiple other (micro) services. I
n general, it may consist of a set of other services, which are integral part of it. Those services are typically part of its installation or its installation target and are required to run the service. These dependencies are called implementation dependencies.

<p align="center">
<img src="./media/service-dependencies.png" style="width:40%" />
</p>

*Implementation dependencies* are dependencies to other services which describe an internal decomposition of the service into smaller services.

Besides those implementation dependencies a service may be orchestrated in an external mesh of interconnected services to finally provide its functionality towards its users. Those services are required to fulfill API requests but are not part of its decomposition. Typically, this set of services is building some kind of business context. The business context binds together a set of services, which share a common interpretation context for elements like business partners, customers, or cost centers.
The particular dependencies are called orchestration dependencies.

<p align="center">
<img src="./media/service-dependencies2.png" style="width:70%" />
</p>[

*Orchestration dependencies* are dependencies of a service to separately maintained services used to embed the service into some kind of business context.

A typical standard on-premise landscape is just a net of interconnected top-level services building the business or landscape context according to the orchestration dependencies of the involved services.
Every service may be decomposed into a set of subsequent other services according to the implementation dependencies of these services.

A typical service description model for such a realm just describes the orchestration dependencies of all involved services. Additionally, implementation dependencies are used for all those services as long as their lifecycles are not covered by the installation procedure of the particular service.

For a cloud-based environment this is not sufficient because a completely new kind of service must be handled. While in on-premise scenarios dedicated service instances are explicitly installed per landscape context, cloud means an arbitrary number of business contexts are created on-demand by customers of the cloud landscape. This requires a completely new kind of service: Services managing the lifecycle of other (managed) services capable to manage the lifecycle of such service instances and orchestrate them on demand.

A *service provider service* (or short just *service provider*) is a service offering a lifecycle API for service instances of one or more other kinds.

A *managed service instance* (or short *managed service*) is a service (instance) whose lifecycle is managed by a service provider.

<p align="center">
<img src="./media/service-provider.png" style="width:70%" />
</p>

Compared to a typical on-premise landscape the service providers are able to create the managed service finally used to orchestrate a dedicated usage context.

This requires three different features:
-	A service provider is used to manage services for a dedicated consumer. Therefore, it requires consumer-specific tenants or accounts.
-	The service provider API can again be decomposed into two different APIs:
     -	Tenant/account creation
     -	Lifecycle API as part of a particular tenant
     If sub accounts are supported (preferred), the account is a further resource manageable by resources in an account.
-	To support the orchestration of managed services in a business context the configuration of service requests must include the dependencies used to embed a service instance into its interconnected business context. The consumer placing an order for an instance must include the intended orchestration dependencies (used to connect the new instance with other services in the business context) in the specification of the order.

Dependencies to a service provider are used to maintain digital twins for managed service instances. For example, a service provider could use another service provider to
maintain runtime services for the own managed service instances (This is not shown by the image for simplicity, because basically runtimes are again services)

 The same is valid for regular installers. They could use a service provider to create services required to fulfill implementation dependencies of the service to be installed. THis leads to the next type of service: installers.

Every service must come into life before it can be orchestrated and used by its consumer. In on-premise scenarios this is typically handled by an installer, which is executed by the landscape administrator.

An *Installer* is a service capable to create and maintain the lifecycle of some other kind of service.

It is configured with the orchestration dependencies required by the installed service instance. Additionally, some implementation dependencies of the service instance to be installed must be given, as long as they are not yet covered by the installer itself (for example, the chosen runtime service instance). They are covered by the installer, if it maintains those instances as part of the installation process or if they are implicitly given by the runtime of the installer.

In a cloud-based scenario, this must be completely automated. This task is handled by the service provider behind its lifecycle management API. This way any number and kind of customer business landscape can be created on-demand.

Nevertheless, instead of explicitly installing the service instances participating in an on-premise business context by an installer, now the service providers used to manage service instances for an arbitrary number of business contexts must be installed. The services building the cloud landscape (the orchestration of service providers) must be installed to create a new cloud instance. There will typically never be a single instance of some kind of public cloud. Instead, there will be several sovereign cloud instances to fulfill legal and other requirements.

So, we still need explicit installers even in cloud-based scenarios and an automation to setup and maintain such landscapes. All this must be describable by the intended service description model.

As mentioned above, even an installer might have service dependencies. In on-premise scenarios they are used to determine the service runtime and to orchestrate the intended business context, in cloud scenarios they are used to connect the various service managements.


#### The Model

All those elements described by the previous section must be describable by the service model. Here, we still use the term service. But the purpose is not to describe a concrete service landscape with service instances, but requirements for instantiating
a landscape and patterns for its layout. Therefore, the term service does not denote particular instances, it just describes the features of an arbitrary instance, a service kind.
For the model elements service means the kind of service, not a particular instance. The instance terms are lifted to the kind level. This way dependencies will describe requirements instead of concrete usage relations among service instances.

<p align="center">
<img src="./media/service-model.png" style="width:70%" />
</p>

Naturally, the central element of the service model is the Service.

A *Service* describes the features of arbibrary instances of a particular version of this service, a service kind.

One such feature is the API (or potentially multiple APIs) of a service instance.
Other ones are the required implementation and orchestration dependencies.

The model distinguishes between three different service sub types:
- ordinary services 
- service providers 
- installation services

And a fourth type, which is not really a service, but a service variable reduced to its API specification: a service contract.

A *Service Contract* is some kind of placeholder for a concrete service, which defines some constraints a service must meet to be substitutable for the contract.

This is basically the semantic of an intended service mainly expressed by its API.

A *Service Provider* is a service offering a lifecycle API for managed services.
In addition to an ordinary service, it declares the managed services and how their 
dependencies are resolved. It can decide whether such a dependency is resolved by the provider (for example, by using one of its dependencies to an appropriate service provider) or exposed as orchestration dependency for the required instance configuration.

An installer is a service usable to install another service.

More details about the model elements can be found in the [model specification on github](https://github.com/open-component-model/service-model/blob/main/docs/ServiceModel.docx)

#### Relation to the Component Model

Now let's consider how this service model is liked to the component model.
The elements of this model are basically development objects like the software artifacts 
used to run services. As such a description format is defined, which will be stored along with other software artifacts as artifacts in component versions.
The elements of the model use identities derived from the component model identities
and the versioning is directly connected.
Together with the explicitly typed model artifacts part of a component version.
this allows lookup the definition of model elements in a component model repository  directly from their identity.

A second relation is given by artifact references. Model elements might be linked
to appropriate artifacts by *relative artifact references*, evaluated relative to the component version hosting the model definition holding the reference.
This is at least required for an installation service. It defines the installer 
artifact(s) required to execute the installation process.
Another possibility is to link implementation artifacts to service definitions.
But this is problematic, because it would not have technical relevance. It just be 
informational content which tends to get outdated as soon it has been established.
A better solution is to let the installer provide this information as will be seen
in the section about the runtime model. The installer required information about the
artifacts required for the deployments anyway. Here, we have technically relevant information, which always reflects the reality. It must dynamically be provided by the installer. The price is that this kind of information is not statically available as part of the model, but only dynamically in a runtime repository. But anyway, it still
completely defined by the content of the component model, because it holds the service model artifacts, which define the installer, which again refers to descriptions and/or code stored by component model elements.

#### The Tooling

Like for the component model, some tooling is useful. There should be standard tools
for extracting model elements from a component repository. One solution here is an
appropriate plugin for the ocm-cli.  

For a given context the information from a component repository can be transferred into 
a dedicated service model repository, which makes the elements and navigating their 
relations directly accessible and queryable, without the need of scanning a
component repository not optimized for relational access.

### The Design Time Model

In contrast to the service model, which deals with describing the features of service kinds, the design time model is intended to describe a concrete orchestrated service landscape (or service mesh). Therefore, it deals with service instances.

#### The Model

Similar to the service model, the central and basically sole element of the design time model is the service instance. It features several kinds of relations:
- orchestration dependencies
- installation dependency to a service provider server or an installation service.
- dependency to a runtime (could be a regular service) if not handled by the installation.


<p align="center">
<img src="./media/designtime-model.png" style="width:70%" />
</p>

The identities of the service instances are not bound to some upper model and can freely be chosen
by the used tooling. The identities must not be globally unique, only in the context
of the described landscape.

If it is intended to aggregate multiple landscape definitions into a larger one,
landscapes should be explicit model elements, which then require some kind of unique
identities in the context they are managed. Service dependencies are then paths following the landscape nesting down to the concrete service instance to use.

The landscape definition must be stored in some design time repository to be accessible
by other tools. It is recommended to follow the component repository pattern supporting
multiple technical stores, like a filesystem archive, a blobstore and additionally some relational database under a common API usable by tools.

The model may contain configuration settings for the modeled service instances
if a complete automation is intended, or the model is used to generate
a git-ops environment, where the missing information must be filled out by an
administrator.

A simple concreate model could then look like this:

<p align="center">
<img src="./media/designtime-model2.png" style="width:70%" />
</p>

The landscape should include a dedicated service instance. This is the intention
by the landscape designer. According to the service model, this implies some requirements, which have to be added by the landscape designer to finally have a valid and complete model. He decides to provide the desired service instance by a service provider using another service provider, for example to provide required runtime services (greyed). The final design is then stored in a design time repository
to be usable by further tooling.


#### Relation to the Component and Service Model

And here, there is the direct link to the service model. The service instance is directly related to a service of the service model. The definition here determines
which dependencies must be resolved to result in a valid design time model.

This information can be used to validate a given model or to assist in completing the model to fulfill all the requirements of the involved service instances.

To support configuration settings, the service model should be enriched by
a formal declaration of the required configuration (not yet part of the model).
This could be explicit elements, or it is mapped to annotation values.

To provide the possibility for a complete automation, there must also be tooling
able to apply such settings in a concrete setting. This could, for example, be
plugins for a landscape installation system (see below), which are again expressed as
artifacts in the component model referenced by the service model elements.

#### The Tooling

The previous section just opens the door for some useful tool support.

##### Landscape Modeller
  Besides the repository, a *landscape modeler* could be provided. It provides
and API (or UI) for the landscape designer assisting in providing a complete, valid and consistent landscape setup.
It uses the service model (available via the service model repository) to
offer choices for the intended service instances. In a second step
it can evaluate the dependencies and offer possibilities how to complete the
setup. Depending on the content of the service model repository, this could be 
- directly implied services and installers, or 
- it offers possibilities to resolve dependencies by service instances already existing in the landscape or to create new ones.
- to offer alternatives how to resolve contract dependencies.

<p align="center">
<img src="./media/landscape-modeler.png" style="width:70%" />
</p>

In the example shown above, the service repository contains an installer and a service provider for the initially intended service instance. The designer now has the choice,
he decides for the provider solution, because there will be multiple such instances in the future. Now, this provider requires some other service, leaving the option for different implementations by describing a contract dependency. But the available service model
contains only one possible matching service, which can automatically be added.
Or the designed decides to make some other service option available, for example,
by browsing a marketplace to look for and buy some other solution. This one the  also contains a service model description, which will be added to the service model repository.
After this it is available for landscape modeler, which now can
offer multiple possibilities for resolving the contract dependency.

##### The Landscape Installation System

For the design time model to technically relevant, it must somehow automatically
be mapped to some operation environment. There might be several levels of support
- mapping the landscape model into a Git-Ops project setup, which must be completed by the landscape admin.
- directly mapping the landscape model into deployments,

The tool for the second possibility could be a *Landscape Installation System*.
The task is quite complex and heterogeneous, but it could significantly be simplified
by generally relying on some declarative ordering environment, as proposed by the *Apeiro Reference Architecture*.

A second simplication could be to rely on Kubernetes-like execution environments and standardize installers to be implemented by *Kubernetes Operators*.

The task of a generic installation system is then just to deploy operator and 
resources to instruct the operator to deploy an instance. The latter part is basically
identical to ordering an instance from a service provider (see Apeiro).

So, required is the link from the service model referring to an appropriate
artifact in the component model (which is already formalized), and the plugins for the installation system used to map configurations into declarative request resources.

###### The Service Account Management

To being able to work service providers must be able to interact with each other,
this typically requires authentication and permissions. A landscape installation system
must therefor fall back to anpthe tool, a *service account management*. THis must be standardized, to enable the installation system to request service accounts for service provides automatically passed to using services as part of the service provider orchestration.


##### Maintenace Optimizer 

<p align="center">
<img src="./media/landscape-installation-system.png" style="width:70%" />
</p>

But the landscape installation system could do even more than just initially setting up a landscape.
It should keep the design time model in an own version describing the current state of the landscape. Whenever no the intended landscape is changed by the landscape designer,
it can determine a delta and apply appropriate changes to the real technical landscape.
Hereby, version contraints described by the service model can be used to propose
and schedule upgrade paths.

The result is then a deployment landscape consisting of the service instances described by the design time model plus implicitly creates service instances for implementation dependencies manged by the service providers, which could be described by the service model. It is important to note here that this description must be complete in terms
of the modeled service dependencies. But especially implementation dependencies must not necessarily be modeled in all details.


### The Runtime Model

The last section just shows some gap between the definitions in a service model and the final technical landscape: not all implementations dependencies managed by a service provider must be (and potentially will not be) completely covered by the service model to avoid unnecessary complexity.

Another reason for diverging landscapes are operators at the end of the definition level, which might create, switch or extend runtime entities on the fly. For example,
 Kubernetes `Deployment`s results in a sequence of `ReplicaSet`s, which dynamically
create `Pod`s for scaling a failure handling.

Those elements will never (and cannot) be modeled at the level of the service or design time model. Nevertheless, it would be extremely helpful to be able to judge on
an actual landscape state, whether it still meets the intended landscape, or whether
there are elements not expected. This is proving extremely difficult.

But even there, the described model hierarchy can help. First, a formal description
of the effective technical landscape must be provided. THis direly leads to an additional model level: the runtime model.


#### The Model

The runtime model must be able to model all relevant runtime elements at an appropriate detail level. The general observation here is that every element in question is 
determined by another element is embedded into, stopping at some base elements, for which there is a conscious decision not to dive deeper. For example, we stop modeling at a VM level, deciding not to model hypervisors.

This again results in a very simple model design.

<p align="center">
<img src="./media/runtime-model.png" style="width:70%" />
</p>

It is basically oriented on the initial definition of a service (instance).

There is one central element, the *Runtime Element* which runs on another element and may use other elements. All tose elements are formally typed, for example, it can be a VM, a Pod, or a Database Scheme.

Additionally, there is an *Abstract Element*, which reflects an expected type of runtime elements. Any dynamically maintained element can be related, directly or indirectly to such an element to describe the reason for its existence.
For example, in Kubernetes it could represent a set of Pods and active execution unit of a service, created via the generation chain Deployment/ReplicatSet/Pod. The relations inside the chain of technically provided by the system and can be used to derive the relation to the expected abstract element of a service deployment.

This directly leads to a tool environment required to derive such a model from concreate landscape elements. And here we found several possibilities to relate the runtime model with the upper model layers.

#### Relation to upper Model

Any runtime element can directly be related to a service instance described by the design time model used to instantiate the landscape. This is the most obvious relation.
Such a relation could, for example, be established by annotations on the technical elements (if supported). Even a relation to artifacts of the component model is partly possible. For example, an image intended to be published via the component model could be enriched already ba the build process by appropriate metadata. 

So, every runtime element must be relatable to either a component, service or design time model element. This information can later be used by an outer tools to relate the runtime elements to appropriate design time model elements.

<p align="center">
<img src="./media/runtime-model-relations.png" style="width:70%" />
</p>

But those relations are not generally possible, the only relations directly derivable from the landscape are the runs-on-relations. So, to setup a complete runtime model and correlate it with the desired design time model requires extnsive tool support, heuristics and specialized analysis funtionality.


#### The Tooling

First of all, the elements of the runtime model must be held in a runtime repository.
It is fed by different sources:
- specialized runtime observers report the existence of technical elements and their runs-on relation
- the observer extracts relations part of the metadata.
- directly generated or expected abstract elements can directly be created by the installation environment.
- the installation environments try to enrich directly deployed elements with appropriate metadata usable to establish a link to the  design time model elements they are created for.

On-top of this, analysis tools can be used trying to use the available information to establish relations to abstract runtime elements or even elements of higher models.
For example, if a pod is created from an image known to be provided by a dedicated component of he component model, which is deployed as part of a service described by the service model, it must belong to a particular service kind. If then the deployment of a service instance according to the design time model is using a separate namespace in Kubernetes, the Pod can be assigned to this service. The more assignment paths are found the more plausible is such a derived relation.

<p align="center">
<img src="./media/runtime-correlation.png" style="width:70%" />
</p>

This analysis environment must be highly configurable and extensible. This could, for example, be achieved by using plugin artifacts described as part of the component model and assigned by the service model.

The result is a runtime model usable to find unrelated runtime elements and following up the model hierarchy to relate runtime elements with delivery artifacts and components.
Using the enhanced analysis and reporting capabilities of the analysis gear on top of the component model, it is then possible to figure out involved software, including libraries, versions, vulnerabilities and triage information.

## Enhancement Remarks

The component model seems quite complete. For the upper models there will for sure be enhancements required. During the development of the various tool chains, such demands will arise to fulfill the need to couple and extend those toolsets.
Fore example is quite evident that the service model must include formalized configuration information to be used by the landscape modeler. Another requirement arises from the runtime analysis, which will for sure require specialized correlation plugins deliverable by the component model. They must be assigned by the service model.

## Bootstrapping

A basis for an automated landscape setup. the required services, like the landscape modeler, the service account management and the landscape installation system must be available. This could be installed by a simplified installer already from the component model information. Their services will be initial parts of a landscape.