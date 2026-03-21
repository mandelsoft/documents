## Responsibility tracking

In simplified environments using declarative order models like the Kubernetes Resource Model (KRM), every in the runtime is basically requested by some digital twin in 
the used dataplane. The link between the request object and the generated resources
can be provided by the responsible controller, as status in the request or annotations
in the runtime environment. Even in hierarchical generation scenarios this information
can be tracked back to the originating resource.

Every request object lives in an organizational context, a workspace in KCP or 
a namespace in a regular Kubernetes dataplane. Such a context represents the organizational
unit finally responsible for the creation of all subsequent resources and therefore for the effective elements in the various involved runtimes.

Using the audit information provided by such a dataplane it is even possible to track down
to dedicated users working on the request level.