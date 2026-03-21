## Responsibility tracking

In simplified environments using declarative order models like the Kubernetes Resource Model (KRM), every element in the runtime is basically requested by some digital twin in 
the used dataplane. The link between the request object and the generated resources
can be provided by the responsible controllers, as status in the request or annotations
in the runtime environment. Even in nested generation scenarios this information
can be tracked back to the originating resource, because all involved resources can be seen in the same dataplane.

Every request object lives in an organizational context, a workspace in KCP or 
a namespace in a regular Kubernetes dataplane. Such a context represents the organizational
unit finally responsible for the creation of all subsequent resources and therefore for the effective elements in the various involved runtimes. Following back the generation relation, the originating object can be found.

THis information can then be used to enrich the [runtime model](rtmodel.md) to be able directly relate a runtime element with the order responsible for generating this object.

Using the audit information provided by such a dataplane it is even possible to track down
to dedicated users working on the request level.

MOre difficult is to track back changes in the runtime model to the orginal cause:
- it could be a reaction of a controller because of some detected drift or
- it could be related to some change in the original order.
- it could be caused by some automated operation, like rolling a certificate or secret.

Both kinds of reason can be combined along the generation chain at any description level.

There is a Kubernetes project [Kausality](https://github.com/kausality-io/kausality), trying to track reasons for changes on resources in a Kuberneets dataplane.