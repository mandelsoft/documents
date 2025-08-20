# Authorizing Object Relations in Kubernetes

Kubernetes comes with powerful mechanisms for authorizing actors like users, tools, applications or controllers to execute operations on resources on its data plane as well as for validating changes of resources done by those actors.
But despite all these mechanisms, it is not possible to enable or prevent the establishment of relationships between objects in a meaningful and controllable way. In the next sections we will see, what this concretely means, why it is required, what is possible today and how the problem could be solved.

## Object Relations

An important concept in Kubernetes is the possibility to establish relations between two objects. This is typically done by preparing the manifest structure of a resource type to include referential information to another object. With its resource type (kind and api group), a namespace and an object name, every object in a Kubernetes dataplane has a unique identity over space (not time). The same name could be reused later in time to create a new object, after the old one has been deleted.  Additionally, it features UID, which is also unique over time.
Typically, the named identity is used to describe such referential fields. A typical example is an object representing a digital twin for a real world object requiring some credential to be used.

> In Kubernetes itself, the most prominent example is using a `Secret`, for example by a `Pod`, which uses container images loaded from an image repository. This might require credentials. Therefore, the Pod has a specification field `imagePullSecrets`.

Such a referential field might have several manifestations:
- it must at least describe the name of the used object. This is possible if the resource type is implied by the meaning of the field (for example the image pull secret reference is always a reference a `Secret` resource.)
- if objects of different types should be addressable, the type information must be included
- if the object is namespaced and cross namespace references should be possible, a namespace field must be included.
- If uniqueness over time is required an object UID can be used.

Typically, the UID is not used, because this would require to adapt all using objects, if accidentally the referenced resource has been deleted and recreates.

> Kubernetes provides with `k8s.io/api/core/v1.ObjectReference` even a standard type for the layout of a resource specification.

So, what we can see, an explicit object relation in the Kubernetes Resource Model (KRM) is typically established by a field in the using resource. It is a tupe *(s,r,t)*, wher *s* is the source object, holding the reference, *r* is the *relation type*, the meaning or *purpose* the referenced object should be used for by *s*, and *t* is the *target* object of the relation, the referenced object.

More complex n-ary relations are not used, although an object featuring multiple binary relations could be seen as an n-ary relation *(s,r,t<sub>1</sub>,...t<sub>n</sub>)*, where *r* is the resource type. So, it is sufficient to deal with binary relations.

### Why restricting object relations.

A controller responsible for a resource type uses the given referenced object to implement the object in the real work (or its target environment) using the content of the referenced resource or its real world instance, It does not know, and it cannot know, whether this should be allowed or not. Therefore, there must be a way to prevent establish a relation, which should not be possible.

For example, a secret describes the access to a technical environment, which should not be usable by all objects of a resource type describing a relation to such an environment.

### Isn't it a regular authorization problem
The first impression is, that this could be the task of the
authorization environment used to authorize modification (or creation) operations on objects. Only selected actors (principals in authorization systems or subjects in Kubernetes) should be able to establish particular relations by setting the reference field accordingly. This assumption may be true within a trust domain, but not when objects from multiple trust domains are involved. An object should usable from a particular sets of other namespaces, but not accessible for users in those namespaces, regardless whether they have admin privileges or not.

