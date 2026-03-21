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
This will be achieved by a hierarchy of separate closed but connected description models.

## The Model Hierarchy

Everything starts with the development of software artifacts, like libraries,
programs, or container images. There are various heterogeneous tools and environments supporting this development step. But all of them typically end up in some build
process providing artifacts intended to be delivered and/or deployed, including
installation procedures or descriptions. Here, we start our journey omitting all
those sneaky details arising from programming ecosystems.

1) [Describing Sets of Delivery Artifacts (Software Bill of Delivery) (Component Model)](swmodel.md)

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


2) [Describing Services (Service Model)](svmodel.md)

   The service model is a federated model for describing services with their
API, installation procedures, and the required deployment artifacts. To this end,
it references identities provided by the underlying model. This underlying model is not limited to software artifacts — it also governs the description of model elements of the service model.
  These descriptions are stored as part of the Software Bill of Materials and make use of its identity model for the service model elements. Registry-based access to the underlying model allows tooling to resolve model element descriptions directly from their identities, in a location-agnostic way.

   Model element descriptions are, like software artifacts, development artifacts — maintained and stored in close coupling with the software they describe. When higher-level tooling drives the creation of deployment instances from these descriptions, their technical relevance is continuously validated, preventing divergence between the described and the actual state of a deployment.


3) [Describing intended Orchestrations (Landscape Design Time Model)](dtmodel.md)

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


4) [Describing Runtime Elements (Landscape Runtime Model)](rtmodel.md)

   A runtime model describes runtimes and elements deployed into runtimes.
Hereby, runtimes are typically nested; for example, a container is running in Pod running on a node, which is part of a Kubernetes cluster. The identified 
elements are assigned to elements of the design model from level 2. This can be achieved with the help of the deployment system, which annotates deployed elements
according to the information received from level 2, which is its client. 
Or correlating technical deployments annotated during the production process by informations from model 1 (for example, by adding annotations to an image manifest)
with deployments initiated by model 3.