In Kubernetes a namespace is a trust-domain for authorizations on namespaced resources. Let's assume we are mainly talking about such resources. One idea of such a trust domain in Kubernetes is, that authorizations can be managed by domain-local administrators. This is fine, as long as both involved object live in the same namespace. The admin could explicitly grant permissions for establishing particular relations for selected users, or no user,if the relation should generally not be possible. The only requirement is to have such a fine-grained access control. With *Validating Admission Webhooks* this would basically be possible even today, but not bóu-of-the-box.
[Later](#operation-authorization-and-resource-validation-in-kuberbetes) we will see what is possible and what not. But our problem is not limited to objects in a single namespace.

If we try to solve the general problem with authorizations in a single trust domain, the domain admin would be able to enable the usage of any object of any other namespace. But this is definitely not intended. We need a separation of responsibility.
The trust domain of the potentially usable object must be the authorization domain for granting the possibility to use objects in this domain by other domains (here namespaces). And the admin of the using domain should be responsibility do enable particular actors to establish such a relation, but only if it hb been granted by the providing domain.

So, what can be [observed](#separation-of-operation-authorization-and-value-validation) is, that the release of an object to be referencable is not an operational authorization problem, it is more a value validation problem. And in both involved domains the operation authorization handles who can release on object to be usable, or who can finally configure such a relation for a particular object.

One may be tempted to solve the problem by using permissions to grant appropriate permissions required to establish such relations. But this will lead into a dead end, because:
- the resulting authorizations will become incredibly complicated.
- they are cross-domain, which means that they cannot be managed inside a domain, but by a system administrator.

And finally, it would not express the actual meaning.

Because of these problems often cross-namespace references are not supported by the resource type (or by the controller). If urgently required for a particular kind of resource a typical solution is to introduce proxy resources working as indirection.
For example a `SecretRef`resource whose sole task is to reference 
a secret in another namespace. With RBAC the creation or modification is prohibited by namespace users and admins. If a secret should be reusable the cluster admin has to create such an object in the intended target namespace. But the price is not only to require the cluster admin to enable the usage by creating such a proxy object, but the controller must be prepared for this, also. They have to support references to the
direct object type (for example secrets), and the proxy resource type, also. This might be a solution for some special cases, but is not a practical solution for the general use case.


## Operation Authorization and Resource Validation in Kubernetes

Let's first have a look at the existing mechanism in Kubernetes
and see what can be done with it.

### RBAC

First of all, there is RBAC, which is an acronym for Role-Based-Access-Control. It is solely meant for authorizing operations on objects stored in the Kubernetes data plane or for object-less operations in a cluster. Principals, called *subjects* in Kubernetes, may be users, service accounts as representatives for machine users applications or controllers, or any actor whose identity can be expressed as string. It is possible grant permissions for data plane operations like get, list, watch and put on resources, resource types and or namespaces. Operation targets are described by role objects, which can be bound to subjects by binding objects. To enable namespace local authorization management, there are cluster wide versions of those resources and namespace-local ones. A cluster admin can grant permissions for managing namespace-local roles and bindings to specific actors by cluster-wide versions.

This mechanism does not support permissions depending on features of an object, like annotations, labels or fields. There are projects dealing with extensions enabling such possibilities.

Those extensions would enable to restrict the usage of particular references for modifications of an object.

### Validating Admission Webhooks.

Once a modification request is authorized validating webhooks are used check whether a requested modification of resource (including creation and deletion) is valid. Therefore, the hook provides the old manifest version together with the requested new one, and the hook can accept or reject the requested change.
It can therefore be used to check for valid field values or the consistency of correlated fields as well as allowed changes (for example for read-only attributes).

The focus here lies on the validation of the given resource manifest. This is important for non-standard resource types (whose validation is done by the API server), which can be declared by CRDs (Custom Resource Definitions). Providing such a webhook can be added to provide type specific checks. As with Kubernetes v1.25, many such checks can be already described by CEL expressions as part of the OpenAPI Schema specification.

> Kubernetes added `x-kubernetes-validations` as an extension to OpenAPI v3.
> ```yaml
>    kind: CustomResourceDefinition
>    metadata:
>      name: widgets.example.com
>    spec:
>      group: example.com
>      names:
>        kind: Widget
>        plural: widgets
>      scope: Namespaced
>      versions:
>      - name: v1
>        served: true
>        storage: true
>        schema:
>          openAPIV3Schema:
>            type: object
>            properties:
>              spec:
>                type: object
>                properties:
>                  size:
>                    type: integer
>                  maxSize:
>                    type: integer
>            x-kubernetes-validations:
>            - rule: "self.size <= self.maxSize"
>              message: "spec.size must not exceed spec.maxSize"
>  ```

But the admission API also provides information about the requesting subject. THis way it can not only be used for manifest validation, but it can also reject modifications not allowed for the requesting user. Therefore, validating admission webhooks can de-facto also be used to implement operation authorizations. Because they are only called for modifications, read-authorizations cannot be handled.

This kind of hybrid-functionality is therefore not sufficient to really implement an authorization system based on attribute based conditions, it is basically only valid for value validation and it [mixes validation with autorizations](#separation-of-operation-authorization-and-value-validation).

Another possibility to gain full control over such an extensive validation is to implement an API group by an extension api server (or aggregated API server) instead of a set of regular controllers.

### Validation Admission Policies

As of version v1.30 Kubernetes supports Validation Admission Policies. They were introduced to replace the need for coding Validation Admission Webhooks with a standardized declaration-based approach controlled via regular resources.

It is based on two new resources ValidatingAdmissionPolicy`and ValidatingAdmissionPolicyBinding`, both are cluster-scoped.
A policy desclares validation rules on some resource types.

> ```yaml
> apiVersion: admissionregistration.k8s.io/v1
> kind: ValidatingAdmissionPolicy
> metadata:
>   name: require-team-label
> spec:
>   matchConstraints:
>     resourceRules:
>       - apiGroups: [""]
>         apiVersions: ["v1"]
>         operations: ["CREATE","UPDATE"]
>         resources: ["pods"]
>   validations:
>     - expression: "has(object.metadata.labels['team'])"
>       message: "All Pods must have a 'team' label"
> ```

> ```yaml
> apiVersion: admissionregistration.k8s.io/v1
> kind: ValidatingAdmissionPolicyBinding
> metadata:
>   name: require-team-label-binding
> spec:
>   policyName: require-team-label   # The policy to bind
>   # Bind only to certain users or service accounts
>   subjects:
>     - kind: User
>       name: "alice@example.com"
>     - kind: Group
>       name: "dev-team"
>     - kind: ServiceAccount
>       name: "ci-pipeline"
>       namespace: "ci"
>   # Apply only in specific namespaces
>   namespaceSelector:
>     matchExpressions:
>       - key: environment
>         operator: In
>         values: ["prod", "staging"]
> ```

## Separation of Operation Authorization and Value Validation.

Instead of trying to reuse, extend or adapt existing technical solutions, we should carefully examine the problem space and clearly distinguish between different meanings.

Only because a problem can be solved by a similar solution implementation, this does not necessarily mean, that the same description methodology should be offered for a user.
It could require different or  different set of attributes to describe a request, or, even more important, different responsibilities could be involved.

On the other side this also does not necessarily mean, that behind the scenes a common implementation may not be used, called
by different frontends.

So, let's have a closer look at the problem space covered by the today's mechanisms in Kubernetes.

First of all, there are obviously two different tasks:
- *Value Validation*: validate whether fields and field values of an object are valid
- *Authorization Validation*: validate whether a requested operation if allowed for the requesting principal.

It is something completely different, whether a particular user is allowed to set a dedicated value combination for a field or whether this combination is possible at all.

### Value Validation

The sole task of value validation is to avoid the setting of field/value combinations of an object which is not valid or possible.

But having a closer look this is not a single task. Whether a field/value combination may depend on two different kinds of contraints arising from two different responsibilities:
- *Domain Validation*: The resource type has a particular meaning determined. And this meaning implies constraints for value and field combinations. For example a component of a RGB color value is always a number in the range 0<=x<=255. This has nothing to do with the context the color should be used, it is completely defined by the meaning behind the element (although it might technically be an integer). Or those constraints might be given by the underlying elements of the real world finally represented by the resource object.
- *Context Validation*: A particular resource object might be used in a concrete context. This context is typically identified by some label, annotation of field setting defined by an application or group of users in a namespace. There may be any number of such contexts per cluster. The task here is to be able to limit field/value combinations according to requirements of this usage context. Those limitations are not intrinsic to the resource type.

As we can see both kinds of limitations are subject to different areas of responsibility. While the constraints for the domain validation are arising from the resource type itself and are the same and enforced for every object for every application of this resource type in a cluster, the context constraints are arising from the usage of particular resources in a particular cluster.
So, responsible for the first flavor is the provider of a resource type, and for the seconds flavor some principals in a particular cluster. In the first case, all objects are (and must be) affected, in the second only objects that belong to a specific context.

This differentiation must be reflected in the API used to declare such validations.
This first one is usefully part of the schema definition of the resource type wherever possible. More dynamic checks, for example checking value transitions, can be done by more expressive elements. In this are Validating Admission Webkooks and Validating Admission Policies can be used. They are valid cluster-wide. But they should no check context constraints.

The second kind of constraints must be declarable by principals
responsible for particular usage contexts, typically inside a namespace. And here we easily can see, that Validating Admission  Policies cannot be used, because they are cluster-scoped resources. And this although, concerning their expressiveness,  they would be able to describe such constaints. So, the design of this new mechanism is not applicable for a common usecase according the semantics of the problem domain, it is just an adaptation of the already existing mechanism of Validating Admission Webhooks. To use it to solve the context problem, you have to leave your trust-domain and require cluster privileges.

For sure, those two kinds of constraints have nothing to do with 
the wish to restrict users in a context or trust domain to set specific values or value combinations. For example, only selected users should be able to set a high priority. All those values are valid in the context, not all users may set those values.

A basic takeaway is, that it would be a good practice to use some *Policy* resources only for context-related constraints. And handling of such resources must be authorizable locally to the related namespace. Thi expresses the local character of such constraints in constrast to contraints instrinsic to a particular resource type.

Returning to our initial problem with cross-namespace references we can see the following:
- the domain validation should accept such references, because there is no technical constraint against it. 
- it is a matter of context validation. But it cannot completely be solved inside a namespace, because one element is missing. The referenced object belongs to different trust domain we need something that authorizes a context to use it. By the way, this could even be useful inside a single namespace, if it hosts multiple usage contexts. But again, the usage of an object must still be released by the owning side, not the consuming one. So even inside a namespace with regular authorization mechanisms, a principal is required which has cross context privileges.

### Authorization

In the last sections we have seen, that value validation is formally a different task than authorization validation.
While value validation is responsible to handle the question,
*what* is valid in a particular context, authorization is responsible to answer the question, *who* may use it.

What should be describable for authorizations. RBAC is able to answer the question, who is able to execute a particular operation
on an object. Here, only the operation type (like GET,PUT,...) and the identity of the concerned object is taken into account.

To maintain authorizations in finer granularity than namespaces,
it not really possible, With validation admission webhooks only modifying requests could be handled, but no read, watch or list requests. Validating Admission Policies underly the same constraints, but at least require to maintain modification authorizations based on labels, annotations and fields and their values. But there are several more severe constraints:
- it is not possible to follow relations between objects. For example: you may establish only a reference to a secret with a particular label value.
- in contrast to RBAC roles and bindings, such policies cannot be maintained under the control of a namespace, their maintenance always requires a cluster admin.

Because of those limitations there are projects trying to
introduce a generic authorization engine like Cedar.
But POCs in this area are always limited by the constraints of admission web hooks, which are used to hook into the authorization process. And as we have seen, this mechanism
is not generally suitable for solving authorization problems. 

But even under the assumption that attribute-based conditional authorizations (Cedar does support even to follow references) would be available, would it be possible to resolve our referencing object problem?

Here, we have to see that regular authorizations alway authorize principal to do something, for example setting a reference field to a dedicated value. But as we have seen in previous section,
the problem is primarily not related to the question, *who* may establish a reference, but is such a reference allowed, it's some kind of context validation.

But even if we would be content with such an authorization solution, we still have the problem with the responsibility. Authorizations
are either maintained in a namespace for objects in this namespace or by cluster wide settings, which require some cluster-wide permissions. This is definitely not what the problem domain requires.

So, some kind of authorization is missing. There two kinds of authorizations:
- *operational authorizations* control who may execute which operation on a target object.
- *relational authorizations* control, which relation between objects is allowed in the context of a data plane.

The second kind of authorizations could be seen as authorizations where the principal role is fed by a object and the operations is the kind of relation. But this is an in implementation view. As usual, the problem arises with the responsibilities.

Le'ts assume we just extend RBAC to allow principal identities describing an object identity and the operation to describe arbitrary relation names. This does not change anything for the
responsibilities for the maintenece of Role and bindings. They are either cluster-wide or namespace-local. This means inside a namespace only roles are observed hosted by this namespace.
And this is exactly what should be avoided in the required scenario. The release of object to be usable in another namespace must be under the control of the providing namespace, not of the consuming namespace.



# Possible solutions



